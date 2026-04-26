---
stage: 03-logical-optimization
title: Infer Window Group Limit
spark_versions: ["3.5"]
sources:
  - raw/explain/2026-04-19-query-B-row-number-top5.txt
  - raw/explain/2026-04-19-query-C-window-moving-avg.txt
  - raw/spark-source/v3.5.6/sql/catalyst/src/main/scala/org/apache/spark/sql/catalyst/optimizer/InferWindowGroupLimit.scala
  - raw/spark-source/v3.5.6/sql/core/src/main/scala/org/apache/spark/sql/execution/SparkOptimizer.scala
last_verified: 2026-04-26
confidence: verified
related:
  - 00-overview/execution-pipeline.md
  - 03-logical-optimization/predicate-pushdown.md
  - 04-physical-planning/window-exec.md
  - 04-physical-planning/window-group-limit.md
  - case/2026-04-19-query-B-row-number-top5.md
  - case/2026-04-19-query-C-window-moving-avg.md
configs:
  - spark.sql.optimizer.windowGroupLimitThreshold
  - spark.sql.optimizer.topKSortFallbackThreshold
apis:
  - SQL ROW_NUMBER OVER (PARTITION BY ... ORDER BY ...)
  - SQL RANK OVER (PARTITION BY ... ORDER BY ...)
  - SQL DENSE_RANK OVER (PARTITION BY ... ORDER BY ...)
tags: [optimizer, catalyst, window, top-n, rank, physical-inspired]
---

## L1. What & Why

`InferWindowGroupLimit`은 `Filter(rank ≤ const, Window(rank-like-fn, ...))`
패턴을 인식해 Window 노드 아래에 `WindowGroupLimit` 논리 노드를
**삽입**하는 규칙이다. 그룹별 top-N (`PARTITION BY k ORDER BY a` +
`WHERE rn ≤ N`)을 표현한 쿼리에서 Window 함수가 모든 row에 rank를
부여하기 전에 각 그룹에서 top-N row만 추려 입력 데이터를 조기 축소한다.

이 규칙은 Catalyst 규칙 중 **드문 사례**다. 대부분의 Catalyst 규칙은
plan 노드를 재배치·병합·치환하지만, 이 규칙은 **새 노드를 삽입**한다.
Filter도 Window도 그대로 남는다. 의미는 보존되며 (rank 값은 여전히
`WindowExec`가 부여), `WindowGroupLimit`은 단지 "Window가 처리할 row
수를 줄여라"는 물리 실행 힌트다. case B trace는 이를 **"physical-inspired
logical optimization"** 패턴으로 명명했다.

규칙 정의는 `InferWindowGroupLimit.scala:45` (Catalyst optimizer 패키지),
batch wiring은 `SparkOptimizer.scala:96-101` (Spark 엔진 측). 즉
**Catalyst `defaultBatches`에는 부재** — 이 규칙은 SparkOptimizer가
super.defaultBatches 위에 얹는 batch에서만 등장한다. Catalyst-순수
배포는 이 최적화를 받지 못한다. 이 분리의 의미는
[predicate-pushdown.md L4](predicate-pushdown.md#l4-performance-levers)
의 비교 표 참조.

## L2. Inputs & Outputs

**입력**: `Filter(condition, Window(windowExpressions, partitionSpec, orderSpec, child))`
형태의 LogicalPlan subtree. 추가 제약:
- 모든 `windowExpressions`가 expanding window (RowFrame, UnboundedPreceding
  → CurrentRow)
- `orderSpec.nonEmpty`
- `child`가 이미 `WindowGroupLimit`이 아님 (idempotency)
- windowExpressions 중 적어도 하나가 rank-like (`Rank | DenseRank | RowNumber`)
- condition에서 limit 값을 추출 가능 (`extractLimits` 매치)

**출력**: 위 조건 만족 시 세 갈래로 분기.

| 조건 | 출력 |
|---|---|
| `limit ≤ 0` | `LocalRelation(filter.output, Seq.empty)` (dead-code elimination) |
| RowNumber + 빈 partitionSpec + `limit < topKSortFallbackThreshold` | `Filter(condition, Limit(literal(limit), Window(...)))` (Sort+Limit 경로 — TakeOrderedAndProject과 동일 철학) |
| 그 외 (case B의 `partitionSpec=[agent_gu]` 경로) | `Filter(condition, Window(WindowGroupLimit(partitionSpec, orderSpec, fn, limit, child)))` (`WindowGroupLimit` 노드 삽입) |

조건 불만족 시 plan 변경 없음 (`filter` 그대로 반환).

### case B 관찰 — 세 번째 분기

case B raw explain의 OPTIMIZED plan (line 41-50):

```
Filter (price_rank#296 <= 5)
+- Window [row_number() ...
           windowspecdefinition(agent_gu#308, deal_amount#304 DESC NULLS LAST,
                                specifiedwindowframe(RowFrame,
                                  unboundedpreceding$(), currentrow$()))
           AS price_rank#296],
          [agent_gu#308],
          [deal_amount#304 DESC NULLS LAST]
   +- WindowGroupLimit [agent_gu#308],
                       [deal_amount#304 DESC NULLS LAST],
                       row_number(), 5     ← 새로 삽입됨
      +- ...
```

case B는 RowNumber지만 `partitionSpec = [agent_gu]`로 비어 있지 않으므로
**세 번째 분기 (`WindowGroupLimit` 삽입)** 로 진입했다. 두 번째 분기
(`Limit + Sort` 경로)는 `partitionSpec.isEmpty` 가드에서 차단된다.

### case C 부재 — 패턴 자체 미매치

case C는 `AVG(deal_amount)` over Window로, `support()` 함수
(`InferWindowGroupLimit.scala:74-77`)가 `Average`를 false로 반환 → 패턴
매치의 `windowExpressions.collect` 단계 (`:87-91`)에서 빈 목록이 만들어지고
`limits.isEmpty` 분기 (`:93`)로 즉시 `filter` 반환. plan 변경 없음. case C
Optimized plan에 `WindowGroupLimit` 노드 부재가 이로 검증된다.

## L3. Key Mechanisms

### 규칙 시그니처와 패턴 가드들

`InferWindowGroupLimit.scala:45`:

```scala
object InferWindowGroupLimit extends Rule[LogicalPlan] with PredicateHelper
```

세 가드 함수가 패턴 매치를 좁힌다.

**`support()`** (`:74-77`) — 지원 함수 화이트리스트:

```scala
private def support(windowFunction: Expression): Boolean = windowFunction match {
  case _: Rank | _: DenseRank | _: RowNumber => true
  case _ => false
}
```

오직 `Rank`, `DenseRank`, `RowNumber`만. case C의 `AVG`는
`DeclarativeAggregate` 계열이라 false.

**`isExpandingWindow()`** (`:67-72`) — frame 형태 검사:

```scala
case Alias(WindowExpression(_, WindowSpecDefinition(_, _,
SpecifiedWindowFrame(RowFrame, UnboundedPreceding, CurrentRow))), _) => true
```

`RowFrame, UnboundedPreceding, CurrentRow` — ROW_NUMBER/RANK/DENSE_RANK의
**기본 frame**. Analyzer의 `ResolveWindowOrder`가 `unspecifiedframe$()`를
이 형태로 채우므로 case B Analyzed plan에서 정확히 이 frame을 관찰
가능. ROWS BETWEEN 등 sliding frame은 미매치.

**Top-level 패턴 매치** (`:82-86`):

```scala
plan.transformWithPruning(_.containsAllPatterns(FILTER, WINDOW), ruleId) {
  case filter @ Filter(condition,
    window @ Window(windowExpressions, partitionSpec, orderSpec, child))
    if !child.isInstanceOf[WindowGroupLimit] && windowExpressions.forall(isExpandingWindow) &&
      orderSpec.nonEmpty =>
```

세 가드: idempotency (`!child.isInstanceOf[WindowGroupLimit]`),
모든 windowExpression이 expanding (`forall`), `orderSpec.nonEmpty`.
mixed window (rank + moving avg 한 노드)는 `forall` 분기에서 차단.

### conf 게이트

`InferWindowGroupLimit.scala:79-80`:

```scala
def apply(plan: LogicalPlan): LogicalPlan = {
  if (conf.windowGroupLimitThreshold == -1) return plan
  ...
}
```

`spark.sql.optimizer.windowGroupLimitThreshold`가 `-1`이면 규칙 자체가
no-op. default `1000`이라 일반적으로는 통과.

### extractLimits — 6가지 비교 패턴

`InferWindowGroupLimit.scala:50-61`:

```scala
def extractLimits(condition: Expression, attr: Attribute): Option[Int] = {
  val limits = splitConjunctivePredicates(condition).collect {
    case EqualTo(IntegerLiteral(limit), e) if e.semanticEquals(attr) => limit
    case EqualTo(e, IntegerLiteral(limit)) if e.semanticEquals(attr) => limit
    case LessThan(e, IntegerLiteral(limit)) if e.semanticEquals(attr) => limit - 1
    case GreaterThan(IntegerLiteral(limit), e) if e.semanticEquals(attr) => limit - 1
    case LessThanOrEqual(e, IntegerLiteral(limit)) if e.semanticEquals(attr) => limit
    case GreaterThanOrEqual(IntegerLiteral(limit), e) if e.semanticEquals(attr) => limit
  }
  if (limits.nonEmpty) Some(limits.min) else None
}
```

매치 가능한 6가지 비교 형태:

| SQL 패턴 | 매치 분기 | 추출 limit |
|---|---|---|
| `rn = 5` | `EqualTo(_, IntegerLiteral)` | 5 |
| `5 = rn` | `EqualTo(IntegerLiteral, _)` | 5 |
| `rn < 5` | `LessThan(_, IntegerLiteral)` | 4 |
| `5 > rn` | `GreaterThan(IntegerLiteral, _)` | 4 |
| `rn <= 5` | `LessThanOrEqual(_, IntegerLiteral)` | 5 |
| `5 >= rn` | `GreaterThanOrEqual(IntegerLiteral, _)` | 5 |

case B의 `WHERE price_rank <= 5`는 `LessThanOrEqual(e, IntegerLiteral(5))`
분기 → `5` 반환. AND로 묶인 여러 limit 조건이 있으면 **최소값** 사용
(`limits.min`). `IN`/`BETWEEN`은 매치 안 됨 — SQL 작성 시 단순 비교
형태가 안정적.

### 두 갈래 출력 분기

`InferWindowGroupLimit.scala:96-117`:

```scala
val (rowNumberLimits, otherLimits) = limits.partition(_._2.isInstanceOf[RowNumber])
// Pick RowNumber first as it's cheaper to evaluate.
val selectedLimits = if (rowNumberLimits.isEmpty) otherLimits else rowNumberLimits

selectedLimits.minBy(_._1) match {
  case (limit, rankLikeFunction) if limit <= conf.windowGroupLimitThreshold &&
    child.maxRows.forall(_ > limit) =>
    if (limit > 0) {
      val newFilterChild = if (rankLikeFunction.isInstanceOf[RowNumber] &&
        partitionSpec.isEmpty && limit < conf.topKSortFallbackThreshold) {
        // Top n (Limit + Sort) have better performance than WindowGroupLimit if the
        // window function is RowNumber and Window partitionSpec is empty.
        Limit(Literal(limit), window)
      } else {
        val windowGroupLimit =
          WindowGroupLimit(partitionSpec, orderSpec, rankLikeFunction, limit, child)
        window.withNewChildren(Seq(windowGroupLimit))
      }
      filter.withNewChildren(Seq(newFilterChild))
    } else {
      LocalRelation(filter.output, data = Seq.empty, isStreaming = filter.isStreaming)
    }
  case _ =>
    filter
}
```

분기 흐름:
1. `rowNumberLimits` 우선 선택 — RowNumber가 Rank/DenseRank보다 평가
   비용이 낮음.
2. `selectedLimits.minBy(_._1)` — 가장 작은 limit 사용 (AND conjunction
   가정).
3. `limit ≤ conf.windowGroupLimitThreshold` (default 1000) 검사 — 너무 큰
   limit은 미적용.
4. `child.maxRows.forall(_ > limit)` — child가 이미 limit보다 적은 row를
   보장하면 미적용 (의미 없음).
5. `limit > 0` → 두 갈래 출력:
   - **두 번째 분기**: `RowNumber` + `partitionSpec.isEmpty` +
     `limit < topKSortFallbackThreshold` → `Limit(literal, window)` 삽입.
     코드 주석이 명시: PARTITION BY 없는 전역 top-N이면 Sort+Limit 경로
     (TakeOrderedAndProject 철학)가 더 효율.
   - **세 번째 분기 (case B)**: `WindowGroupLimit(partitionSpec, orderSpec,
     fn, limit, child)`을 Window의 child로 삽입.
6. **`limit ≤ 0`** → `LocalRelation(empty)` 반환 (`:119-121`). 의미상
   결과가 0행임을 plan에서 직접 표현 (dead-code elimination).

### 동반 5규칙 batch + Extract Python UDFs 위치

`SparkOptimizer.scala:96-101`:

```scala
Batch("Infer window group limit", Once,
  InferWindowGroupLimit,
  LimitPushDown,
  LimitPushDownThroughWindow,
  EliminateLimits,
  ConstantFolding) :+
```

`Once` 실행. 5규칙이 의도적으로 묶여 있다.

| 규칙 | 역할 |
|---|---|
| `InferWindowGroupLimit` | `WindowGroupLimit` 또는 `Limit` 삽입 (이 페이지 주제) |
| `LimitPushDown` | 새로 생긴 `Limit`을 더 아래로 |
| `LimitPushDownThroughWindow` | `Limit`을 Window 통과시켜 push |
| `EliminateLimits` | 중복/불필요한 `Limit` 제거 |
| `ConstantFolding` | `Limit` 표현식 정리 |

Batch 위치 — `SparkOptimizer.scala:87`의 `Extract Python UDFs` batch
**다음**에 등장. 즉 Python UDF가 plan에 eval-python 노드로 삽입된 후
`InferWindowGroupLimit`이 그 위의 plan을 보고 동작한다 — 순서 의존성이
명시적으로 설계됨. `InferWindowGroupLimit`이 만든 `Limit` 노드가
같은 batch 안에서 즉시 `LimitPushDown`/`LimitPushDownThroughWindow`로
재배치 + `EliminateLimits`로 중복 제거되어, 사실상 **단일 트리 순회 안의
작은 fixed-point 효과**를 만든다.

## L4. Performance Levers

### `spark.sql.optimizer.windowGroupLimitThreshold`

- default `1000`.
- `-1` → 규칙 자체 비활성화 (`InferWindowGroupLimit.scala:80` 게이트).
- 양수 N → `limit > N`인 경우 미적용 (`:105` 가드). 의도: limit이 너무
  크면 조기 축소 효과가 작아지고 `WindowGroupLimit` 노드의 오버헤드만
  누적될 수 있다는 판단.

### `spark.sql.optimizer.topKSortFallbackThreshold`

- 두 번째 분기 (`Limit + Sort` 경로) 진입 결정.
- RowNumber + `partitionSpec.isEmpty`일 때 `limit < this`이면 Limit 경로,
  아니면 WindowGroupLimit 경로.
- 같은 설정이 `TakeOrderedAndProject` 등 다른 top-N 최적화와 공유 — case A의
  global top-N에서도 활용된다 (별도 페이지 ingest 시 검증).

### 적용 안 되는 경우들 (lever라기보다 미적용 트리거)

- 비-expanding window frame (`ROWS BETWEEN 1 PRECEDING AND 1 FOLLOWING`,
  `RANGE BETWEEN ...`) — `isExpandingWindow` false.
- rank-like 외 함수 — case C의 `AVG`, NTILE, CUME_DIST, PERCENT_RANK 등.
  `support()`가 false.
- mixed window — 한 Window 노드에 expanding + non-expanding 함수 혼재.
  `forall(isExpandingWindow)` 가드.
- `orderSpec.empty` — 의미상 rank 정의 불가.
- `child.maxRows`가 limit 이하임이 이미 plan에서 확정 — 무의미.
- Filter condition이 `extractLimits`의 6가지 비교 패턴에 매치 안 됨 (예:
  `IN`, `BETWEEN`).

## L5. How to Observe

- `df.queryExecution.optimizedPlan` — `logical.WindowGroupLimit(...)` 노드
  또는 `Limit(literal, Window(...))` 등장 관찰. case B는 전자, RowNumber +
  PARTITION BY 없는 쿼리는 후자. case B Optimized plan의 네 인자 —
  partitionSpec, orderSpec, rankLikeFunction, limit — 가
  `InferWindowGroupLimit.scala:114-115` 생성자 호출과 정확히 일치.
- case C Optimized plan에서 `WindowGroupLimit` **부재** 관찰 — `AVG`가
  rank-like 아님을 plan에서 직접 확인.
- 게이트 검사 실험: `spark.conf.set("spark.sql.optimizer.windowGroupLimitThreshold", -1)`
  로 비활성화 후 case B 재실행 → Optimized plan에서 `WindowGroupLimit`이
  사라지고 `Filter(price_rank ≤ 5)` 위에 `Window(row_number())`만 남는다.
  비활성화/활성화 비교가 이 규칙의 실효를 가장 명확히 드러낸다.
- 두 분기 비교 실험: case B의 SQL을 `PARTITION BY` 없이 재작성 (전역 top-5)
  → `Limit(literal(5), Window)` 형태로 plan이 바뀌는지 관찰. 두 갈래 분기
  (`InferWindowGroupLimit.scala:108-115`)를 직접 보는 가장 좋은 방법.
- Physical plan 후속 관찰은 [04/window-group-limit.md](../04-physical-planning/window-group-limit.md)
  참조 — `WindowGroupLimit` 논리 노드가 `WindowGroupLimitExec` Partial +
  Final 두 물리 노드로 분할되는 strategy.

> 📝 probe 페이지 미작성. 후속 `/probe 03-logical-optimization/infer-window-group-limit`
> 대상 — "비활성화/활성화 비교 + 두 갈래 분기 (Limit vs WindowGroupLimit)
> 관찰 + 5규칙 동반 batch의 fixed-point 효과 추적".
