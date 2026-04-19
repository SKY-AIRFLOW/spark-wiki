---
stage: 04-physical-planning
title: WindowGroupLimit — Top-N per Group 최적화
spark_versions: ["3.5"]
sources:
  - raw/explain/2026-04-19-query-B-row-number-top5.txt
  - raw/spark-source/v3.5.6/sql/core/src/main/scala/org/apache/spark/sql/execution/SparkStrategies.scala
last_verified: 2026-04-19
confidence: tentative
related:
  - 00-overview/execution-pipeline.md
  - 04-physical-planning/window-exec.md
  - 04-physical-planning/hash-aggregate-partial-final.md
  - 03-logical-optimization/predicate-pushdown.md
  - case/2026-04-19-query-B-row-number-top5.md
configs: []
apis:
  - SQL ROW_NUMBER() / RANK() / DENSE_RANK() + WHERE rank <= N
  - DataFrame (equivalent)
tags: [window, top-n, optimization, partial-final, rank-like]
---

## L1. What & Why

`WindowGroupLimit`은 "PARTITION BY 그룹 내 rank-like function의 상위 N개
row만 필요"한 패턴을 전용 연산자로 실현하는 최적화다. Spark 3.2에서 도입,
3.5에서 개선된 연산자.

대상 패턴:
```sql
SELECT * FROM (
  SELECT ..., ROW_NUMBER() OVER (PARTITION BY k ORDER BY o) AS rank
  FROM t
) WHERE rank <= N
```

순진한 실현은:
1. 전체 row를 k 기준 shuffle
2. partition 내부를 o 기준 정렬
3. row_number() 부여 (전체 row)
4. rank <= N filter (대부분 버림)

`WindowGroupLimit`의 실현은:
1. 각 input partition에서 k별 **local top-N만** 유지 (**Partial**)
2. k 기준 shuffle
3. shuffle 결과에서 k별 **global top-N만** 유지 (**Final**)
4. 그 축소된 row 집합에 대해서만 WindowExec + filter 적용

본질적으로 `HashAggregate`의 partial/final 패턴을 **top-N per group**에
적용한 것. 단, 집계값이 아니라 row 자체를 "top-N만" 유지한다는 점이
다름. 참조: [wiki/04-physical-planning/hash-aggregate-partial-final.md](hash-aggregate-partial-final.md).

Java 개발자 관점: partition 하나당 `Map<K, BoundedPriorityQueue<Row>>`를
두고 각 row를 해당 그룹의 PQ에 `offer`. PQ는 size N으로 제한되어 있어
N+1번째 offer부터는 최악 원소와 비교만. shuffle 후 final 측에서 각
그룹의 PQ들을 합쳐 global top-N 산출.

## L2. Inputs & Outputs

**입력** (Optimized LogicalPlan — 이 규칙이 삽입한 뒤):
```
Filter (rank <= N)
+- Window [rankLikeFn() ... AS rank]
   +- WindowGroupLimit [<partitionSpec>], [<orderSpec>],
                       rankLikeFn(), N
      +- <child>
```

`WindowGroupLimit`이 **Window 바로 아래**에 삽입된다는 점에 주목. 위의
`Filter`와 `Window`는 **제거되지 않는다** — 의미적 보존을 위해 남는다.
`Filter`가 추후 no-op이 되어도 plan에서는 유지.

**출력** (Physical Plan) — strategy가 **Partial + Final 2개로 분할**:

```
WindowGroupLimit [<partitionSpec>], [<orderSpec>], rankFn(), N, Final
+- WindowGroupLimit [<partitionSpec>], [<orderSpec>], rankFn(), N, Partial
   +- planLater(child)
```

**쿼리 B 관찰** (executed plan, nodes 5 & 8):

```
(8) WindowGroupLimit                                       # Final
    Input [5]: [agent_gu#308, apt_name#297, deal_date#313,
                deal_amount#304, area_sqm#305]
    Arguments: [agent_gu#308], [deal_amount#304 DESC NULLS LAST],
               row_number(), 5, Final

(7) Sort ... (partitionSpec ASC, orderSpec)
(6) Exchange hashpartitioning(agent_gu#308, 200)

(5) WindowGroupLimit                                       # Partial
    Input [5]: [agent_gu#308, apt_name#297, deal_date#313,
                deal_amount#304, area_sqm#305]
    Arguments: [agent_gu#308], [deal_amount#304 DESC NULLS LAST],
               row_number(), 5, Partial

(4) Sort [agent_gu#308 ASC NULLS FIRST,
          deal_amount#304 DESC NULLS LAST], false, 0
```

주목:
- **각 WindowGroupLimit 아래에 Sort가 있다** — partial side도 local 정렬이
  필요함 (그룹별 top-N 식별에 ordering 필요).
- partial → final 사이 `Exchange hashpartitioning(agent_gu, 200)` — aggregate
  와 같은 shuffle 구조.
- `rankLikeFunction`은 `row_number()` 그 자체를 인자로 받음. RANK / DENSE_RANK
  도 지원됨.
- limit 값 `5`는 Catalyst가 `Filter(rank <= 5)`에서 추출한 상수.

## L3. Key Mechanisms

### Strategy 변환 (verified — strategy layer만)

`sql/core/.../execution/SparkStrategies.scala:657-667`에 정의:

```scala
object WindowGroupLimit extends Strategy {
  def apply(plan: LogicalPlan): Seq[SparkPlan] = plan match {
    case logical.WindowGroupLimit(partitionSpec, orderSpec,
                                  rankLikeFunction, limit, child) =>
      val partialWindowGroupLimit = execution.window.WindowGroupLimitExec(
        partitionSpec, orderSpec, rankLikeFunction, limit,
        execution.window.Partial, planLater(child))
      val finalWindowGroupLimit = execution.window.WindowGroupLimitExec(
        partitionSpec, orderSpec, rankLikeFunction, limit,
        execution.window.Final, partialWindowGroupLimit)
      finalWindowGroupLimit :: Nil
    case _ => Nil
  }
}
```

확정된 사실:
- **Partial → Final 분할은 strategy 레벨에서 일어난다** (logical 단계가 아님).
  Optimizer가 삽입하는 것은 하나의 `logical.WindowGroupLimit`이고, strategy가
  이를 물리적으로 둘로 쪼갠다.
- 분할 시 partitionSpec, orderSpec, rankLikeFunction, limit는 그대로 전달
  되고 `Partial`/`Final` mode 파라미터만 다름.
- partitionSpec 기반 Exchange는 이 strategy가 직접 삽입하지 않음 — 다음
  단계의 `EnsureRequirements` rule이 Final 측의 distribution 요구 사항을 보고
  자동 삽입.

### Logical Optimizer 측 규칙 — `InsertWindowGroupLimit`

> 📝 Optimizer.scala ingest 대기 중
> (`sql/catalyst/src/main/scala/org/apache/spark/sql/catalyst/optimizer/InsertWindowGroupLimit.scala`
> 또는 `Optimizer.scala`의 rule batch).

Plan 관찰로부터 추론된 사실 (소스 ingest 전):
- Analyzed plan에는 없고 Optimized plan에 등장 → Catalyst rule에 의한 삽입.
- 대상 패턴: `Filter(rank <= constant) over Window(rank-like function)`.
- `<= N`, `< N+1`, `BETWEEN 1 AND N` 등 다양한 filter 형태 인식 여부는 규칙
  소스 확인 필요.
- `row_number()`, `rank()`, `dense_rank()`만 대상 — SUM/AVG 등 aggregate
  window는 대상 아님 (논리적으로 top-N 축소 불가능).

### WindowGroupLimitExec 내부 구현

> 📝 소스 코드 ingest 대기 중
> (`sql/core/.../execution/window/WindowGroupLimitExec.scala`).

Partial/Final mode의 내부 차이, BoundedPriorityQueue vs 다른 자료구조, spill
여부, codegen 지원 여부는 소스 ingest 후 기록.

## L4. Performance Levers

> 📝 소스 코드 ingest 대기 중.

Plan 관찰 기반 잠정:
- `N`이 작을수록 (e.g. top-5 vs top-100) Partial 단계의 압축률이 높다 —
  shuffle 통과량 축소.
- partition 키의 cardinality가 N에 비해 클수록 유리. 반대로 키가 하나뿐이면
  Partial이 전체 row의 top-N만 줄이는 셈으로, 그래도 이득이 남.
- 이 최적화가 적용되지 않는 경우 (ORDER BY 없음, SUM/AVG window, `<=` 아닌
  `=`, N이 컬럼 참조 등)에는 순진한 Window + Filter가 그대로 실행. 정확한
  조건은 규칙 소스에서.

## L5. How to Observe

### Plan에서 최적화 적용 확인

- `df.queryExecution.optimizedPlan` 또는 `df.explain(mode="extended")` →
  Optimized Logical Plan에 `WindowGroupLimit [<keys>], [<order>], <rankFn>(), N`
  노드가 있으면 규칙이 매칭된 것.
- 노드가 **없이** 순수 `Window + Filter(rank<=N)`만 있으면 규칙 미매칭 — 패턴
  형태 확인 (`Filter` 조건 형식, rank function 종류).

### Physical에서 Partial/Final 분할 확인

- `df.explain(mode="formatted")` → 연속된 두 `WindowGroupLimit` 노드. 위는
  `Final`, 아래는 `Partial` argument. 그 사이에 Exchange + Sort가 배치.

### 성능 체크

- Spark UI → SQL tab → 해당 쿼리의 scan metrics (input rows)와 partial
  `WindowGroupLimit`의 output rows 비교. 축소율이 최적화의 실제 이득.
- Final `WindowGroupLimit` 위의 `Window` 노드에 들어오는 row 수 = 이론적
  상한 = `(partition key 개수) × N`.

> 📝 probe 페이지 미작성. 후속 `/probe 04-physical-planning/window-group-limit`
> 대상 — "최적화 적용/미적용 두 쿼리의 input/output row 수 비교".
