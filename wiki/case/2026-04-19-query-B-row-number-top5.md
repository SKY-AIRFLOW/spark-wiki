---
stage: case
title: "Query B: 서울 자치구별 거래금액 Top 5 아파트"
spark_versions: ["3.5.3"]
sources:
  - raw/explain/2026-04-19-query-B-row-number-top5.txt
  - raw/spark-source/v3.5.6/sql/core/src/main/scala/org/apache/spark/sql/execution/SparkStrategies.scala
last_verified: 2026-04-19
confidence: verified
related:
  - 00-overview/execution-pipeline.md
  - 02-analysis/analyzer-overview.md
  - 03-logical-optimization/column-pruning.md
  - 03-logical-optimization/infer-filters-from-constraints.md
  - 03-logical-optimization/infer-window-group-limit.md
  - 03-logical-optimization/like-simplification.md
  - 03-logical-optimization/predicate-pushdown.md
  - 04-physical-planning/window-exec.md
  - 04-physical-planning/window-group-limit.md
  - 04-physical-planning/ensure-requirements.md
  - 04-physical-planning/partitioning-compatibility.md
  - 04-physical-planning/hash-aggregate-partial-final.md
  - 04-physical-planning/batch-scan-iceberg.md
  - 05-cost-and-aqe/aqe-overview.md
  - case/2026-04-19-query-A-self-join-growth.md
configs:
  - spark.sql.adaptive.enabled
  - spark.sql.shuffle.partitions
apis:
  - SQL ROW_NUMBER / PARTITION BY / ORDER BY
  - SQL WHERE / ORDER BY
tags: [case, window, row-number, window-group-limit, iceberg, aqe, rangepartitioning]
---

## Query 전문

Plan 트리로부터 재구성한 SQL (실제 제출 SQL과 의미적 동치):

```sql
SELECT agent_gu, apt_name, deal_date, deal_amount, area_sqm, price_rank
FROM (
  SELECT agent_gu, apt_name, deal_date, deal_amount, area_sqm,
         ROW_NUMBER() OVER (PARTITION BY agent_gu
                            ORDER BY deal_amount DESC) AS price_rank
  FROM nessie.real_estate.apt_trades
  WHERE deal_year = 2024
    AND agent_gu LIKE '서울%'
) ranked
WHERE price_rank <= 5
ORDER BY agent_gu ASC, price_rank ASC
```

"서울 자치구별로 2024년 거래금액 상위 5건" 쿼리.

## Version notes

- 캡처 환경: **Spark 3.5.3** (Application `local-1776575126027`,
  2026-04-19 캡처; 쿼리 A와 **같은 세션**).
- 인용 Spark 소스: v3.5.6 `SparkStrategies.scala`. 3.5 마이너 패치 차이
  — 이 쿼리가 의존하는 `Window` strategy (641-655)와 `WindowGroupLimit`
  strategy (657-667)는 3.5 시리즈에서 구조 변경이 알려진 바 없어 동일
  추정.
- **`InferWindowGroupLimit`** (Catalyst rule)는 2026-04-26 ingest 사이클로
  verified 페이지 작성 완료 —
  [03/infer-window-group-limit.md](../03-logical-optimization/infer-window-group-limit.md).
  실제 클래스명은 `InferWindowGroupLimit` (Insert가 아닌 Infer).
- Iceberg: `iceberg-spark-runtime-3.5_2.12:1.7.1` (쿼리 A와 동일).

## 캡처 환경 설정 (쿼리 A와 동일)

| 설정 | 값 |
|---|---|
| `spark.sql.adaptive.enabled` | `true` (단, `isFinalPlan=false` — 아래 05 참고) |
| `spark.sql.shuffle.partitions` | `200` |
| `spark.sql.autoBroadcastJoinThreshold` | `10485760b` (10 MiB) — 이 쿼리에 JOIN 없어 무관 |
| `spark.sql.join.preferSortMergeJoin` | `true` — 이 쿼리에 무관 |

---

## 쿼리 A와의 비교

### 구조적 차이

| 관점 | 쿼리 A (self-join-growth) | 쿼리 B (row-number-top5) |
|---|---|---|
| 데이터 소스 접근 | `apt_trades` self-join (curr 2024 / prev 2023) — 2개 BatchScan | 단일 BatchScan (2024만) |
| 핵심 연산 | SortMergeJoin + HashAggregate | Window (ROW_NUMBER) |
| Top-N 실현 방식 | `TakeOrderedAndProject` (전역 top-20) | `WindowGroupLimit` (그룹별 top-5) |
| Exchange 종류 | `hashpartitioning` × 2 (on `legal_dong`) | `hashpartitioning` × 1 (on `agent_gu`) + `rangepartitioning` × 1 (outer ORDER BY) |
| partial/final 패턴 | `HashAggregate` Partial/Final | `WindowGroupLimit` Partial/Final — 같은 패턴의 재현 |
| Statistics 추정 | Join 결과 `413.3 TiB` (fallback 상한) | 전 체인 `30.5 MiB` — 현실 수치 |
| JoinSelection 경로 | 사용됨 (preferSMJ, threshold 초과) | **사용 안 됨** — JOIN 없음 |

### "fused Top-N 최적화"의 두 경로 — 이 Wiki 학습의 핵심

Spark는 "전체 정렬 후 자르기"를 두 가지 방식으로 피한다. 같은 목적이지만
경로가 완전히 다르다.

**`TakeOrderedAndProject` (쿼리 A)**
- **전역 top-N**. 정렬 순서가 전체 결과에 하나.
- 실현: 각 partition에서 local top-20을 `PriorityQueue`로 유지 → driver
  로 모아서 global merge → final top-20.
- Exchange 없음. reduce-side task가 driver 하나.
- 적용 조건: `LIMIT N` + `ORDER BY ...` (그룹 없는 단일 키).

**`WindowGroupLimit` (쿼리 B)**
- **그룹별 top-N**. 각 `(PARTITION BY key)` 그룹마다 top-N.
- 실현: Partial WindowGroupLimitExec이 각 input partition에서 **그룹별**
  top-N 유지 → shuffle (hashpartitioning on PARTITION BY key) → Final
  WindowGroupLimitExec이 shuffle된 row들에서 그룹별 top-N 재적용 →
  Window가 그 축소된 row들에만 row_number 부여 → Filter가 재확인.
- Exchange 1개 (partial → final 사이).
- 적용 조건: rank-like window function + `Filter(rank ≤ constant)`.

Java 개발자 관점: 두 경로 모두 `PriorityQueue` (또는
`BoundedPriorityQueue`)를 사용하는 **Top-N 구현체**다. 차이는 "몇 개의
큐를 어디에 두느냐":
- A: 각 partition에 큐 하나 × 1 = N개 큐 (N = partition 수).
- B: 각 partition에 `Map<agent_gu, BoundedPQ>` × 2단계 = partition당 수십
  개의 큐.

후자가 더 복잡해 보이지만, 각 큐가 매우 작아 (size 5) cache-friendly이고
그룹별 조기 pruning이 더 공격적이다.

### 공통점

- Analyzer의 `exprId` 부여 방식 동일 (예: `agent_gu#308`, `price_rank#296`).
- Column pruning: 17 컬럼 → 6 컬럼 (이 쿼리는 `apt_name`까지 필요해 A의
  5개보다 하나 많음).
- Predicate pushdown: `deal_year=2024 + agent_gu LIKE '서울%'`의 `LIKE`
  → `StartsWith`, `IS NOT NULL` 추론, BatchScan filters까지 전달.
- Iceberg BatchScan 노드 구조 및 `filters=[...]` 포맷 동일.
- **AQE `isFinalPlan=false`** — 두 쿼리 모두 `.explain()` 시점 캡처라 AQE
  런타임 결정 미반영.

---

## 파이프라인 요약

| 단계 | 이 쿼리에서 일어나는 일 | Wiki 참조 |
|---|---|---|
| 01 Parsing | `'UnresolvedRelation` 1회 + `'Project` (`ROW_NUMBER()` windowspecdefinition) + `'Filter (price_rank <= 5)` + outer `'Sort`. | (skeleton 대기) |
| 02 Analysis | RelationV2 17 컬럼 스키마, 각 컬럼 exprId 부여. ROW_NUMBER()가 `Window` 논리 노드로 해결되고 `unspecifiedframe` → `specifiedwindowframe(RowFrame, unboundedpreceding, currentrow)`로 확정. | [02/analyzer-overview.md](../02-analysis/analyzer-overview.md) |
| 03 Logical Optimization | Column pruning (17→6), predicate pushdown (WHERE → 아래로 + `LIKE → StartsWith` + `IS NOT NULL`), **`InferWindowGroupLimit` 규칙 발동**: `Filter(price_rank ≤ 5) over Window(row_number())` 패턴 인식 → Window 아래에 `logical.WindowGroupLimit(..., 5)` 삽입. `Filter`와 `Window`는 의미 보존을 위해 유지. | [03/predicate-pushdown.md](../03-logical-optimization/predicate-pushdown.md), [03/column-pruning.md](../03-logical-optimization/column-pruning.md), [03/like-simplification.md](../03-logical-optimization/like-simplification.md), [03/infer-filters-from-constraints.md](../03-logical-optimization/infer-filters-from-constraints.md), [03/infer-window-group-limit.md](../03-logical-optimization/infer-window-group-limit.md) |
| 04 Physical Planning | `Window` strategy가 `WindowExec` 생성 (SparkStrategies.scala:641-655). `WindowGroupLimit` strategy가 **단일 논리 노드를 Partial + Final 두 물리 노드로 분할** (SparkStrategies.scala:657-667). `EnsureRequirements`(line 71-80)가 Final 위에 `Exchange hashpartitioning(agent_gu, 200)` + Sort 배치, outer ORDER BY는 `OrderedDistribution.createPartitioning`을 통해 `Exchange rangepartitioning` + Sort로 배치. RangePartitioning ↔ OrderedDistribution prefix 매칭은 [04/partitioning-compatibility.md](../04-physical-planning/partitioning-compatibility.md) 박스 ② (`partitioning.scala:449-467`). | [04/window-exec.md](../04-physical-planning/window-exec.md), [04/window-group-limit.md](../04-physical-planning/window-group-limit.md), [04/ensure-requirements.md](../04-physical-planning/ensure-requirements.md), [04/partitioning-compatibility.md](../04-physical-planning/partitioning-compatibility.md), [04/hash-aggregate-partial-final.md](../04-physical-planning/hash-aggregate-partial-final.md) (교차 패턴), [04/batch-scan-iceberg.md](../04-physical-planning/batch-scan-iceberg.md) |
| 05 Cost & AQE | `AdaptiveSparkPlan isFinalPlan=false`. EXPLAIN COST는 30.5 MiB 전후 — 쿼리 A 같은 fallback 폭발 없음. | [05/aqe-overview.md](../05-cost-and-aqe/aqe-overview.md) |
| 06 Codegen | 추측 금지. fused 가능 구역 추정만. | (별도 trace) |
| 07 RDD & Stages | Exchange 2개 → shuffle map stage 2개 + 중간 처리 stage. 정확한 stage 개수는 Spark UI 권장. | (Exchange 기반 추정) |
| 08 Scheduling | 이 trace 범위 밖. | — |
| 09 Execution | 이 trace 범위 밖. | — |

---

## 01 Parsing

### Plan 트리 (raw 01 PARSED)

```
'Sort ['agent_gu ASC NULLS FIRST, 'price_rank ASC NULLS FIRST], true
+- 'Project ['agent_gu, 'apt_name, 'deal_date, 'deal_amount,
             'area_sqm, 'price_rank]
   +- 'Filter ('price_rank <= 5)
      +- 'SubqueryAlias ranked
         +- 'Project ['agent_gu, 'apt_name, 'deal_date, 'deal_amount,
                      'area_sqm,
                      'ROW_NUMBER()
                        windowspecdefinition('agent_gu,
                                             'deal_amount DESC NULLS LAST,
                                             unspecifiedframe$())
                      AS price_rank#296]
            +- 'Filter (('deal_year = 2024) AND 'agent_gu LIKE 서울%)
               +- 'UnresolvedRelation [nessie, real_estate, apt_trades],
                                      [], false
```

### Java 개발자 관점

- **`'SubqueryAlias ranked`**: subquery의 이름. Java로 치면 inner class
  에 이름을 붙인 것. Analyzer가 해결해서 제거한다.
- **`'ROW_NUMBER() windowspecdefinition(...)`**: `ROW_NUMBER()` 호출과
  `OVER (PARTITION BY ... ORDER BY ...)` 절이 구문 레벨에서 결합된 표현.
  `windowspecdefinition`은 "window 함수의 specification" — partition,
  order, frame의 묶음. `unspecifiedframe$()`는 "프레임을 사용자가 안 썼다"는
  표식. Analyzer가 function 종류에 따라 기본 frame을 채울 것.
- **`price_rank#296`이 parsed에 이미 있음**: 쿼리 A와 동일하게, alias AS
  뒤 표현식에 parser가 "새 출력 ID 필요"를 표시.
- **`'Filter ('price_rank <= 5)`의 위치**: inner subquery 밖, outer
  SELECT의 WHERE. parser는 쿼리 문법 순서대로 쌓는다 — Analyzer/Optimizer가
  이후 적절한 위치로 재배치.

---

## 02 Analysis

### Plan 트리 (raw 02 ANALYZED)

```
Sort [agent_gu#308 ASC NULLS FIRST, price_rank#296 ASC NULLS FIRST], true
+- Project [agent_gu#308, apt_name#297, deal_date#313, deal_amount#304,
            area_sqm#305, price_rank#296]
   +- Filter (price_rank#296 <= 5)
      +- SubqueryAlias ranked
         +- Project [agent_gu#308, apt_name#297, ..., price_rank#296]
            +- Project [agent_gu#308, ..., price_rank#296, price_rank#296]
                                                         # 이중 Project
               +- Window [row_number()
                          windowspecdefinition(agent_gu#308,
                                               deal_amount#304 DESC NULLS LAST,
                                               specifiedwindowframe(RowFrame,
                                                 unboundedpreceding$(),
                                                 currentrow$()))
                          AS price_rank#296],
                         [agent_gu#308],
                         [deal_amount#304 DESC NULLS LAST]
                  +- Project [agent_gu#308, apt_name#297, deal_date#313,
                              deal_amount#304, area_sqm#305]
                     +- Filter ((deal_year#301 = 2024)
                                AND agent_gu#308 LIKE 서울%)
                        +- SubqueryAlias nessie.real_estate.apt_trades
                           +- RelationV2[apt_name#297, ..., deal_date#313]
                                nessie.real_estate.apt_trades ...
```

### Java 개발자 관점

- **`Window [...]` 논리 노드로 해결됨**: Analyzer의 `ResolveWindowOrder`
  류 규칙이 `'Project` 안의 `ROW_NUMBER() windowspecdefinition`을
  뽑아내 별도 `Window` 노드로 승격. Java 관점: 표현식 안에 섞여 있던
  stateful 연산을 전용 연산자로 분리한 것.
- **`unspecifiedframe$()` → `specifiedwindowframe(RowFrame, unboundedpreceding,
  currentrow)`**: ROW_NUMBER()의 기본 frame이 Analyzer에 의해 채워짐.
  ROW_NUMBER 자체는 frame에 영향받지 않지만 plan의 균일성을 위해 명시.
- **이중 Project** (`+- Project [..., price_rank#296] +- Project [..., price_rank#296, price_rank#296]`):
  상위 Project는 outer SELECT의 projection, 하위 Project는 Window가 추가한
  `price_rank` 컬럼을 포함한 상태. Analyzer는 resolution만 하고 중복 제거는
  Optimizer 책임 — 다음 단계에서 `CollapseProject`가 정리.
- **partitionSpec `[agent_gu#308]`과 orderSpec `[deal_amount#304 DESC NULLS LAST]`가 Window 노드의 명시적 필드로 승격**: `windowspecdefinition` 표현식 안에
  있던 값들이 `Window` 노드의 argument로 추출. 이후 Physical Planning에서
  이들이 `ClusteredDistribution` + `Ordering` 요구로 매핑된다.

---

## 03 Logical Optimization

### Plan 트리 (raw 03 OPTIMIZED)

```
Sort [agent_gu#308 ASC NULLS FIRST, price_rank#296 ASC NULLS FIRST], true
+- Filter (price_rank#296 <= 5)
   +- Window [row_number() windowspecdefinition(agent_gu#308,
                                                deal_amount#304 DESC NULLS LAST,
                                                specifiedwindowframe(RowFrame,
                                                  unboundedpreceding$(),
                                                  currentrow$()))
              AS price_rank#296],
             [agent_gu#308],
             [deal_amount#304 DESC NULLS LAST]
      +- WindowGroupLimit [agent_gu#308],
                          [deal_amount#304 DESC NULLS LAST],
                          row_number(), 5                # ← 새로 삽입됨
         +- Project [agent_gu#308, apt_name#297, deal_date#313,
                     deal_amount#304, area_sqm#305]
            +- Filter (((isnotnull(deal_year#301)
                         AND isnotnull(agent_gu#308))
                        AND (deal_year#301 = 2024))
                       AND StartsWith(agent_gu#308, 서울))
               +- RelationV2[apt_name#297, deal_year#301, deal_amount#304,
                             area_sqm#305, agent_gu#308, deal_date#313]
                    nessie.real_estate.apt_trades
```

### Analyzed 대비 변경점

| 변경 | Analyzed | Optimized |
|---|---|---|
| RelationV2 컬럼 수 | 17 | **6** |
| SubqueryAlias | 이중 (ranked + 테이블 경로) | 모두 제거됨 |
| 이중 Project | 있음 | `CollapseProject`로 병합 |
| WHERE filter | `deal_year = 2024 AND agent_gu LIKE '서울%'` | `isnotnull(deal_year) AND isnotnull(agent_gu) AND deal_year=2024 AND StartsWith(agent_gu, 서울)` |
| LIKE | `LIKE 서울%` | `StartsWith(agent_gu, 서울)` |
| **WindowGroupLimit 노드** | 없음 | **새로 삽입됨** (Window 바로 아래) |
| outer `Filter(price_rank <= 5)` | Window 위 | 그대로 유지 (제거 안 됨) |

### Java 개발자 관점 — `WindowGroupLimit` 삽입이 특별한 이유

일반적으로 **논리 최적화**는 plan의 "의미는 같지만 더 가벼운 형태"로
재작성한다. 여기서 벌어진 일은 구조가 다르다:

- 원 plan: `Filter(rank ≤ 5) → Window(row_number) → child`
- 최적화 후: `Filter(rank ≤ 5) → Window(row_number) → WindowGroupLimit(..., 5) → child`

**Filter도, Window도 제거되지 않았다**. `WindowGroupLimit`이 **새로 추가되어
하위에 들어감**. 이는 "필요한 것만 Window로 보내라"는 **조기 축소** 의도의
논리 노드 삽입이다.

Java 개발자 관점에서는 다음과 같다:
- 원래 코드: `stream.map(enumerateRank).filter(r <= 5)` — 전부 enumerate
  후 필터.
- 최적화된 코드: `stream.limitTopNPerGroup(5, byOrder).map(enumerateRank).filter(r <= 5)`
  — 전처리로 각 그룹에서 상위 5개만 남기고, enumerate는 그 축소된 데이터에만,
  filter는 no-op 안전망.

왜 filter와 Window를 지우지 않았는가? 두 가지 이유:
1. **의미 보존**: `WindowGroupLimit`은 물리적 최적화 힌트지, Window의 동작
   을 대체하지 않는다. 실제 `row_number` 값은 여전히 `WindowExec`가
   부여해야 한다.
2. **안전망**: 규칙이 일부 edge case (ties, null handling)에서 정확히
   `top-N`을 고르지 못해도 최종 `Filter(rank ≤ 5)`가 불변식을 보장.

**Catalyst rule이지만 물리적 최적화처럼 느껴지는 이유**: `WindowGroupLimit`
은 논리 노드지만 그 존재 목적이 "물리 실행 시 적은 row만 통과시킨다"는
물리적 효과다. 이건 Catalyst의 "physical-inspired logical optimization"
패턴 — 규칙이 순수 의미 재작성을 넘어, 실행 엔진이 쓸 힌트를 논리 plan에
심어두는 방식.

### 적용된 최적화 규칙 (관찰 기반)

| 규칙 (추정 이름) | 관찰 증거 |
|---|---|
| `ColumnPruning` | RelationV2 17 컬럼 → 6 컬럼 |
| `CollapseProject` | 이중 Project 병합 + SubqueryAlias 제거 |
| `PushDownPredicates` | WHERE가 Window 아래 Project 아래로 |
| `InferFiltersFromConstraints` | `isnotnull(deal_year)`, `isnotnull(agent_gu)` 자동 추가 |
| `LikeSimplification` | `LIKE '서울%'` → `StartsWith(agent_gu, 서울)` |
| **`InferWindowGroupLimit`** | `logical.WindowGroupLimit` 노드가 Window 아래 삽입됨. 정의: `optimizer/InferWindowGroupLimit.scala:45`, batch wiring: `SparkOptimizer.scala:96-101`. 자세한 메커니즘 + 두 갈래 출력 분기는 [03/infer-window-group-limit.md](../03-logical-optimization/infer-window-group-limit.md). |

### 관찰 요점

- **Optimizer가 추가한 물리 성격의 논리 노드** (`WindowGroupLimit`)는 드문
  사례다. 대부분의 Catalyst 규칙은 기존 노드를 재배치·병합·치환하지만, 이
  규칙은 새 노드를 **삽입**한다.
- `Filter(price_rank ≤ 5)`가 plan에 계속 존재함 — 이 Filter는 실행 시
  거의 no-op이지만, 플래너가 제거를 보장하는 규칙을 가지지 않은 한 남는다.

---

## 04 Physical Planning

### Plan 트리 (raw 04 PHYSICAL — Pre-AQE)

```
Sort [agent_gu#308 ASC NULLS FIRST, price_rank#296 ASC NULLS FIRST], true, 0
+- Filter (price_rank#296 <= 5)
   +- Window [row_number() ... AS price_rank#296],
             [agent_gu#308], [deal_amount#304 DESC NULLS LAST]
      +- WindowGroupLimit [agent_gu#308],
                          [deal_amount#304 DESC NULLS LAST],
                          row_number(), 5, Final                   # 물리 Final
         +- WindowGroupLimit [agent_gu#308],
                             [deal_amount#304 DESC NULLS LAST],
                             row_number(), 5, Partial              # 물리 Partial
            +- Project [agent_gu#308, apt_name#297, deal_date#313,
                        deal_amount#304, area_sqm#305]
               +- Filter (... deal_year#301 = 2024 ...)
                  +- BatchScan nessie.real_estate.apt_trades
                        [apt_name#297, deal_year#301, deal_amount#304,
                         area_sqm#305, agent_gu#308, deal_date#313]
                        (branch=null)
                        [filters=deal_year IS NOT NULL,
                                 agent_gu IS NOT NULL,
                                 deal_year = 2024,
                                 agent_gu LIKE '서울%',
                                 groupedBy=]
                        RuntimeFilters: []
```

### Executed (`isFinalPlan=false` 래핑 후)

`EnsureRequirements`가 Partial → Final 사이에 Exchange + Sort를 추가하고,
outer ORDER BY에 대해서도 Exchange + Sort를 추가한다:

```
AdaptiveSparkPlan isFinalPlan=false
+- Sort [agent_gu ASC NULLS FIRST, price_rank ASC NULLS FIRST], true, 0
   +- Exchange rangepartitioning(agent_gu ASC NULLS FIRST,
                                 price_rank ASC NULLS FIRST, 200),
               ENSURE_REQUIREMENTS, [plan_id=374]             # outer ORDER BY용
      +- Filter (price_rank <= 5)
         +- Window [row_number() ... AS price_rank],
                   [agent_gu], [deal_amount DESC NULLS LAST]
            +- WindowGroupLimit ..., Final
               +- Sort [agent_gu ASC NULLS FIRST,
                        deal_amount DESC NULLS LAST], false, 0
                  +- Exchange hashpartitioning(agent_gu, 200),
                              ENSURE_REQUIREMENTS, [plan_id=368]
                                                                # WGL Partial→Final
                     +- WindowGroupLimit ..., Partial
                        +- Sort [agent_gu ASC NULLS FIRST,
                                 deal_amount DESC NULLS LAST], false, 0
                           +- Project ...
                              +- Filter ...
                                 +- BatchScan ...
```

### 네 가지 물리 변환의 Java 개발자 해설

**(1) `Window` logical → `WindowExec` physical** — SparkStrategies.scala:641-655

```scala
object Window extends Strategy {
  def apply(plan: LogicalPlan): Seq[SparkPlan] = plan match {
    case PhysicalWindow(WindowFunctionType.SQL,
                        windowExprs, partitionSpec, orderSpec, child) =>
      execution.window.WindowExec(
        windowExprs, partitionSpec, orderSpec, planLater(child)) :: Nil
    ...
  }
}
```

Java 관점: 논리 `Window` 노드에 담긴 4개 필드를 그대로 물리 `WindowExec`
생성자에 넘긴다. 이 strategy 자체가 하는 일은 "포장 교체". Python UDF면
`WindowInPandasExec`로 가는 두 번째 case도 존재. 참조:
[04-physical-planning/window-exec.md](../04-physical-planning/window-exec.md).

**(2) `WindowGroupLimit` logical → Partial + Final physical** — SparkStrategies.scala:657-667

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
    ...
  }
}
```

**이것이 이 trace의 핵심 발견 중 하나**: 하나의 논리 노드가 **strategy
레벨에서 두 개의 물리 노드로 분할**된다. Java 관점: factory method 안에서
두 개의 객체를 생성하고 서로 wiring해서 최종으로 하나만 반환하는 패턴.

- `partialWindowGroupLimit`이 `planLater(child)`를 감싸고
- `finalWindowGroupLimit`이 `partialWindowGroupLimit`을 감싼다
- 반환은 `finalWindowGroupLimit :: Nil` — 바깥 노드 하나만 반환되지만
  내부적으로 두 노드가 연결.

이 분할은 `HashAggregate`의 partial/final 분할과 **같은 철학**이다. 참조:
[04-physical-planning/hash-aggregate-partial-final.md](../04-physical-planning/hash-aggregate-partial-final.md).
두 경우 모두 map-side 축소 + shuffle + reduce-side 최종. 차이는:
- HashAggregate: 값을 누적 (sum, count) — output row 수 ≤ input row 수.
- WindowGroupLimit: row 자체를 top-N만 유지 — output row 수 ≤ `partition 수 × N`.

Java 개발자가 기억할 공통점: **"map 단계에서 데이터를 줄일 수 있으면
shuffle 전에 줄인다"** — distributed computation의 기본 공리.

**(3) Partial → Final 사이 Exchange + Sort 삽입** — `EnsureRequirements` rule

Pre-AQE plan에는 Exchange가 보이지 않지만 executed plan에는 있다 (`plan_id=368`).
`WindowGroupLimitExec(mode=Final)`이 요구하는 distribution
(`ClusteredDistribution(partitionSpec)`)과 ordering
(`partitionSpec ASC :: orderSpec`)을 만족시키려고 `EnsureRequirements` rule
(단계 04의 또 다른 phase)이 자동 삽입한다. 정확한 진입은
[04/ensure-requirements.md](../04-physical-planning/ensure-requirements.md)의
**단일 자식 distribution check** 분기 (`EnsureRequirements.scala:71-80`) +
ordering check 분기 (`:207-215`). Partial 자식의 `outputPartitioning`은
`UnknownPartitioning`이라
[04/partitioning-compatibility.md](../04-physical-planning/partitioning-compatibility.md)
매트릭스의 default trait fallback (`partitioning.scala:241-245`)에서 false
반환 → `distribution.createPartitioning(200)`으로 새 `HashPartitioning(agent_gu, 200)`
생성 + `ShuffleExchange`로 감싸기.

Java 관점: `List` 이터레이션 전에 `list.sort()`를 반드시 수행하는 것과 같은
**invariant 강제**. 소비자가 전제하는 조건을 만족시키기 위해 삽입되는
boilerplate를 프레임워크가 자동화한 것.

**(4) Outer Exchange rangepartitioning** — 이 쿼리에서 **처음 등장**하는 partitioning

```
Exchange rangepartitioning(agent_gu#308 ASC NULLS FIRST,
                           price_rank#296 ASC NULLS FIRST, 200),
         ENSURE_REQUIREMENTS, [plan_id=374]
```

outer `ORDER BY agent_gu ASC, price_rank ASC` 때문에 삽입됨. **Global sort를
분산으로 실현하려면**:
- 단순 hashpartitioning은 부적합 — 같은 해시 bucket에 들어온 row들은 인접하지만
  bucket들 사이의 순서가 보장되지 않음.
- **rangepartitioning**: 각 partition이 정렬 키의 연속 구간을 담당하게 보장 —
  partition 0: `[agent_gu ≤ k0]`, partition 1: `[k0 < agent_gu ≤ k1]`, ... .
- 각 partition 내부를 sort하면 모든 partition을 concat해 읽는 것만으로
  전체 정렬.

`Sort(global=true)`가 요구하는 `OrderedDistribution(agent_gu ASC, price_rank ASC)`
은 자식의 `outputPartitioning`이 만족하는 partitioning을 가지고 있지 않으므로
[04/ensure-requirements.md](../04-physical-planning/ensure-requirements.md)의
단일 자식 분기 (`:71-80`)에서 false → `OrderedDistribution.createPartitioning`
(`partitioning.scala:181-183`)이 `RangePartitioning(ordering, 200)`을 새로
생성해 `ShuffleExchange`로 감싼다. 만약 자식이 이미 `RangePartitioning(agent_gu)`
이었다면
[04/partitioning-compatibility.md](../04-physical-planning/partitioning-compatibility.md)
박스 ②의 prefix 매칭 (`partitioning.scala:449-467`)에 의해 Exchange 없이 통과
가능했을 것이다 — 본 case는 그 호환성이 사용되는 사례가 아니라 신규 생성
사례.

Java 관점: `TreeMap<agent_gu, List<Row>>`를 200개 bucket으로 쪼개되 **key
범위를 연속적으로 분할**한 것. Spark는 샘플링을 통해 quantile을 추정하고
boundary를 정한다 (`RangePartitioner`).

rangepartitioning은 쿼리 A에 없었다 — A의 `TakeOrderedAndProject`는 local
top-20 + global merge 패턴이라 global sort가 아니기 때문. **쿼리 B는 모든
매칭 row에 대해 완전 정렬을 요구**하므로 rangepartitioning이 필요.

### 물리 plan의 최종 형태 — row 흐름

```
BatchScan (530K rows, ~30 MiB)
  → Filter (2024 + 서울*)      [약 row 축소, 추정 수 MB]
  → Project (6 col)
  → Sort [agent_gu ASC, deal_amount DESC] (local)
  → WindowGroupLimit Partial (각 partition의 agent_gu별 top-5만 유지)
  → Exchange hashpartitioning(agent_gu, 200)  [shuffle barrier]
  → Sort [agent_gu ASC, deal_amount DESC] (local, shuffle 후)
  → WindowGroupLimit Final (agent_gu별 global top-5)
  → Window (row_number 부여)
  → Filter (price_rank <= 5)   [no-op, 안전망]
  → Exchange rangepartitioning(agent_gu ASC, price_rank ASC, 200)  [shuffle barrier]
  → Sort [agent_gu ASC, price_rank ASC] (global sort)
  → [collect / show]
```

서울은 25개 자치구. agent_gu cardinality가 ~25이므로 최종 결과는 `25 × 5 =
125 row`를 넘을 수 없다. 30 MiB 입력이 partial 단계에서 부분당 수백 row
으로 줄고, final 단계에서 125 row 이하로 수렴. **매우 효과적인 축소**.

### BatchScan filters pushdown

```
[filters=deal_year IS NOT NULL, agent_gu IS NOT NULL,
         deal_year = 2024, agent_gu LIKE '서울%',
         groupedBy=]
```

쿼리 A와 같은 4개 filter (A에는 `legal_dong IS NOT NULL`이 추가로 있었으나
이 쿼리는 join 없어 해당 없음). 참조:
[04-physical-planning/batch-scan-iceberg.md](../04-physical-planning/batch-scan-iceberg.md).

---

## 05 Cost & AQE

### isFinalPlan=false — 쿼리 A와 동일한 제약

```
AdaptiveSparkPlan isFinalPlan=false
```

쿼리 A와 같은 이유로, 이 캡처는 AQE 런타임 결정을 포함하지 않는다. `.show()`
실행 후 재캡처 필요. 참조:
[05-cost-and-aqe/aqe-overview.md](../05-cost-and-aqe/aqe-overview.md).

### EXPLAIN COST — 현실적 수치

```
RelationV2 (pruned to 6 cols):  sizeInBytes=30.5 MiB, rowCount=5.32E+5
Filter:                          sizeInBytes=30.5 MiB
Project:                         sizeInBytes=28.7 MiB
WindowGroupLimit:                sizeInBytes=28.7 MiB
Window:                          sizeInBytes=30.5 MiB  ← price_rank 컬럼 추가
Filter (≤ 5):                    sizeInBytes=30.5 MiB
Sort:                            sizeInBytes=30.5 MiB
```

쿼리 A의 413 TiB fallback 폭발과 극명한 대조. **JOIN이 없으면 Catalyst의
cardinality estimator는 잘 작동한다**. Join의 fallback `leftRows × rightRows`
상한 추정은 column-level histogram 없이는 피하기 힘든 왜곡이지만, 단일
table scan + window는 row count가 그대로 유지되거나 단조 감소해 추정이
정확해진다.

주목할 점:
- `WindowGroupLimit`의 `28.7 MiB`는 Catalyst가 "이 노드가 데이터를 줄인다"는
  사실을 일부 반영한 값. 그러나 실제 축소 (530K row → 125 row)에는 훨씬
  못 미치는 과추정. Catalyst가 partitioning key cardinality (`agent_gu ≈ 25`)
  를 몰라서 그렇다 — column histogram이 있으면 더 정확해질 여지.
- `Window` 이후 `30.5 MiB`는 `price_rank` 컬럼 추가로 증가한 byte size.
- 쿼리 결과 자체는 최대 `25 × 5 × row_size ≈ 10 KB` 수준이지만 plan은
  `Sort`에서 `30.5 MiB`로 마감 — **쿼리 끝에서 `WindowGroupLimit`의 실효
  축소가 통계에 반영되지 않음**. 역시 col stats 부재의 현상.

참조: [05-cost-and-aqe/leveraging-statistics.md](../05-cost-and-aqe/leveraging-statistics.md).

### AQE가 바꿀 여지

- Partial/Final 사이 Exchange (plan_id=368)의 map output이 매우 작을 가능성
  — AQE coalesce가 200 partition을 소수로 축소 가능.
- Outer rangepartitioning (plan_id=374)도 마찬가지. 최종 row가 125개 이하
  이므로 AQE coalesce가 매우 공격적으로 작동할 것.
- 이 쿼리에는 JOIN이 없어 SMJ→BHJ/SHJ 전환은 해당 없음.

확정에는 `.show()` 후 `isFinalPlan=true` plan 재캡처 필요.

---

## 06 Codegen

### 처리

쿼리 A와 동일한 정책 (생성 Java 추측 금지). Plan 파일에 `debug.codegen()`
출력 없음.

### Fused 가능 구역 추정 (plan 구조 기반)

```
BatchScan → Filter → Project → Sort [경계: Sort는 codegen 경계]
  → WindowGroupLimit Partial
  → [Exchange: 경계]
  → Sort [경계]
  → WindowGroupLimit Final
  → Window
  → Filter
  → [Exchange rangepartitioning: 경계]
  → Sort [경계]
```

- `BatchScan → Filter → Project`: Iceberg BatchScan이 codegen을 지원하면 fuse
  가능.
- Sort는 대개 codegen 경계.
- `WindowGroupLimit Partial`, `Final`: codegen 지원 여부는 소스 확인 필요
  — 이 trace에서는 미확정.
- `Window`와 `Filter`는 fuse 가능성 있음.

> 📝 `df.queryExecution.debug.codegen()` 캡처 후 `raw/codegen/` 저장으로
> 확정 (별도 trace).

---

## 07 RDD & Stages

### Exchange 2개 기반 stage 경계 추정

| Stage 추정 | 하는 일 | Exchange/Task 수 (대략) |
|---|---|---|
| Stage 0 (shuffle map) | BatchScan → Filter → Project → Sort → WGL Partial → Exchange write (`plan_id=368`) | Iceberg scan split 수 |
| Stage 1 (shuffle map) | Exchange read → Sort → WGL Final → Window → Filter → Exchange write (`plan_id=374`) | 200 (AQE coalesce 전) |
| Stage 2 (result) | Exchange read → Sort → collect | 200 (AQE coalesce 전) |

쿼리 A는 SMJ 때문에 양쪽 scan이 각각 shuffle map stage가 되고 SMJ 수행
stage 하나로 3 stage 구조였다. 쿼리 B는 단일 scan 경로이지만 **Exchange
2개로 stage 3개** — SMJ 대신 "outer ORDER BY용 추가 shuffle"이 stage를
하나 더 만든다.

### Java 개발자 관점

- 한 쿼리의 stage 수는 "Exchange 개수 + 1"의 근사. A는 2+1, B도 2+1 = 3.
- 하지만 A와 B의 stage 역할은 완전히 다르다: A는 "양쪽 scan + join", B는
  "scan + window 처리 + 외부 정렬". 같은 수라도 연산 성격이 다르면 wall clock
  시간의 bottleneck 위치도 다르다.

**정확한 stage 개수와 task 수는 Spark UI 권장**.

---

## 08 Scheduling

이 trace 범위 밖. 쿼리 A와 동일.

## 09 Execution

이 trace 범위 밖. 쿼리 A와 동일.

---

## 이 쿼리가 드러낸 Wiki 갭

### 단계 03 — Logical Optimization

- [페이지 작성 완료] **`InferWindowGroupLimit` 규칙** —
  [03/infer-window-group-limit.md](../03-logical-optimization/infer-window-group-limit.md)
  (verified, 2026-04-26 사이클). 실제 클래스명은 Insert 아닌 Infer.
  정의: `optimizer/InferWindowGroupLimit.scala`, batch wiring:
  `SparkOptimizer.scala:96-101` (Catalyst defaultBatches 부재 — SparkOptimizer
  전용).
- [후속 페이지 후보] `wiki/03-logical-optimization/physical-inspired-optimizations.md`
  — "물리 성격의 논리 최적화" 패턴을 모은 cross-cutting 페이지 (현재
  infer-window-group-limit 단발 사례. 후속 trace에서 동일 패턴 누적 후
  생성).

### 단계 04 — Physical Planning

- [관찰됨] `Exchange rangepartitioning` — outer ORDER BY global sort용 첫
  사례. `hashpartitioning`과 구조·목적 차이.
- [후속 페이지 후보] `wiki/04-physical-planning/range-partitioning.md` — 여러
  ORDER BY 쿼리 관찰 축적 후 (예: C의 window ORDER BY도 함께) 생성.
- [소스 ingest 필요] `sql/core/.../exchange/ShuffleExchangeExec.scala`,
  `RangePartitioning`, `RangePartitioner` (sampling 기반 boundary 계산).
- [관찰됨] `EnsureRequirements` rule이 Exchange + Sort를 자동 삽입하는 현상 —
  "distribution + ordering invariant 강제" 메커니즘.
- [후속 페이지 후보] `wiki/04-physical-planning/ensure-requirements.md`.
- [관찰됨] `WindowExec` 내부 — 이 trace는 변환까지만 추적, 실제 row 순회
  로직은 미확정.
- [소스 ingest 필요] `sql/core/.../execution/window/WindowExec.scala`,
  `WindowGroupLimitExec.scala`, `WindowFunctionFrame.scala`.

### 단계 05 — Cost & AQE

- [관찰됨] 쿼리 A와 B 모두 `isFinalPlan=false` — 공통 제약.
- [별도 trace 필요] 세 쿼리 (A, B, C) 모두 `.show()` 후 재캡처해 `isFinalPlan=true`
  plan을 `raw/explain/2026-04-19-*-executed.txt`로 저장. 특히 B는 cardinality
  가 매우 작아 AQE partition coalesce가 극적으로 작동할 것 — 좋은 관찰 대상.

### 단계 06

- [후속 trace 필요] 쿼리 A와 동일. `debug.codegen()` 캡처.
- [특별 관찰 포인트] `WindowGroupLimitExec`이 codegen을 지원하는지, Partial과
  Final이 같은 WholeStageCodegen 블록에 fuse되지 않는지 (Exchange 경계로 인해
  fuse 불가 예상).

### 단계 07

- [후속 페이지 후보] `wiki/07-rdd-and-stages/exchange-and-stage-boundary.md`
  (쿼리 A 갭에서 이미 언급됨) — B가 rangepartitioning 케이스까지 추가하므로
  재료가 더 풍부.

### Cross-cutting — "Top-N 최적화"의 여러 경로

- [후속 페이지 후보] `wiki/04-physical-planning/top-n-optimizations.md` —
  `TakeOrderedAndProject` (A) vs `WindowGroupLimit` (B)의 공통점·차이점을
  한 페이지에 정리. 향후 `Limit` + `Offset`, `APPROX_PERCENTILE` 등 다른
  top-N 전략도 함께.

---

## 개인 학습 노트

<!-- 이 공간은 Song이 직접 채운다. Claude는 수정 금지. -->
<!--
    이 쿼리를 돌려본 후 깨달은 것:
    - WindowGroupLimit이 HashAggregate와 같은 Partial/Final 패턴이라는 걸
      깨달은 순간 Spark의 "distributed reduction" 골격이 선명해졌다.
    - rangepartitioning은 global sort를 위한 것. hashpartitioning과의 차이가
      "bucket 내부 순서"에 있다는 걸 이해하니 ORDER BY 쿼리가 왜 추가 shuffle
      을 요구하는지 자명해짐.
    - Filter(price_rank <= 5)가 Window 뒤에도 남는 이유: 물리 최적화가 의미
      레이어를 건드리지 않는다는 원칙. 안전망이자 semantic contract.
    - ...
-->

---

## Trace 메타

- 생성 명령: `/trace query-B-row-number-top5`
- 생성된 skeleton: 2개
  - `04-physical-planning/window-exec.md` (L3에 SparkStrategies.scala:641-655 인용, 나머지 tentative)
  - `04-physical-planning/window-group-limit.md` (L3에 SparkStrategies.scala:657-667 인용, 짝이 되는 Catalyst rule은 2026-04-26 사이클에서 [03/infer-window-group-limit.md](../03-logical-optimization/infer-window-group-limit.md) verified로 작성됨)
- 재인용한 기존 skeleton: 4개 (analyzer-overview, predicate-pushdown, hash-aggregate-partial-final [교차 패턴], batch-scan-iceberg)
- 인용한 verified 페이지: `SparkStrategies.scala` 직접 인용 2회 (Window + WindowGroupLimit strategies)
- 다음 우선 trace: 쿼리 C (`/trace query-C-window-moving-avg`) — WindowGroupLimit이 **아닌** Window 케이스 (aggregate-over-window + CTE). 또는 동일 쿼리 A/B/C의 `.show()` 후 AQE executed plan 재캡처.
