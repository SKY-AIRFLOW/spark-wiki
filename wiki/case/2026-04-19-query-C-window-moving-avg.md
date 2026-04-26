---
stage: case
title: "Query C: 서울 자치구별 월간 거래금액 6개월 이동평균 (CTE + Window)"
spark_versions: ["3.5.3"]
sources:
  - raw/explain/2026-04-19-query-C-window-moving-avg.txt
  - raw/spark-source/v3.5.6/sql/core/src/main/scala/org/apache/spark/sql/execution/SparkStrategies.scala
last_verified: 2026-04-26
confidence: verified
related:
  - 00-overview/execution-pipeline.md
  - 02-analysis/analyzer-overview.md
  - 03-logical-optimization/column-pruning.md
  - 03-logical-optimization/cte-inline.md
  - 03-logical-optimization/infer-filters-from-constraints.md
  - 03-logical-optimization/infer-window-group-limit.md
  - 03-logical-optimization/like-simplification.md
  - 03-logical-optimization/predicate-pushdown.md
  - 04-physical-planning/ensure-requirements.md
  - 04-physical-planning/partitioning-compatibility.md
  - 04-physical-planning/hash-aggregate-partial-final.md
  - 04-physical-planning/batch-scan-iceberg.md
  - 04-physical-planning/window-exec.md
  - 04-physical-planning/window-group-limit.md
  - 05-cost-and-aqe/aqe-overview.md
  - 05-cost-and-aqe/leveraging-statistics.md
  - case/2026-04-19-query-A-self-join-growth.md
  - case/2026-04-19-query-B-row-number-top5.md
configs:
  - spark.sql.adaptive.enabled
  - spark.sql.shuffle.partitions
apis:
  - SQL WITH / CTE
  - SQL AVG() OVER (PARTITION BY ... ORDER BY ... ROWS BETWEEN ... AND ...)
  - SQL GROUP BY + aggregate functions
tags: [case, cte, window, aggregate-over-window, moving-average, bounded-frame, iceberg, aqe]
---

## Query 전문

Plan 트리로부터 재구성한 SQL (실제 제출 SQL과 의미적 동치):

```sql
WITH monthly_avg AS (
  SELECT agent_gu, deal_year, deal_month,
         AVG(deal_amount) AS avg_amount,
         COUNT(1)         AS trade_count
  FROM nessie.real_estate.apt_trades
  WHERE agent_gu LIKE '서울%'
  GROUP BY agent_gu, deal_year, deal_month
)
SELECT agent_gu, deal_year, deal_month, avg_amount, trade_count,
       AVG(avg_amount) OVER (
         PARTITION BY agent_gu
         ORDER BY deal_year, deal_month
         ROWS BETWEEN 5 PRECEDING AND CURRENT ROW
       ) AS moving_avg_6m
FROM monthly_avg
ORDER BY agent_gu, deal_year, deal_month
```

"서울 자치구별로 월간 평균 거래금액을 구한 뒤, 각 월에 대해 **최근 6개월
(현재 포함)** 이동평균을 계산" 쿼리.

## Version notes

- 캡처 환경: **Spark 3.5.3** (Application `local-1776575126027`, 2026-04-19
  — 쿼리 A, B와 **같은 세션**).
- 인용 Spark 소스: v3.5.6 `SparkStrategies.scala` (Window, WindowGroupLimit
  strategies). 3.5 마이너 패치 차이 무영향 추정.
- `InlineCTE`, `CTESubstitution` Catalyst rule은 어제 ingest 범위 밖 —
  CTE가 inline됐다는 사실은 plan 관찰로 확정, 규칙 내부 조건은 잠정.
- Iceberg: `iceberg-spark-runtime-3.5_2.12:1.7.1` (A, B와 동일).

## 캡처 환경 설정 (A, B와 동일)

| 설정 | 값 |
|---|---|
| `spark.sql.adaptive.enabled` | `true` (단 `isFinalPlan=false` — 아래 05 참고) |
| `spark.sql.shuffle.partitions` | `200` |
| `spark.sql.autoBroadcastJoinThreshold` | `10485760b` — 무관 (JOIN 없음) |
| `spark.sql.join.preferSortMergeJoin` | `true` — 무관 |

---

## 쿼리 A, B와의 비교 — 3쿼리 대조

### 표 1: 구조 대조

| 항목 | Query A (self-join-growth) | Query B (row-number-top5) | Query C (window-moving-avg) |
|---|---|---|---|
| 핵심 논리 구조 | self-join + aggregate | 단일 scan + rank Window | CTE + group aggregate + aggregate-over-Window |
| 테이블 접근 | `apt_trades` × 2 (curr 2024 / prev 2023) | `apt_trades` × 1 (2024만) | `apt_trades` × 1 (서울만, 모든 연도) |
| 결과 형태 | ORDER BY + LIMIT 20 (전역 Top-20) | WHERE rank ≤ 5 + ORDER BY (그룹별 Top-5) | ORDER BY (그룹별 시계열 + 이동평균) |
| Window 여부 | 없음 | `ROW_NUMBER()` (rank-like) | `AVG()` over window (aggregate over window) |
| 집계 여부 | `HashAggregate` (GROUP BY agent_gu, legal_dong) | 없음 | `HashAggregate` (GROUP BY agent_gu, deal_year, deal_month) |
| JOIN | SortMergeJoin | 없음 | 없음 |
| CTE | 없음 | 없음 | 단일 참조 CTE (Optimized에서 inline) |
| Top-N 최적화 | `TakeOrderedAndProject` (전역) | `WindowGroupLimit` (그룹별) | 해당 없음 (제한 없음) |

### 표 2: Exchange 개수와 종류

| 쿼리 | Exchange 개수 | 세부 내역 |
|---|:-:|---|
| A | 2 | `hashpartitioning(legal_dong, 200)` × 2 (SMJ 양쪽) |
| B | 2 | `hashpartitioning(agent_gu, 200)` (WGL Partial → Final) + `rangepartitioning(agent_gu ASC, price_rank ASC, 200)` (outer ORDER BY) |
| **C** | **3** | `hashpartitioning(agent_gu, deal_year, deal_month, 200)` (HashAgg Partial → Final) + `hashpartitioning(agent_gu, 200)` (HashAgg → Window) + `rangepartitioning(agent_gu, deal_year, deal_month, 200)` (outer ORDER BY) |

**쿼리 C가 Exchange 가장 많음**. 핵심 이유: **aggregate의 GROUP BY 키
(`agent_gu, deal_year, deal_month`)**와 **Window의 PARTITION BY 키
(`agent_gu`)가 서로 다름**. Catalyst는 앞선 `hashpartitioning(a, b, c)`가
`a` 단독 분포를 보장하지 않음을 알아 **재분배를 강제**한다. 이것이 아래 04
단계의 메인 발견.

### 표 3: Statistics 대조 (EXPLAIN COST)

| 쿼리 | 시작 Relation | 중간 최대 추정 | 최종 | 특이점 |
|---|---|---|---|---|
| A | 34.5 MiB + 14.2 MiB (두 side) | **413.3 TiB** (Join) | 1.6 KB (20 row) | **JOIN fallback 폭발** — column histogram 없으면 `leftRows × rightRows` 상한 추정 |
| B | 30.5 MiB | 30.5 MiB | 30.5 MiB | 단조 흐름, 현실적 |
| C | 16.2 MiB | 24.4 MiB (Window) | 24.4 MiB | **Aggregate 후 size 증가 현상** (21.1 MiB) — Catalyst가 row 수 축소를 과소평가하고 중간 컬럼 추가만 반영 |

**관찰 — Catalyst cardinality estimator의 세 얼굴:**
- A: column histogram 부재 시 join의 worst-case 상한이 얼마나 왜곡되는가.
- B: row 수 변화가 예측 가능할 때는 추정이 정확.
- C: aggregate의 row 수 축소 효과가 통계에 반영되지 않음 (group cardinality
  미지). 실제로는 서울 25구 × 연월 조합 ≈ 900 row지만 Catalyst는 21.1 MiB
  상한을 유지.

세 현상 모두 "column-level histogram이 있으면 크게 개선될 여지"를 공통
시사. 참조: [05-cost-and-aqe/leveraging-statistics.md](../05-cost-and-aqe/leveraging-statistics.md).

### 표 4: WindowGroupLimit 적용 여부

| 쿼리 | 적용 여부 | 근거 |
|---|---|---|
| A | 해당 없음 | Window 자체 없음 |
| B | **적용** | `Filter(price_rank ≤ 5) over Window(row_number())` 패턴 매칭 → Optimizer가 `logical.WindowGroupLimit` 삽입 → strategy가 Partial + Final로 분할 |
| C | **미적용** | `AVG()`는 rank-like 함수 아님 → `InferWindowGroupLimit`의 `support()` 분기 ([03/infer-window-group-limit.md](../03-logical-optimization/infer-window-group-limit.md))가 `Rank|DenseRank|RowNumber`만 매치 → `logical.WindowGroupLimit` 노드 삽입 안 됨 → strategy 발동 조건 불충족 |

**verified 소스 근거** (`raw/spark-source/v3.5.6/sql/core/.../SparkStrategies.scala:657-667`):

```scala
object WindowGroupLimit extends Strategy {
  def apply(plan: LogicalPlan): Seq[SparkPlan] = plan match {
    case logical.WindowGroupLimit(partitionSpec, orderSpec,
                                  rankLikeFunction, limit, child) => ...
    case _ => Nil
  }
}
```

strategy의 match가 `logical.WindowGroupLimit` 노드 존재 여부만 본다. 그
노드는 **rank-like function + limit filter**가 있을 때만 upstream
Optimizer rule이 삽입 — C는 두 조건 모두 불충족 (AVG이 rank-like 아님, 어떤
filter도 없음). 따라서 쿼리 C의 `AVG() OVER` 는 일반 `WindowExec` 경로로
실행. 참조: [04-physical-planning/window-group-limit.md](../04-physical-planning/window-group-limit.md).

### Skeleton 생성 추세

세 쿼리를 순차 trace하며 신규 skeleton 생성 개수:

| Trace | 신규 skeleton | 재인용 skeleton |
|---|:-:|:-:|
| A | 4 | 0 (최초) |
| B | 2 | 4 (A의 4개 모두) |
| **C** | **1** | **6** (A 4 + B 2 모두) |

**건강한 추세**: Wiki가 성숙할수록 신규 생성이 줄고 재인용이 늘어난다.
첫 trace에서 만든 기초 어휘 (analyzer-overview, predicate-pushdown,
hash-aggregate-partial-final, batch-scan-iceberg)가 세 쿼리 모두에서 공통
재료가 됨. B의 window-exec / window-group-limit도 C에서 재인용 (특히
window-group-limit은 "미적용" 대조로).

다음 trace부터는 신규 skeleton 0개로 진행될 여지도 있다. skeleton 생성이
필요 없다는 신호는 "현재 Wiki의 개념 집합이 이 쿼리를 설명하기에 충분"의
지표.

---

## 파이프라인 요약

| 단계 | 이 쿼리에서 일어나는 일 | Wiki 참조 |
|---|---|---|
| 01 Parsing | `CTE [monthly_avg]` 루트 + CTE body (`'Aggregate → 'Filter → 'UnresolvedRelation`) + main query (`'Sort → 'Project → 'UnresolvedRelation [monthly_avg]`). | (skeleton 대기) |
| 02 Analysis | `WithCTE` + `CTERelationDef 0, false` + `CTERelationRef 0, true, [cols], false`로 CTE 구조 명시화. exprId 부여. Window의 frame은 `specifiedwindowframe(RowFrame, -5, currentrow$())`로 확정. | [02/analyzer-overview.md](../02-analysis/analyzer-overview.md) |
| 03 Logical Optimization | **CTE inline** (WithCTE/Def/Ref 완전 소거, 단일 참조). Column pruning (17→4), predicate pushdown (`LIKE → StartsWith`, `IS NOT NULL`). **WindowGroupLimit 미적용** (AVG는 rank-like 아님). | [03/cte-inline.md](../03-logical-optimization/cte-inline.md), [03/predicate-pushdown.md](../03-logical-optimization/predicate-pushdown.md) |
| 04 Physical Planning | `HashAggregate` Partial + Final (3-key hashpartitioning shuffle). **Final HashAgg → Window 사이 추가 Exchange** (`hashpartitioning(agent_gu)`) — aggregate key ⊋ window partition key. **이 case의 verified ground truth**는 [04/partitioning-compatibility.md](../04-physical-planning/partitioning-compatibility.md) 박스 ① (`partitioning.scala:283-290`) + L4 일반화 박스 ("structural 한계"). `Window` strategy → `WindowExec` (frame = RowFrame -5 ~ CURRENT ROW). Outer ORDER BY용 rangepartitioning. | [04/ensure-requirements.md](../04-physical-planning/ensure-requirements.md), [04/partitioning-compatibility.md](../04-physical-planning/partitioning-compatibility.md), [04/hash-aggregate-partial-final.md](../04-physical-planning/hash-aggregate-partial-final.md), [04/window-exec.md](../04-physical-planning/window-exec.md), [04/batch-scan-iceberg.md](../04-physical-planning/batch-scan-iceberg.md), [04/window-group-limit.md](../04-physical-planning/window-group-limit.md) (미적용 대조) |
| 05 Cost & AQE | `AdaptiveSparkPlan isFinalPlan=false`. Statistics 16.2 → 21.1 → 24.4 MiB (aggregate 후 증가 현상). | [05/aqe-overview.md](../05-cost-and-aqe/aqe-overview.md), [05/leveraging-statistics.md](../05-cost-and-aqe/leveraging-statistics.md) |
| 06 Codegen | 추측 금지. `WindowFunctionFrame`의 bounded frame ring buffer 구현은 소스 필요. | (별도 trace) |
| 07 RDD & Stages | Exchange **3개** → shuffle map stage 3개 + 결과 stage. 세 쿼리 중 가장 많음. | (Exchange 기반 추정) |
| 08 Scheduling | 이 trace 범위 밖. | — |
| 09 Execution | 이 trace 범위 밖. | — |

---

## 01 Parsing

### Plan 트리 (raw 01 PARSED)

```
CTE [monthly_avg]
:  +- 'SubqueryAlias monthly_avg
:     +- 'Aggregate ['agent_gu, 'deal_year, 'deal_month],
:          ['agent_gu, 'deal_year, 'deal_month,
:           'AVG('deal_amount) AS avg_amount#364,
:           'COUNT(1) AS trade_count#365]
:        +- 'Filter 'agent_gu LIKE 서울%
:           +- 'UnresolvedRelation [nessie, real_estate, apt_trades],
:                                  [], false
+- 'Sort ['agent_gu ASC NULLS FIRST,
          'deal_year ASC NULLS FIRST,
          'deal_month ASC NULLS FIRST], true
   +- 'Project ['agent_gu, 'deal_year, 'deal_month,
                'avg_amount, 'trade_count,
                'AVG('avg_amount) windowspecdefinition(
                   'agent_gu,
                   'deal_year ASC NULLS FIRST,
                   'deal_month ASC NULLS FIRST,
                   specifiedwindowframe(RowFrame, -5, currentrow$()))
                AS moving_avg_6m#363]
      +- 'UnresolvedRelation [monthly_avg], [], false
```

### Java 개발자 관점

- **`CTE [...]` 루트 노드**: parser가 `WITH name AS (...)` 구문을 별도 루트
  노드로 표현. A/B에는 등장하지 않은 구조. Java로 치면 class 안의 named
  inner method declaration + 그 method의 call site가 따로 있는 구조.
- **두 개의 `'UnresolvedRelation`**: 하나는 실제 테이블 (`[nessie,
  real_estate, apt_trades]`), 다른 하나는 **CTE 이름** (`[monthly_avg]`).
  parser는 CTE 이름도 UnresolvedRelation으로 일관되게 표현 — Analyzer가
  해결 단계에서 이름을 보고 CTE 정의로 bind할지 카탈로그 테이블로 bind할지
  결정.
- **frame spec `specifiedwindowframe(RowFrame, -5, currentrow$())`**:
  사용자 SQL의 `ROWS BETWEEN 5 PRECEDING AND CURRENT ROW`가 parser 단계에서
  이미 내부 표현으로 변환됨. `-5`는 음의 offset = preceding. Java 관점:
  사용자 구문의 `5 PRECEDING`은 parser가 `-5`의 signed int offset으로
  정규화 (내부 frame API의 일관된 표현).
- 쿼리 B의 frame `specifiedwindowframe(RowFrame, unboundedpreceding$(), currentrow$())`
  와 대조: B는 unbounded의 **특수 마커** 객체, C는 정수 offset. 구현
  경로가 이미 parse 단계에서 갈린다 (`WindowFunctionFrame`의 growing vs
  sliding 구현).

---

## 02 Analysis

### Plan 트리 (raw 02 ANALYZED)

```
WithCTE
:- CTERelationDef 0, false
:  +- SubqueryAlias monthly_avg
:     +- Aggregate [agent_gu#377, deal_year#370, deal_month#371],
:          [agent_gu#377, deal_year#370, deal_month#371,
:           avg(deal_amount#373) AS avg_amount#364,
:           count(1) AS trade_count#365L]
:        +- Filter agent_gu#377 LIKE 서울%
:           +- SubqueryAlias nessie.real_estate.apt_trades
:              +- RelationV2[apt_name#366, ..., deal_date#382]
:                   nessie.real_estate.apt_trades ...    # 17 cols
+- Sort [agent_gu#377 ASC NULLS FIRST,
         deal_year#370 ASC NULLS FIRST,
         deal_month#371 ASC NULLS FIRST], true
   +- Project [agent_gu#377, deal_year#370, deal_month#371,
               avg_amount#364, trade_count#365L, moving_avg_6m#363]
      +- Project [agent_gu#377, deal_year#370, deal_month#371,
                  avg_amount#364, trade_count#365L,
                  moving_avg_6m#363, moving_avg_6m#363]     # 이중 Project
         +- Window [avg(avg_amount#364) windowspecdefinition(
                      agent_gu#377,
                      deal_year#370 ASC NULLS FIRST,
                      deal_month#371 ASC NULLS FIRST,
                      specifiedwindowframe(RowFrame, -5, currentrow$()))
                    AS moving_avg_6m#363],
                   [agent_gu#377],                          # partitionSpec
                   [deal_year#370 ASC NULLS FIRST,          # orderSpec
                    deal_month#371 ASC NULLS FIRST]
            +- Project [agent_gu#377, deal_year#370, deal_month#371,
                        avg_amount#364, trade_count#365L]
               +- SubqueryAlias monthly_avg
                  +- CTERelationRef 0, true,
                       [agent_gu#377, deal_year#370, deal_month#371,
                        avg_amount#364, trade_count#365L], false
```

### Java 개발자 관점

- **`WithCTE` + `CTERelationDef 0` + `CTERelationRef 0`의 삼중주**: 같은
  `id=0`으로 정의와 참조가 연결됨. Java `private static final` 필드의
  선언과 call site가 분리된 것과 유사 — 정의와 참조는 문법적으로 떨어져
  있지만 같은 identity로 묶임.
- `CTERelationDef 0, false` 에서 `false`는 resolved flag (CTE 내부
  resolution 진행 상황). `CTERelationRef 0, true, [cols], false`에서 두
  번째 `true`는 참조가 해결됨, 네 번째 `false`는 recursive 아님.
- **CTERelationRef의 `[cols]` 목록**: CTE 정의가 밖으로 노출하는 컬럼들.
  이 컬럼들의 exprId는 CTE body의 Aggregate output과 **같다** (e.g.
  `avg_amount#364`가 CTE body에서 생성되고 Ref에서도 같은 `#364`). Java
  관점: private method의 반환 값 reference가 동일 객체 identity를 공유.
- **이중 Project** (`+- Project [..., moving_avg_6m#363] +- Project [..., moving_avg_6m#363, moving_avg_6m#363]`):
  쿼리 B와 같은 패턴. Window가 자신이 생성한 `moving_avg_6m` 컬럼을 output
  에 추가한 상태를 Analyzer가 중간 Project로 표현. CollapseProject가
  Optimized에서 병합.
- **`SubqueryAlias nessie.real_estate.apt_trades`**: Analyzer가 자동
  주입하는 정규화된 테이블 경로 alias. `SubqueryAlias monthly_avg`는
  사용자 CTE 이름. 두 종류의 SubqueryAlias가 이 plan에 공존.

---

## 03 Logical Optimization

### Plan 트리 (raw 03 OPTIMIZED)

```
Sort [agent_gu#377 ASC NULLS FIRST,
      deal_year#370 ASC NULLS FIRST,
      deal_month#371 ASC NULLS FIRST], true
+- Window [avg(avg_amount#364) windowspecdefinition(
             agent_gu#377,
             deal_year#370 ASC NULLS FIRST,
             deal_month#371 ASC NULLS FIRST,
             specifiedwindowframe(RowFrame, -5, currentrow$()))
           AS moving_avg_6m#363],
          [agent_gu#377],
          [deal_year#370 ASC NULLS FIRST,
           deal_month#371 ASC NULLS FIRST]
   +- Aggregate [agent_gu#377, deal_year#370, deal_month#371],
        [agent_gu#377, deal_year#370, deal_month#371,
         avg(deal_amount#373) AS avg_amount#364,
         count(1) AS trade_count#365L]
      +- Filter (isnotnull(agent_gu#377)
                 AND StartsWith(agent_gu#377, 서울))
         +- RelationV2[deal_year#370, deal_month#371, deal_amount#373,
                       agent_gu#377]
              nessie.real_estate.apt_trades
```

### Analyzed 대비 변경점

| 변경 | Analyzed | Optimized |
|---|---|---|
| `WithCTE`, `CTERelationDef`, `CTERelationRef` | 존재 | **모두 소거** (inline) |
| SubqueryAlias | 2개 (monthly_avg, 테이블 경로) | 모두 제거 |
| 이중 Project | 존재 | `CollapseProject`로 병합 |
| RelationV2 컬럼 수 | 17 | **4** (`ColumnPruning`) |
| WHERE filter | `agent_gu LIKE 서울%` | `isnotnull(agent_gu) AND StartsWith(agent_gu, 서울)` |
| WindowGroupLimit | 없음 | **여전히 없음** (매칭 조건 불충족) |

### Java 개발자 관점 — CTE inline이 특별한 이유

- **Inline 전**: CTE body가 독립 subtree. 상위 plan은 CTE body의 output만
  알고 body 내부 구조는 모른다. Catalyst의 최적화 규칙들은 이 경계를 넘지
  못한다 (예: main query의 projection이 "CTE body의 Aggregate가 어떤 컬럼
  을 쓰는지"를 알 수 없다).
- **Inline 후**: CTE body가 main query의 일부가 되어 경계가 사라진다.
  Column Pruning이 CTE body 안쪽까지 침투해서 17 → 4 컬럼으로 줄인다.
  Predicate Pushdown도 같은 방식으로 경계를 넘어 RelationV2 바로 위에
  `Filter`를 안착시킨다.

Java 관점: `private static` helper method를 호출 지점에 inline하면
JIT compiler가 caller의 context와 함께 escape analysis를 할 수 있어
추가 최적화 (stack allocation, dead store elimination 등)가 열리는 것과
같다. 호출 경계는 최적화 장벽이다.

참조: [03-logical-optimization/cte-inline.md](../03-logical-optimization/cte-inline.md).

### WindowGroupLimit가 적용되지 않는 이유 (verified 소스 근거)

Optimized plan에 `WindowGroupLimit` 노드가 없음. 조건 확인:
- 이 쿼리에는 `Filter(rank ≤ N)` 패턴 없음 — 애초에 rank cutoff 없음.
- Window function이 `AVG()` — rank-like function이 아님.

`SparkStrategies.scala:657-667`의 strategy는 `logical.WindowGroupLimit`
노드를 찾는데, 그 노드는 upstream Catalyst rule `InferWindowGroupLimit`
([03/infer-window-group-limit.md](../03-logical-optimization/infer-window-group-limit.md))이
삽입한다. rule의 삽입 조건 (rank-like function + limit filter)은
쿼리 C에서 둘 다 불충족 → 노드 없음 → strategy 발동 안 됨.

참조: [04-physical-planning/window-group-limit.md](../04-physical-planning/window-group-limit.md).

### 적용된 최적화 규칙 (관찰 기반)

| 규칙 (추정) | 관찰 증거 |
|---|---|
| `CTESubstitution` (analysis/early-opt) | Parsed의 `CTE [...]` → Analyzed의 `WithCTE` + `CTERelationDef` + `CTERelationRef` |
| **`InlineCTE`** | Optimized에서 `WithCTE`, `CTERelationDef`, `CTERelationRef` 모두 소거 |
| `CollapseProject` | 이중 Project 병합 + SubqueryAlias 제거 |
| `ColumnPruning` | RelationV2 17 → 4 컬럼 |
| `InferFiltersFromConstraints` | `isnotnull(agent_gu)` 자동 추가 |
| `LikeSimplification` | `LIKE '서울%'` → `StartsWith(agent_gu, 서울)` |
| `PushDownPredicates` | Filter가 RelationV2 바로 위로 (CTE 경계 사라졌으므로) |

> 📝 `InlineCTE`, `CTESubstitution` 규칙 정의는 Optimizer.scala + Analyzer.scala
> 영역 — 어제 ingest 범위 밖. 정확한 rule 이름, inline 결정 조건 (참조 횟수
> threshold, deterministic 여부, cost heuristic), recursive CTE 처리 방식은
> 소스 ingest 후 기록.

---

## 04 Physical Planning — 이 case의 메인 분량

### Plan 트리 (raw 04 PHYSICAL — Pre-AQE)

```
Sort [agent_gu#377 ASC NULLS FIRST,
      deal_year#370 ASC NULLS FIRST,
      deal_month#371 ASC NULLS FIRST], true, 0
+- Window [avg(avg_amount#364) ... AS moving_avg_6m#363],
          [agent_gu#377],
          [deal_year#370 ASC NULLS FIRST,
           deal_month#371 ASC NULLS FIRST]
   +- HashAggregate(keys=[agent_gu#377, deal_year#370, deal_month#371],
                    functions=[avg(deal_amount#373), count(1)], ...)  # Final
      +- HashAggregate(keys=[agent_gu#377, deal_year#370, deal_month#371],
                       functions=[partial_avg(deal_amount#373),
                                  partial_count(1)], ...)               # Partial
         +- Project [deal_year#370, deal_month#371,
                     deal_amount#373, agent_gu#377]
            +- Filter (isnotnull(agent_gu#377)
                       AND StartsWith(agent_gu#377, 서울))
               +- BatchScan nessie.real_estate.apt_trades
                     [deal_year#370, deal_month#371,
                      deal_amount#373, agent_gu#377]
                     (branch=null)
                     [filters=agent_gu IS NOT NULL,
                              agent_gu LIKE '서울%',
                              groupedBy=]
                     RuntimeFilters: []
```

### Plan 트리 (raw Executed — `EnsureRequirements` 적용 후)

```
AdaptiveSparkPlan isFinalPlan=false
+- Sort [agent_gu#377 ASC, deal_year#370 ASC, deal_month#371 ASC], true, 0
   +- Exchange rangepartitioning(agent_gu ASC, deal_year ASC, deal_month ASC,
                                 200),
               ENSURE_REQUIREMENTS, [plan_id=511]                # Exchange #3
      +- Window [avg(avg_amount#364) ... AS moving_avg_6m#363],
                [agent_gu#377],
                [deal_year#370 ASC, deal_month#371 ASC]
         +- Sort [agent_gu ASC, deal_year ASC, deal_month ASC], false, 0
            +- Exchange hashpartitioning(agent_gu#377, 200),
                        ENSURE_REQUIREMENTS, [plan_id=507]       # Exchange #2
               +- HashAggregate(keys=[agent_gu#377, deal_year#370, deal_month#371],
                                functions=[avg(...), count(1)], ...)  # Final
                  +- Exchange hashpartitioning(agent_gu#377, deal_year#370,
                                               deal_month#371, 200),
                              ENSURE_REQUIREMENTS, [plan_id=504]  # Exchange #1
                     +- HashAggregate(keys=[...],
                                      functions=[partial_avg(...),
                                                 partial_count(1)], ...)  # Partial
                        +- Filter (isnotnull(agent_gu#377)
                                   AND StartsWith(agent_gu#377, 서울))
                           +- BatchScan ... [filters=..., groupedBy=]
```

### 세 Exchange의 해설 — 이 쿼리의 핵심 학습 포인트

**Exchange #1** — `plan_id=504`, `hashpartitioning(agent_gu, deal_year, deal_month, 200)`

HashAggregate Partial → Final 사이. Partial이 각 input partition에서 그룹
키로 부분 집계를 누적한 뒤, Final이 **같은 그룹 키의 모든 row를 한
partition에 모으려고** shuffle 필요. GROUP BY 키 **3개 전체**로 해시.

쿼리 A의 HashAggregate 구조와 동일한 패턴. 참조:
[04-physical-planning/hash-aggregate-partial-final.md](../04-physical-planning/hash-aggregate-partial-final.md).

**Exchange #2** — `plan_id=507`, `hashpartitioning(agent_gu, 200)` — **⭐ 핵심**

Final HashAggregate → Window 사이. 여기서 **또 하나의 shuffle이 발생**.

**왜 이 Exchange가 필요한가? — verified ground truth**

- Exchange #1의 결과: data가 **`(agent_gu, deal_year, deal_month)`** 튜플의
  해시로 partition되어 있다 — `outputPartitioning =
  HashPartitioning([agent_gu, deal_year, deal_month], 200)`.
- Window가 요구하는 것: `requiredChildDistribution =
  ClusteredDistribution([agent_gu], requireAllClusterKeys=false, None)`.
- [`EnsureRequirements`](../04-physical-planning/ensure-requirements.md)의
  단일 자식 distribution check (`EnsureRequirements.scala:71-80`)에서
  `child.outputPartitioning.satisfies(distribution)` 호출:
  1. `Partitioning.satisfies` (`partitioning.scala:218-220`) —
     `requiredNumPartitions=None`이므로 `forall`은 vacuously true →
     `satisfies0` 위임.
  2. `HashPartitioningLike.satisfies0` (`partitioning.scala:276-294`) —
     [04/partitioning-compatibility.md](../04-physical-planning/partitioning-compatibility.md)
     **박스 ①**의 false 분기:
     - `requireAllClusterKeys=false` (default) 분기 (`:289`):
       `expressions.forall(x => requiredClustering.exists(_.semanticEquals(x)))`.
     - `expressions = [agent_gu, deal_year, deal_month]`,
       `requiredClustering = [agent_gu]`.
     - `agent_gu` ∈ `[agent_gu]` → true. **`deal_year` ∉ `[agent_gu]`** → false.
       `forall` short-circuit → **false**.
     - (`requireAllClusterKeys=true` 분기도 `length 3 != 1`로 false.)
- → `satisfies` false → `distribution.createPartitioning(200)` 호출 →
  `HashPartitioning([agent_gu], 200)` 신규 생성 → `ShuffleExchangeExec`로
  자식을 감싸 Exchange #2 삽입.
- 이 패턴은 단발 사례가 아니라 **structural 한계** — [04/partitioning-compatibility.md](../04-physical-planning/partitioning-compatibility.md)
  L4 일반화 박스: "HashAggregate output partitioning은 hash agg key 전체로 결정
  → 하위 Window가 PARTITION BY를 그 키의 strict subset으로 사용하면 항상
  superset 케이스 → 재shuffle 불가피".

**Java 개발자 관점:**

```java
// Exchange #1 후 상태:
Map<Tuple3<String, Int, Int>, Agg> partitioned =
    stream.collect(Collectors.groupingByConcurrent(
        row -> Tuple3.of(row.agent_gu, row.year, row.month),
        MyCollector.aggregate()));

// Window가 요구하는 형태:
Map<String, List<Row>> byAgentGu =
    stream.collect(Collectors.groupingByConcurrent(
        row -> row.agent_gu));

// 전자에서 후자로 가는 건 단순 rename이 아니라 실제 재분배 필요.
// hash(String) ≠ hash((String, Int, Int)).first() 이므로
// 같은 agent_gu 문자열이라도 이미 여러 bucket에 흩어져 있다.
```

**관계 정확화 (2026-04-26 정정)** — 본 case 작성 당시 표현 "같은 키의
prefix라도 hashpartitioning을 재사용할 수 없다"는 의미상 정확하지만, 정확한
용어는 [04/partitioning-compatibility.md](../04-physical-planning/partitioning-compatibility.md)
박스 ①의 표 기준으로 다음과 같이 표현된다:

| 관계 | satisfies (default `requireAllClusterKeys=false`) |
|---|---|
| partition keys ⊆ distribution keys (subset) | **true** — 더 약한 partition이 더 강한 clustering을 자동 만족 |
| partition keys = distribution keys (정확 일치) | true |
| **partition keys ⊋ distribution keys (superset)** ★ 본 case | **false** → 재shuffle |
| 둘 다 다른 원소 (교집합만) | false |

쿼리 C는 `partition keys [agent_gu, deal_year, deal_month] ⊋ [agent_gu]
= distribution keys`인 **superset 케이스**. `HashPartitioning(a, b, c)`의
해시 bucket 내부에 같은 `a` 값의 row가 모여 있어도 같은 `a`가 다른 bucket에도
흩어져 있을 수 있기 때문에 `ClusteredDistribution([a])` 요구를 만족하지 못한다.

> ⚠️ **2026-04-26 ingest 사이클의 folklore 차단 사례.** 이전 본문 ("특수 경로가
> 일부 있지만", "정확한 key 집합 일치 요구")는 부정확했다. v3.5.6
> `HashPartitioningLike.satisfies0`(`partitioning.scala:276-294`)는 default에서
> **subset 또는 정확 일치**를 satisfies하고 superset은 satisfies하지 않는다 —
> "정확한 일치 요구"가 아니다. 정정된 표현은 위 표.

**Exchange #3** — `plan_id=511`, `rangepartitioning(agent_gu ASC, deal_year ASC, deal_month ASC, 200)`

Window → 외부 ORDER BY 사이. 쿼리 B와 같은 outer global sort를 위한
rangepartitioning. 참조: 쿼리 B case 파일의 04 섹션.

### 세 쿼리의 Exchange 비교 (Java 관점)

| 쿼리 | Exchange 1 | Exchange 2 | Exchange 3 |
|---|---|---|---|
| A | `hash(legal_dong)` SMJ 좌 | `hash(legal_dong)` SMJ 우 | — |
| B | `hash(agent_gu)` WGL Partial→Final | `range(agent_gu, price_rank)` outer sort | — |
| C | `hash(agent_gu, year, month)` HashAgg P→F | `hash(agent_gu)` HashAgg→Window | `range(agent_gu, year, month)` outer sort |

**공통점:**
- 모든 Exchange는 `ENSURE_REQUIREMENTS` 마커 — 사용자 지정 아닌 자동 삽입.
- 모두 `EnsureRequirements` rule의 책임. 이 rule은 plan의 각 노드가 요구
  하는 distribution과 ordering을 아래 자식의 output과 비교해 불일치 시
  Exchange/Sort를 삽입하는 **invariant 강제** 메커니즘.

**쿼리 C 고유:**
- Exchange 3개 중 Exchange #2는 "aggregate key ⊋ window partition key"
  불일치가 원인. 이 불일치 패턴은 광범위하게 나타남 — "서로 다른 level의
  집계/정렬이 이어지는 모든 분석 쿼리"에서 발생. 일반화 가치가 높음.

### Bounded frame `-5`의 해설

Window 노드의 frame:
```
specifiedwindowframe(RowFrame, -5, currentrow$())
```

- `RowFrame`: row 개수 기준 (RangeFrame이면 ORDER BY value 기준).
- `-5`: 현재 row로부터 5 row 앞 (음수 = preceding).
- `currentrow$()`: 현재 row.

따라서 frame은 `[current-5, current]`의 **6-row sliding window**. 쿼리 B의
`unboundedpreceding$()`와 대조되는 **bounded frame**.

**Java 관점 — running accumulator vs sliding window:**
- Unbounded (B의 `ROW_NUMBER` frame): `long counter = 0; counter++;` — state
  하나로 충분. 각 row마다 증가만.
- Bounded RowFrame (C): `Deque<Double> last6 = new ArrayDeque<>(6);` —
  window가 이동할 때마다 oldest를 `pollFirst()`, newest를 `offerLast()`,
  sum은 incremental update. O(1) per row.

`WindowFunctionFrame` 구현은 이 두 패턴을 다른 클래스로 분리. 구체 구현은
소스 ingest 필요.

> 📝 `sql/core/.../execution/window/WindowFunctionFrame.scala` ingest 대기.
> bounded frame의 incremental update 전략 (AVG의 경우 `sum` 과 `count`를
> 유지하며 window 이동 시 가감), spill 여부, frame 크기 상한은 소스 확인.

### BatchScan filters pushdown (A, B와 비교)

```
[filters=agent_gu IS NOT NULL, agent_gu LIKE '서울%', groupedBy=]
```

A/B에 있던 `deal_year IS NOT NULL, deal_year = <year>`가 **없다**. 이유:
쿼리 C는 연도 필터가 없음 — `deal_year`는 grouping key일 뿐 WHERE 조건
아님. 모든 연도의 월별 이동평균을 구하는 쿼리이므로. 참조:
[04-physical-planning/batch-scan-iceberg.md](../04-physical-planning/batch-scan-iceberg.md).

---

## 05 Cost & AQE

### isFinalPlan=false — A, B와 동일

```
AdaptiveSparkPlan isFinalPlan=false
```

`.explain()` 시점 캡처이므로 AQE 런타임 결정 미반영. `.show()` 후 재캡처
필요. 참조: [05-cost-and-aqe/aqe-overview.md](../05-cost-and-aqe/aqe-overview.md).

### EXPLAIN COST — 특이 관찰

```
RelationV2 (4 col):  sizeInBytes=16.2 MiB, rowCount=5.32E+5
Filter (서울):        sizeInBytes=16.2 MiB
Aggregate:            sizeInBytes=21.1 MiB   ← 증가 (!)
Window:               sizeInBytes=24.4 MiB   ← 더 증가
Sort:                 sizeInBytes=24.4 MiB
```

**"Aggregate 후 size가 증가"** — 직관에 반하는 현상. 실제로는:
- Aggregate가 row 수를 줄인다 (530K → 서울 25구 × 연월 조합 ≈ 900 row).
- Catalyst는 **group key cardinality를 모른다** (column histogram 부재).
- 따라서 row 수 축소를 반영 못 하고, 대신 **컬럼 수 증가만 반영** (원본 4
  컬럼 → `sum`, `count` 중간 컬럼 추가로 6 컬럼 → final aggregate output
  5 컬럼).
- 결과: `sizeInBytes`가 증가로 나타남.

**실제 대 추정 오차:**
- 실제 최종 결과: `~900 row × ~80 byte ≈ 70 KB`.
- Catalyst 추정: `24.4 MiB` — **약 360배 과추정**.

**세 쿼리의 cardinality estimator 현상:**
- A: `413 TiB` — join fallback worst-case (과추정 극단).
- B: `30.5 MiB` — single scan, row 수 변화 없어 정확.
- C: `24.4 MiB` — aggregate 축소 미반영 (중간 과추정).

세 경우 모두 **column histogram이 있으면 크게 개선**. 참조:
[05-cost-and-aqe/leveraging-statistics.md](../05-cost-and-aqe/leveraging-statistics.md).

### AQE가 바꿀 여지

- Exchange #1 (3-key hashpartitioning) map output이 작으면 coalesce 공격적
  작동.
- Exchange #2 (1-key hashpartitioning) 이후 row 수가 극히 작아 (~900 row)
  AQE가 partition 수를 대폭 줄일 것 예상.
- Exchange #3 (outer rangepartitioning)도 마찬가지.
- 세 Exchange 모두 AQE coalesce 대상일 가능성 높음 — 실제 작동은 `.show()`
  후 재캡처 필요.

---

## 06 Codegen

### 처리

A/B와 같은 정책 (생성 Java 추측 금지). `debug.codegen()` 출력 없음.

### Fused 가능 구역 추정

```
BatchScan → Filter → [Codegen?]
  → HashAggregate Partial
  → [Exchange #1: 경계]
  → HashAggregate Final
  → [Exchange #2: 경계]
  → Sort [경계]
  → Window
  → [Exchange #3: 경계]
  → Sort [경계]
```

- `BatchScan → Filter → HashAggregate Partial`: fuse 가능성 — Iceberg
  BatchScan의 codegen 지원에 따라.
- `HashAggregate Final`: 단독 block.
- `Sort → Window`: Sort는 codegen 경계. Window는 별도.

### 특별 관심 포인트

**`WindowExec`의 bounded frame (`-5` offset) codegen 지원 여부**:
- Unbounded (running accumulator)는 state 1개라 codegen이 쉬움.
- Bounded sliding window는 deque/ring buffer 관리가 필요해 codegen이 덜
  간단할 수 있음.
- 실제 지원 여부는 `WindowExec`, `WindowFunctionFrame` 소스 + 생성 Java
  관찰로만 확정.

> 📝 `df.queryExecution.debug.codegen()` 캡처 → `raw/codegen/` 저장 → 별도
> trace.

---

## 07 RDD & Stages

### Exchange 3개 → Stage 추정

| Stage | 하는 일 | Boundary |
|---|---|---|
| Stage 0 (shuffle map) | BatchScan → Filter → HashAgg Partial → Exchange #1 write | → Exchange #1 |
| Stage 1 (shuffle map) | Exchange #1 read → HashAgg Final → Exchange #2 write | Exchange #1 → Exchange #2 |
| Stage 2 (shuffle map) | Exchange #2 read → Sort → Window → Exchange #3 write | Exchange #2 → Exchange #3 |
| Stage 3 (result) | Exchange #3 read → Sort → collect | Exchange #3 → driver |

**4 stage**. 세 쿼리 중 가장 긴 stage chain:
- A: 3 stage (2 shuffle + 1 result).
- B: 3 stage (2 shuffle + 1 result).
- C: **4 stage** (3 shuffle + 1 result).

**Java 개발자 관점**: stage = shuffle barrier로 구분된 "같은 이터레이터
pipeline을 실행하는 task 집합". C는 "aggregate key에서 window key로, window
key에서 sort key로" 세 번의 key 전환이 있어 stage chain이 길어짐.

**정확한 stage 개수와 task 수는 Spark UI 권장**. AQE가 partition을 coalesce
하면 task 수는 대폭 감소할 것이지만, stage chain 자체의 길이는 유지됨 (stage는
shuffle 경계로만 결정, task 수와 무관).

---

## 08 Scheduling

이 trace 범위 밖. A/B와 동일.

## 09 Execution

이 trace 범위 밖. 단, 이 쿼리의 `WindowExec` 실행 중 bounded frame을 위한
`WindowFunctionFrame`의 내부 동작 (sum/count incremental update, oldest row
drop, spill 정책)은 별도 관찰 대상.

---

## 이 쿼리가 드러낸 Wiki 갭

### 단계 03 — Logical Optimization

- [관찰됨] CTE inline (`WithCTE` + `CTERelationDef` + `CTERelationRef` 완전
  소거, 단일 참조).
- [후속 페이지 생성됨] `wiki/03-logical-optimization/cte-inline.md` (이
  trace에서 생성, tentative).
- [소스 ingest 필요] `sql/catalyst/.../optimizer/InlineCTE.scala` 또는
  Optimizer.scala, `CTESubstitution.scala`. inline 결정 조건, 다중 참조
  CTE 처리, recursive CTE 처리 확정.
- [후속 trace 필요] 다중 참조 CTE 쿼리 — inline vs `ReusedExchange` /
  `InMemoryRelation` 분기 관찰.

### 단계 04 — Physical Planning — **우선순위 높음**

- **[관찰됨][우선순위 높음] `partitioning-compatibility.md` 페이지 후보**:
  aggregate key와 window partition key 불일치 시 추가 Exchange 삽입.
  `hashpartitioning(a, b, c)`가 `ClusteredDistribution(a)`를 satisfy하지
  **않는** 이유, prefix match의 예외 조건, `HashPartitioning.satisfies`
  로직. 이 개념은 모든 "다중 level 집계/분석 쿼리"에 보편적으로 적용 —
  다음 세션 최우선.
- **[관찰됨][우선순위 높음] `ensure-requirements.md` 페이지 후보**:
  세 쿼리 모두에서 Exchange + Sort 삽입의 주체. 각 물리 노드의
  `requiredChildDistribution`/`requiredChildOrdering`과 자식의
  `outputPartitioning`/`outputOrdering` 비교 후 자동 삽입. 쿼리 A/B/C의
  모든 Exchange가 `ENSURE_REQUIREMENTS` 마커로 이 rule의 책임을 표시. 다음
  세션 최우선 공동 1순위.
- [후속 페이지 후보] `window-frame-spec.md`: `RowFrame` vs `RangeFrame`,
  bounded (`-5`) vs unbounded (`unboundedpreceding$()`) vs following. 쿼리
  B (unbounded preceding ~ current) + C (`-5` ~ current) 두 사례 확보. 한두
  쿼리 더 축적 후 생성.
- [관찰됨] 추가 shuffle을 피할 수 있었을까?: 사용자가 CTE body의 `GROUP
  BY (agent_gu, deal_year, deal_month)` 대신 **먼저 agent_gu로 grouping
  후 nested aggregation**을 쓰거나, `PARTITION BY (agent_gu, deal_year,
  deal_month)`로 Window를 재정의하면 Exchange #2 제거 가능할까? — 쿼리
  형태별 실험 주제.
- [소스 ingest 필요] `sql/core/.../exchange/EnsureRequirements.scala`,
  `Distribution.scala`, `Partitioning.scala`.

### 단계 06 — Codegen

- [후속 trace 필요] `debug.codegen()` 캡처. 특히 bounded frame Window의
  codegen 지원 여부.
- [소스 ingest 필요] `sql/core/.../execution/window/WindowFunctionFrame.scala`
  — bounded vs unbounded frame 구현 분기, incremental update 전략.

### 단계 05 — Cost & AQE

- [별도 trace 필요 — 세 쿼리 공통] `.show()` 후 `isFinalPlan=true`
  재캡처 → AQE 런타임 결정 관찰. C는 최종 row 수가 극히 작아 AQE coalesce
  가 극적으로 작동할 것 예상.
- [관찰 축적] Catalyst cardinality estimator의 세 얼굴 (A: join fallback
  폭발, B: 정확, C: aggregate 축소 미반영) — `leveraging-statistics.md` L5
  강화 재료.

### Cross-cutting

- [관찰됨] Skeleton 생성 추세 4 → 2 → 1 — Wiki 성숙도 지표. 다음 세션의
  trace에서 skeleton 0 → 1 개 수준이 예상되는 지점.
- [후속 페이지 후보] `top-n-optimizations.md`: A의 `TakeOrderedAndProject`
  vs B의 `WindowGroupLimit` vs C의 "top-N 최적화 없음 — 전체 정렬". 세 가지
  대조가 하나의 페이지에 자연스럽게 모임.

---

## 개인 학습 노트

<!-- 이 공간은 Song이 직접 채운다. Claude는 수정 금지. -->
<!--
    이 쿼리를 돌려본 후 깨달은 것:
    - "aggregate key의 prefix라도 window partition key로 재사용 불가"
      라는 사실은 한 번 이해하고 나면 GROUP BY와 PARTITION BY를 나란히
      쓰는 쿼리 설계에 영향을 준다. 재shuffle을 피하려면 두 key 집합을
      일치시키거나 한쪽을 다른쪽의 정확한 subset으로 맞춰야 한다.
    - CTE는 문법적으로는 가독성 도구지만 성능적으로는 대부분 투명 (inline).
      예외는 다중 참조 + deterministic이 불분명한 body — 이 때 Spark가
      materialize 선택. 별도 trace 필요.
    - bounded frame `-5`는 단순한 숫자처럼 보이지만 구현은 deque/ring buffer
      기반이라 unbounded와 다른 codegen 경로를 탈 수 있음. 확인 필요.
    - Statistics가 실제의 360배 과추정 (24.4 MiB vs 70 KB 실제)인 것을 본
      이상, `ANALYZE TABLE ... COMPUTE STATISTICS FOR COLUMNS ...`의 가치를
      체감.
    - ...
-->

---

## Trace 메타

- 생성 명령: `/trace query-C-window-moving-avg`
- 생성된 skeleton: **1개**
  - `03-logical-optimization/cte-inline.md` (tentative, Optimizer ingest 대기)
- 재인용한 기존 skeleton: **6개**
  - `02-analysis/analyzer-overview.md`
  - `03-logical-optimization/predicate-pushdown.md`
  - `04-physical-planning/hash-aggregate-partial-final.md` (주 집계 엔진)
  - `04-physical-planning/batch-scan-iceberg.md`
  - `04-physical-planning/window-exec.md` (AVG over Window도 WindowExec)
  - `04-physical-planning/window-group-limit.md` (**미적용 대조** — 소스 근거로 확정)
- 인용한 verified 소스: `SparkStrategies.scala:657-667` (WindowGroupLimit
  match 조건)
- Skeleton 생성 추세 (A → B → C): 4 → 2 → 1.
- 다음 우선 trace / 후속:
  - **최우선 1**: `ensure-requirements.md` — 세 쿼리 Exchange 삽입의 공통
    주체.
  - **최우선 2**: `partitioning-compatibility.md` — 쿼리 C의 핵심 발견 (추가
    Exchange 불가피성).
  - 세 쿼리 모두 `.show()` 후 executed plan 재캡처 → AQE 런타임 결정 관찰.
  - Optimizer.scala ingest → 2026-04-26 사이클로 완료. `InlineCTE`,
    `InferWindowGroupLimit` (Insert가 아닌 Infer), `ColumnPruning`,
    `LikeSimplification`, `InferFiltersFromConstraints`, `PushDownPredicates`
    모두 verified로 승격됨.
