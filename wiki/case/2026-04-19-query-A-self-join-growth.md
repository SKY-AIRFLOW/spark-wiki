---
stage: case
title: "Query A: 서울 자치구·법정동별 2023→2024 거래 증감 Top 20"
spark_versions: ["3.5.3"]
sources:
  - raw/explain/2026-04-19-query-A-self-join-growth.txt
  - raw/spark-source/v3.5.6/sql/core/src/main/scala/org/apache/spark/sql/execution/SparkStrategies.scala
  - raw/spark-source/v3.5.6/sql/catalyst/src/main/scala/org/apache/spark/sql/catalyst/optimizer/joins.scala
last_verified: 2026-04-19
confidence: verified
related:
  - 00-overview/execution-pipeline.md
  - 02-analysis/analyzer-overview.md
  - 03-logical-optimization/predicate-pushdown.md
  - 04-physical-planning/join-strategy-hints.md
  - 04-physical-planning/ensure-requirements.md
  - 04-physical-planning/partitioning-compatibility.md
  - 04-physical-planning/hash-aggregate-partial-final.md
  - 04-physical-planning/batch-scan-iceberg.md
  - 05-cost-and-aqe/aqe-overview.md
configs:
  - spark.sql.adaptive.enabled
  - spark.sql.shuffle.partitions
  - spark.sql.autoBroadcastJoinThreshold
  - spark.sql.join.preferSortMergeJoin
apis:
  - DataFrame.groupBy
  - DataFrame.filter
  - SQL SELECT / JOIN / GROUP BY / HAVING / ORDER BY / LIMIT
tags: [case, self-join, aggregate, iceberg, sort-merge-join, aqe]
---

## Query 전문

Plan 트리로부터 재구성한 SQL (실제 제출 SQL과 의미적 동치):

```sql
SELECT curr.agent_gu,
       curr.legal_dong,
       COUNT(curr.apt_seq)                              AS trades_2024,
       ROUND(AVG(curr.deal_amount), 0)                  AS avg_2024,
       ROUND(AVG(prev.deal_amount), 0)                  AS avg_2023,
       ROUND(((AVG(curr.deal_amount) - AVG(prev.deal_amount))
              / AVG(prev.deal_amount)) * 100, 2)        AS growth_pct
FROM nessie.real_estate.apt_trades curr
JOIN nessie.real_estate.apt_trades prev
  ON curr.legal_dong = prev.legal_dong
WHERE curr.deal_year  = 2024
  AND prev.deal_year  = 2023
  AND curr.agent_gu LIKE '서울%'
GROUP BY curr.agent_gu, curr.legal_dong
HAVING COUNT(curr.apt_seq) >= 5
   AND AVG(prev.deal_amount) > 0
ORDER BY growth_pct DESC
LIMIT 20
```

## Version notes

- 캡처 환경: **Spark 3.5.3** (Application `local-1776575126027`,
  2026-04-19 캡처).
- 인용할 Spark 소스는 어제 ingest한 **v3.5.6**. 같은 3.5 마이너의
  패치 차이 — `JoinSelection`, `Aggregation`, `Predicate` 최적화의
  로직 구조는 동일 추정. 재작성이 발생한 규칙이 있으면 후속 ingest
  때 명시.
- 외부 플러그인: **iceberg-spark-runtime-3.5_2.12:1.7.1**. 테이블
  카탈로그는 Nessie. 세 쿼리 모두 `nessie.real_estate.apt_trades`
  읽음.

## 캡처 환경 설정 (Plan 파일 헤더에서)

| 설정 | 값 | 이 쿼리에 미치는 영향 |
|---|---|---|
| `spark.sql.adaptive.enabled` | `true` | AQE wrapper 노드 등장. 단 이 캡처는 `isFinalPlan=false` — 아래 05 단계 참고. |
| `spark.sql.shuffle.partitions` | `200` | Exchange의 `hashpartitioning(legal_dong, 200)`에 반영. |
| `spark.sql.autoBroadcastJoinThreshold` | `10485760b` (10 MiB) | 양쪽 relation 크기 초과 → SMJ 선택 (아래 04 참고). |
| `spark.sql.join.preferSortMergeJoin` | `true` | SHJ 후보 조건 미달이어도 SMJ로 안착. |

## 파이프라인 요약

| 단계 | 이 쿼리에서 일어나는 일 | Wiki 참조 |
|---|---|---|
| 01 Parsing | SQL 텍스트 → `'UnresolvedRelation [nessie, real_estate, apt_trades]` 2회(curr/prev) + `'Aggregate` + `'UnresolvedHaving` + `'Sort` + `'Limit` 트리. | (skeleton 대기) |
| 02 Analysis | 양쪽 `UnresolvedRelation` → 같은 `RelationV2` (17 컬럼 스키마). 각 컬럼에 별도 `exprId` 부여 (`legal_dong#196` vs `#213`). `UnresolvedHaving` → `Filter` 위치 확정. | [02-analysis/analyzer-overview.md](../02-analysis/analyzer-overview.md) |
| 03 Logical Optimization | Column pruning (17→5/3), predicate pushdown (WHERE를 Join 양쪽 child로 분배 + `IS NOT NULL` 추론), `LIKE '서울%'` → `StartsWith`, `cast` constant folding. | [03-logical-optimization/predicate-pushdown.md](../03-logical-optimization/predicate-pushdown.md) |
| 04 Physical Planning | **SortMergeJoin** (두 side 모두 broadcastable 미만 아님, preferSMJ=true, equi-join). **HashAggregate partial → final 2단계**. **TakeOrderedAndProject** (LIMIT+ORDER BY fused). **BatchScan** 2회 (Iceberg V2). SMJ 양쪽 `Exchange hashpartitioning(legal_dong, 200)`은 `EnsureRequirements`의 **multi-child co-partitioning 경로**(line 99-205) — 양쪽 자식 모두 partitioning을 자체 생성 못 해 양쪽 모두 ShuffleExchange 삽입. | [04/join-strategy-hints.md](../04-physical-planning/join-strategy-hints.md), [04/ensure-requirements.md](../04-physical-planning/ensure-requirements.md), [04/partitioning-compatibility.md](../04-physical-planning/partitioning-compatibility.md), [04/hash-aggregate-partial-final.md](../04-physical-planning/hash-aggregate-partial-final.md), [04/batch-scan-iceberg.md](../04-physical-planning/batch-scan-iceberg.md) |
| 05 Cost & AQE | `AdaptiveSparkPlan isFinalPlan=false` 래퍼만 존재 — 이 캡처는 `.explain()` 시점 plan이라 **AQE 런타임 결정이 아직 적용 전**. EXPLAIN COST는 상세 통계 제공 (아래 05 참고). | [05-cost-and-aqe/aqe-overview.md](../05-cost-and-aqe/aqe-overview.md) |
| 06 Codegen | Plan 파일에는 codegen 출력 미포함. fused 가능 구역 추정만 기록. 생성 Java 관찰은 후속 `debug.codegen()` trace로 분리. | (별도 trace) |
| 07 RDD & Stages | Exchange 2개 → shuffle map stage 2개, SMJ+partial HashAggregate 실행 stage, final HashAggregate + TakeOrderedAndProject 마무리 stage. 정확한 stage 개수는 Spark UI 권장. | (Exchange 기반 추정) |
| 08 Scheduling | 이 trace 범위 밖. | — |
| 09 Execution | 이 trace 범위 밖. | — |

---

## 01 Parsing

### Plan 트리 (raw 01 PARSED 섹션 발췌)

```
'GlobalLimit 20
+- 'LocalLimit 20
   +- 'Sort ['growth_pct DESC NULLS LAST], true
      +- 'UnresolvedHaving (('COUNT('curr.apt_seq) >= 5)
                           AND ('AVG('prev.deal_amount) > 0))
         +- 'Aggregate ['curr.agent_gu, 'curr.legal_dong],
              [... 'COUNT('curr.apt_seq) AS trades_2024#182,
                   'ROUND('AVG('curr.deal_amount), 0) AS avg_2024#183,
                   'ROUND('AVG('prev.deal_amount), 0) AS avg_2023#184,
                   'ROUND(((('AVG('curr.deal_amount)
                            - 'AVG('prev.deal_amount))
                           / 'AVG('prev.deal_amount))
                          * 100), 2) AS growth_pct#185]
            +- 'Filter ((('curr.deal_year = 2024)
                        AND ('prev.deal_year = 2023))
                        AND 'curr.agent_gu LIKE 서울%)
               +- 'Join Inner, ('curr.legal_dong = 'prev.legal_dong)
                  :- 'SubqueryAlias curr
                  :  +- 'UnresolvedRelation [nessie, real_estate, apt_trades], [], false
                  +- 'SubqueryAlias prev
                     +- 'UnresolvedRelation [nessie, real_estate, apt_trades], [], false
```

### Java 개발자 관점

- **`'` prefix는 "unresolved"의 표식**. Scala 쪽 `Expression` DSL
  관습으로, 아직 카탈로그와 대조되지 않았다는 의미.
- `'UnresolvedRelation [nessie, real_estate, apt_trades]`은 Java의
  `import nessie.real_estate.apt_trades` 문장이 아직 컴파일러에
  의해 해결되기 전 상태에 가깝다. "이런 이름의 심볼이 있다"는
  선언만 있고, 어떤 클래스인지·어떤 필드를 가졌는지는 모름.
- `'SubqueryAlias curr` / `'SubqueryAlias prev`가 **같은
  UnresolvedRelation을 두 번 감싼다**. Java로 치면 같은 테이블
  클래스를 두 개의 변수로 참조한 것. 별칭만으로는 아직 컬럼들이
  구분되지 않음 — 이것이 다음 Analysis 단계에서 `exprId`로 분리된다.
- aggregate 결과 컬럼에만 이미 `#182 ~ #185`가 있는 이유: parser가
  alias AS 뒤 표현식에 대해 "이 출력에 새 ID 필요"를 표시한 것.
  입력 측 컬럼 (deal_amount, legal_dong 등)은 아직 ID 없음.
- `'UnresolvedHaving`이 `'Aggregate` 위에 있는 것은 구문 순서를
  반영. HAVING이 aggregate result를 참조해야 함을 Analyzer가 알아야
  하므로, 구문 그대로 위에 쌓아두고 해결은 뒤로 미룸.

### 관찰 요점

- 이 단계의 산출물은 **심볼 테이블 조회 전** 구문 트리다. 어떤
  컬럼이 있는지, 타입이 무엇인지 모른다 — 그러므로 Optimizer 규칙을
  여기에 적용할 수 없다. Analysis가 반드시 먼저다.

---

## 02 Analysis

### Plan 트리 (raw 02 ANALYZED 섹션 발췌)

```
GlobalLimit 20
+- LocalLimit 20
   +- Sort [growth_pct#185 DESC NULLS LAST], true
      +- Project [agent_gu#197, legal_dong#196, trades_2024#182L,
                  avg_2024#183, avg_2023#184, growth_pct#185]
         +- Filter ((trades_2024#182L >= cast(5 as bigint))
                    AND (avg(deal_amount#210)#238 > cast(0 as double)))
            +- Aggregate [agent_gu#197, legal_dong#196],
                 [agent_gu#197, legal_dong#196,
                  count(apt_seq#188) AS trades_2024#182L,
                  round(avg(deal_amount#193), 0) AS avg_2024#183,
                  round(avg(deal_amount#210), 0) AS avg_2023#184,
                  round((((avg(deal_amount#193) - avg(deal_amount#210))
                          / avg(deal_amount#210)) * cast(100 as double)),
                        2) AS growth_pct#185,
                  avg(deal_amount#210) AS avg(deal_amount#210)#238]
               +- Filter (((deal_year#190 = 2024)
                           AND (deal_year#207 = 2023))
                          AND agent_gu#197 LIKE 서울%)
                  +- Join Inner, (legal_dong#196 = legal_dong#213)
                     :- SubqueryAlias curr
                     :  +- SubqueryAlias nessie.real_estate.apt_trades
                     :     +- RelationV2[apt_name#186, ..., legal_dong#196,
                                          agent_gu#197, ...,
                                          deal_date#202]
                                  nessie.real_estate.apt_trades ...
                     +- SubqueryAlias prev
                        +- SubqueryAlias nessie.real_estate.apt_trades
                           +- RelationV2[apt_name#203, ..., legal_dong#213,
                                          agent_gu#214, ...,
                                          deal_date#219]
                                  nessie.real_estate.apt_trades ...
```

### Java 개발자 관점

- **`exprId`는 Java의 객체 참조 ID와 같다**. `legal_dong#196`과
  `legal_dong#213`은 이름도 타입도 같지만 plan 트리 안에서는 서로
  다른 참조다. self-join에서 "curr의 legal_dong"과 "prev의
  legal_dong"을 구별하는 근본적인 메커니즘이 바로 이것. Java
  에서 `==` (참조 비교)와 `.equals()` (값 비교)를 구분하는 것과
  정확히 같은 layer의 개념.
- `RelationV2[apt_name#186, ..., deal_date#202]`: 17개 컬럼 모두
  등장. **아직 column pruning 전**이다. Analyzer의 책임은 심볼을
  해결하는 것이지, 최적화가 아니다 — 최적화는 다음 단계.
- `avg(deal_amount#210) AS avg(deal_amount#210)#238`: 이 synthetic
  attribute가 Aggregate의 output 리스트에 **HAVING 절 참조용으로만
  추가**된 것. HAVING이 `AVG(prev.deal_amount) > 0`을 필요로 하므로,
  Aggregate 노드가 이 중간 값을 밖으로 노출하도록 ID `#238`을 부여.
  Java 개발자 관점: 람다에서 closure에 캡처해야 할 값을 명시적으로
  밖으로 뽑아두는 동작과 유사.
- `SubqueryAlias`가 이중으로 쌓여 있는 것 (`SubqueryAlias curr` ->
  `SubqueryAlias nessie.real_estate.apt_trades` -> `RelationV2`):
  하나는 쿼리 쓴 사람이 준 별칭 (`curr`), 다른 하나는 Analyzer가
  자동으로 붙인 정규화된 테이블 경로. 이들은 Optimizer가 곧 제거함.
- `cast(5 as bigint)`, `cast(0 as double)`: 구문의 `5`와 `0`은
  Scala `Int`로 파싱되었다가 HAVING 절이 비교하는 aggregate 타입
  (count는 bigint, avg는 double) 에 맞춰 Analyzer가 타입 강제 변환
  (TypeCoercion) 삽입. Java의 auto-boxing과 암묵적 변환에 대응.

### 관찰 요점

- Analyzed plan은 의미적으로 "해결된" 상태지만 **실행 가능한
  형태는 아니다** — 모든 컬럼을 다 읽고 Join 후 Filter를 적용하는
  비효율적 구조 그대로다. 이 비효율성을 해결하는 것이 다음 단계.
- `curr`와 `prev`의 `RelationV2`는 인스턴스로는 별개다 (다른
  `#186~#202` vs `#203~#219` ID 레인지). Optimizer가 두 scan을
  별개의 서브트리로 다루는 근거.

---

## 03 Logical Optimization

### Plan 트리 (raw 03 OPTIMIZED 섹션 발췌)

```
GlobalLimit 20
+- LocalLimit 20
   +- Sort [growth_pct#185 DESC NULLS LAST], true
      +- Project [agent_gu#197, legal_dong#196, trades_2024#182L,
                  avg_2024#183, avg_2023#184, growth_pct#185]
         +- Filter (isnotnull(avg(deal_amount#210)#238)
                    AND ((trades_2024#182L >= 5)
                         AND (avg(deal_amount#210)#238 > 0.0)))
            +- Aggregate [agent_gu#197, legal_dong#196],
                 [..., round((((avg(deal_amount#193) - avg(deal_amount#210))
                               / avg(deal_amount#210)) * 100.0), 2)
                       AS growth_pct#185,
                       avg(deal_amount#210) AS avg(deal_amount#210)#238]
               +- Project [apt_seq#188, deal_amount#193, legal_dong#196,
                           agent_gu#197, deal_amount#210]
                  +- Join Inner, (legal_dong#196 = legal_dong#213)
                     :- Project [apt_seq#188, deal_amount#193,
                                 legal_dong#196, agent_gu#197]
                     :  +- Filter (((isnotnull(deal_year#190)
                                     AND isnotnull(agent_gu#197))
                                    AND (deal_year#190 = 2024))
                                   AND StartsWith(agent_gu#197, 서울))
                                  AND isnotnull(legal_dong#196))
                     :     +- RelationV2[apt_seq#188, deal_year#190,
                                         deal_amount#193, legal_dong#196,
                                         agent_gu#197]
                                    nessie.real_estate.apt_trades
                     +- Project [deal_amount#210, legal_dong#213]
                        +- Filter ((isnotnull(deal_year#207)
                                    AND (deal_year#207 = 2023))
                                   AND isnotnull(legal_dong#213))
                           +- RelationV2[deal_year#207, deal_amount#210,
                                         legal_dong#213]
                                    nessie.real_estate.apt_trades
```

### Analyzed 대비 변경점

| 변경 | Analyzed | Optimized |
|---|---|---|
| RelationV2 컬럼 수 | 17 (curr), 17 (prev) | **5** (curr), **3** (prev) |
| SubqueryAlias | 이중으로 쌓여 있음 | 완전히 제거됨 |
| WHERE filter 위치 | Join 위 단일 Filter | Join의 각 child 아래 분리 |
| 파생된 IS NOT NULL | 없음 | `legal_dong IS NOT NULL`, `deal_year IS NOT NULL`, `agent_gu IS NOT NULL` |
| `LIKE '서울%'` | 원형 | `StartsWith(agent_gu, 서울)` |
| `cast(5 as bigint)` | 있음 | `5` (folded) |
| `cast(100 as double)` | 있음 | `100.0` |
| HAVING 절 추가 조건 | 없음 | `isnotnull(avg(deal_amount#210)#238)` (division by zero 가드와 결합) |

### Java 개발자 관점

- **Column Pruning (17→5, 17→3)**: Java builder pattern에서 필요한
  필드만 `.set()` 하고 나머지는 default로 두는 것과 유사. 다만
  여기서는 "읽지 않음"이 물리 I/O에까지 영향 — scan 노드에
  column list를 전달해 Parquet/Iceberg가 해당 컬럼 chunk만 읽도록.
- **Predicate Pushdown (Filter가 양쪽 child 아래로)**: "조건이
  한쪽에서만 결정되면 그쪽 데이터가 적을수록 유리"의 본능을 Catalyst
  가 자동화한 것. Java로 치면 stream pipeline에서 `.filter()`를
  `.map()` 뒤로 미는 대신 **앞으로 당기는** 재작성. 이 당기기가
  언제 valid한지 (outer join, aggregate, window를 넘을 수 있는지)는
  각 논리 규칙이 case-by-case로 정의.
- **`isnotnull` 자동 추가**: join key (`legal_dong`)에 대해 양쪽에
  `isnotnull`가 추가된 것은 `InferFiltersFromConstraints`류 규칙의
  작동. Inner equi-join은 null을 버리므로 미리 null을 쳐내도 의미
  보존 — 그리고 null을 미리 쳐내면 BatchScan까지 전파되어 I/O가
  줄어든다. Java 관점: `if (x != null)` 검사를 stream의 가장 앞에
  두어 null edge case 처리 분기를 줄이는 것과 같다.
- **`LIKE '서울%'` → `StartsWith(agent_gu, 서울)`**: LikeSimplification
  규칙. Java `Pattern.matches("서울.*", s)` 대신 `s.startsWith("서울")`
  을 쓰면 빠른 것과 정확히 같은 최적화. 컴파일러가 아닌 Catalyst가
  이걸 해준다.
- **`cast(5 as bigint)` → `5`**: `ConstantFolding`. Java의 javac도
  `5L`처럼 리터럴 레벨에서 타입을 확정해 런타임 cast 없앰.
- **HAVING의 `isnotnull(avg(...)#238)` 추가**: 수동으로 쓰지 않은
  null 가드가 등장한 이유 — HAVING에 `avg > 0`이 있기에, null이
  들어오면 조건이 false가 아니라 NULL이 되는데, 이후 Filter에서
  NULL 제거를 보장하려고. Three-valued logic을 다루는 SQL의
  관습적인 자동 변환.

### 적용된 최적화 규칙 (관찰 기반)

| 규칙 (추정 이름) | 관찰 증거 |
|---|---|
| `ColumnPruning` | RelationV2 17 컬럼 → 5/3 컬럼 |
| `CollapseProject` | SubqueryAlias 이중 제거 + Project 체인 정리 |
| `PushPredicateThroughJoin` + `PushDownPredicates` | Filter가 Join 각 child 아래로 분배 |
| `InferFiltersFromConstraints` | `legal_dong IS NOT NULL` 등 자동 추가 |
| `LikeSimplification` | `LIKE '서울%'` → `StartsWith(agent_gu, 서울)` |
| `ConstantFolding` | `cast(5 as bigint)` → `5` 등 |
| `NullPropagation` 또는 관련 규칙 | HAVING에 `isnotnull(avg(...))` 추가 |

> 📝 각 규칙의 정의 위치와 정확한 매칭 조건은 Optimizer.scala ingest
> 대기 중. 현재 기술은 "어떤 규칙이 이 효과를 냈을 것"의 잠정 매핑.

### 관찰 요점

- 이 단계가 없었다면 양쪽 apt_trades를 17 컬럼 × 전체 row로 읽어
  Join 후 filter — 수백 MB × 수백 MB의 cross match. 지금은 양쪽
  모두 **pushdown된 filter + pruned column으로 축소된 상태로**
  scan되며, join은 `(legal_dong, agent_gu 2024 filter 통과 row) ×
  (legal_dong, 2023 filter 통과 row)`로 들어간다.

---

## 04 Physical Planning

### Plan 트리 (raw 04 PHYSICAL 섹션 — Pre-AQE SparkPlan)

```
TakeOrderedAndProject(limit=20, orderBy=[growth_pct#185 DESC NULLS LAST],
  output=[agent_gu#197, legal_dong#196, trades_2024#182L,
          avg_2024#183, avg_2023#184, growth_pct#185])
+- Project [...]
   +- Filter (isnotnull(avg(deal_amount#210)#238)
              AND ((trades_2024#182L >= 5)
                   AND (avg(deal_amount#210)#238 > 0.0)))
      +- HashAggregate(keys=[agent_gu#197, legal_dong#196],
           functions=[count(apt_seq#188),
                      avg(deal_amount#193), avg(deal_amount#210)],
           output=[..., avg(deal_amount#210)#238])                # final
         +- HashAggregate(keys=[agent_gu#197, legal_dong#196],
              functions=[partial_count(apt_seq#188),
                         partial_avg(deal_amount#193),
                         partial_avg(deal_amount#210)],
              output=[agent_gu#197, legal_dong#196,
                      count#258L, sum#259, count#260L, sum#261,
                      count#262L])                                # partial
            +- Project [apt_seq#188, deal_amount#193, legal_dong#196,
                        agent_gu#197, deal_amount#210]
               +- SortMergeJoin [legal_dong#196], [legal_dong#213], Inner
                  :- Project [apt_seq#188, deal_amount#193,
                              legal_dong#196, agent_gu#197]
                  :  +- Filter (... deal_year#190 = 2024 ...)
                  :     +- BatchScan nessie.real_estate.apt_trades
                  :           [apt_seq#188, deal_year#190, deal_amount#193,
                  :            legal_dong#196, agent_gu#197]
                  :           (branch=null)
                  :           [filters=deal_year IS NOT NULL,
                  :                    agent_gu IS NOT NULL,
                  :                    deal_year = 2024,
                  :                    agent_gu LIKE '서울%',
                  :                    legal_dong IS NOT NULL,
                  :                    groupedBy=]
                  :           RuntimeFilters: []
                  +- Project [deal_amount#210, legal_dong#213]
                     +- Filter (... deal_year#207 = 2023 ...)
                        +- BatchScan nessie.real_estate.apt_trades
                              [deal_year#207, deal_amount#210,
                               legal_dong#213]
                              (branch=null)
                              [filters=..., deal_year = 2023, ...,
                               groupedBy=]
                              RuntimeFilters: []
```

그리고 05 단계 (AQE 래핑)에서는 SMJ 아래에 양쪽 `Sort [... ASC NULLS
FIRST], false, 0` + `Exchange hashpartitioning(legal_dong, 200),
ENSURE_REQUIREMENTS, [plan_id=100|101]`이 추가된다.

### Java 개발자 관점

- **SortMergeJoin**: Java `TreeMap.merge` 스타일이지만 memory에
  국한되지 않는다. "정렬된 두 stream을 동시에 걸으며 key가 같을
  때 pair를 방출"하는 알고리즘 — 단, Spark 버전은 disk spill과
  external sort를 지원해 메모리보다 큰 input에서도 작동. `Exchange
  hashpartitioning(legal_dong, 200)`은 양쪽 input을 같은 키로
  shuffling해서 "같은 key는 같은 partition으로" 보장하고, `Sort
  [legal_dong ASC NULLS FIRST]`가 각 partition 내부를 정렬한다.
  두 전제 (co-partitioned + sorted)가 갖춰져야 SMJ가 성립.
- **Exchange hashpartitioning**: Java `Map<K, List<V>>`로 key별로
  값을 재분배하는 것과 개념적으로 같지만, distributed. `200` 개의
  bucket으로 `hash(legal_dong) mod 200`을 해 각 executor로 보낸다.
  `ENSURE_REQUIREMENTS`는 "이 exchange는 자식 연산자가 요구하는
  distribution을 만족시키기 위해 삽입됨"을 의미 — Spark가 자동
  삽입했다는 표식. 사용자가 `repartition`을 쓴 경우 `REPARTITION_BY_COL`
  같은 다른 마커가 붙는다. 이 두 Exchange가 삽입된 정확한 경로는
  [04/ensure-requirements.md](../04-physical-planning/ensure-requirements.md)의
  **multi-child co-partitioning 분기**(`EnsureRequirements.scala:99-205`)
  — SMJ가 양쪽 자식에 `ClusteredDistribution([legal_dong])`을 요구하지만
  양쪽 자식 모두 `BatchScan`의 `UnknownPartitioning` 출력 상태라 어느 한
  쪽 spec을 채택할 후보(`canCreatePartitioning=true`)가 없어 양쪽 모두
  `ShuffleExchange`가 삽입된다. `reorderJoinKeys`(`:264-326`)는 이 케이스
  에서는 미발동 (key가 단일).
- **`plan_id=100`, `plan_id=101`**: 두 Exchange의 고유 식별자.
  AQE는 stage 경계를 이 plan_id로 추적해 어느 ShuffleQueryStage가
  어느 Exchange에 대응되는지 매핑한다. Java에서 `System.identityHashCode`
  로 객체를 추적하는 것과 목적이 비슷.
- **HashAggregate partial → final (MapReduce combine+reduce)**:
  partial은 각 partition 내 `HashMap<GroupKey, AggBuffer>` — key가
  `(agent_gu, legal_dong)`, value가 `(count, sum_2024, count_2024,
  sum_2023, count_2023)`. 각 row가 들어오면 map의 entry를 꺼내
  in-place update. final은 shuffle 후 같은 key로 도착한 여러
  partial buffer들을 합산해 최종 `count`와 `avg` (= sum/count)
  산출. Java MapReduce의 Combiner (Mapper.cleanup에서 흘려보내는
  partial 결과)와 Reducer (key별 buffer merge)의 정확한 대응이다.
  **이 쿼리에서는 partial과 final 사이 별도 Exchange가 생성되지
  않았다** — 왜냐하면 SMJ 결과가 이미 `legal_dong` 기준으로
  partitioned되어 있고, partial HashAggregate가 그 위에 올라타면서
  partitioning이 유지되어 final HashAggregate가 요구하는 `(agent_gu,
  legal_dong)` distribution을 그대로 재사용 가능했기 때문.
- **TakeOrderedAndProject**: ORDER BY + LIMIT의 fused 연산자.
  Java의 `stream.sorted().limit(20)`을 순진하게 구현하면 전체 정렬
  후 자르지만, Spark는 각 partition에서 **local top-20만 heap으로
  유지** → 모으기 → global top-20. `PriorityQueue`를 partition당
  하나씩 두고 마지막에 merge하는 패턴.
- **BatchScan (Iceberg V2)**: leaf. 실제 어떤 Parquet 파일을
  읽을지, manifest skip이 얼마나 되는지는 Iceberg connector
  영역 — Spark plan에서는 `filters=[...]`의 리스트로 **Spark가
  Iceberg에 무엇을 제안했는지**만 보인다. Iceberg가 그 중 무엇을
  실제 파일 skip에 썼는지는 Spark UI의 scan metric 필요.

### Why SortMergeJoin? (JoinSelection 해석)

참조: [04-physical-planning/join-strategy-hints.md](../04-physical-planning/join-strategy-hints.md).

EXPLAIN COST 통계 (Logical Optimized 기준):

```
Filter (curr side), Statistics(sizeInBytes=32.7 MiB)
RelationV2 (curr side), Statistics(sizeInBytes=34.5 MiB, rowCount=5.32E+5)

Filter (prev side), Statistics(sizeInBytes=12.6 MiB)
RelationV2 (prev side), Statistics(sizeInBytes=14.2 MiB, rowCount=5.32E+5)
```

`JoinSelection`의 분기 순서 (3.5.6 기준):
1. Broadcast hint? — 없음.
2. 한쪽이 `autoBroadcastJoinThreshold=10 MiB` 미만? — **curr 32.7 MiB,
   prev 12.6 MiB 둘 다 초과**. BroadcastHashJoin 탈락.
3. SHUFFLE_HASH hint? — 없음.
4. SortMergeJoin 조건 (equi-join key + orderable + preferSMJ=true)?
   — `legal_dong = legal_dong` equi-join, key 타입 orderable, 설정
   `preferSortMergeJoin=true`. **성립 → SortMergeJoin 선택**.

Broadcast도 SHJ도 아닌, 설정 기반 기본 경로로 안착한 전형적인 케이스.

### AQE가 이 선택을 바꿀 여지

`spark.sql.adaptive.autoBroadcastJoinThreshold`가 런타임에 더 크게
잡혀 있고, shuffle map stage 완료 후 **한쪽 실제 출력 크기가 그
값 미만**으로 드러나면 AQE가 SMJ를 BroadcastHashJoin으로 변경 가능.
또는 map output 크기가 `maxShuffledHashJoinLocalMapThreshold` 미만
이면 SMJ → ShuffledHashJoin 전환. 이 캡처는 `isFinalPlan=false`라
해당 여부 미확정.

### BatchScan filters pushdown

```
[filters=deal_year IS NOT NULL, agent_gu IS NOT NULL,
         deal_year = 2024, agent_gu LIKE '서울%',
         legal_dong IS NOT NULL,
         groupedBy=]
```

이 5개 filter가 Iceberg에 전달되었다는 사실만 plan에서 확정 가능.
각 filter가 partition pruning, manifest skip, row group skip,
row-level filter 중 어디에 쓰였는지는 Iceberg의 내부 전략 — Iceberg
소스 ingest 대기.

> 📝 Iceberg 소스 ingest 대기 중 (iceberg-spark-runtime-3.5_2.12:1.7.1,
> `SparkScanBuilder`, `SparkBatch`). 상세는
> [04-physical-planning/batch-scan-iceberg.md](../04-physical-planning/batch-scan-iceberg.md).

### 관찰 요점

- 물리 plan은 `TakeOrderedAndProject`, `HashAggregate` 2회,
  `SortMergeJoin`, `Exchange`, `BatchScan` 2회, 그리고 `Project`/
  `Filter`의 조합으로 9단계 중 04의 본 모습을 거의 다 드러낸다.
- `Statistics(sizeInBytes=413.3 TiB)`가 Join 결과 logical 추정으로
  찍혔다 — column-level histogram 통계 부재 시 Spark의 fallback
  cardinality estimator의 특유 현상. "절대 이만큼 안 되지만
  상한이다"의 안전한 과추정. 이 숫자는 **AQE 전에만 존재**하고,
  실제 shuffle 결과 통계가 들어오면 AQE가 재조정.

---

## 05 Cost & AQE

### Plan 트리 (raw 05 EXECUTED 섹션 발췌)

```
AdaptiveSparkPlan isFinalPlan=false
+- TakeOrderedAndProject(...)
   +- ... (위 Physical과 동일) ...
      +- SortMergeJoin [legal_dong#196], [legal_dong#213], Inner
         :- Sort [legal_dong#196 ASC NULLS FIRST], false, 0
         :  +- Exchange hashpartitioning(legal_dong#196, 200),
                ENSURE_REQUIREMENTS, [plan_id=100]
         :     +- ...
         +- Sort [legal_dong#213 ASC NULLS FIRST], false, 0
            +- Exchange hashpartitioning(legal_dong#213, 200),
                 ENSURE_REQUIREMENTS, [plan_id=101]
               +- ...
```

### `isFinalPlan=false`의 의미 — 이 캡처에서는 AQE 결정 관찰 불가

`AdaptiveSparkPlan isFinalPlan=false`는 "AQE wrapper는 설치됐지만
**아직 어떤 shuffle도 materialized 되지 않아 런타임 재최적화가
적용되지 않은 상태**"를 의미한다. `.explain()`만 호출되고 `.show()`
또는 `.collect()`가 아직 실행되지 않았기 때문 — AQE는 실행 중 각
shuffle stage 완료 시점마다 재-plan을 일으킨다.

참조: [05-cost-and-aqe/aqe-overview.md](../05-cost-and-aqe/aqe-overview.md).

Java 개발자 관점: **JIT 컴파일러가 "hot method인지 판단하려면 실제
실행 카운트가 필요"** 한 것과 정확히 같은 사정. `.explain()`은
컴파일된 바이트코드를 dump 하는 동작이고, `.show()`가 실제 프로파일링
실행이다. 프로파일링이 없으면 JIT은 기본 인라이닝만 할 뿐 호출
빈도 기반 최적화를 못 한다.

### 관찰 가능한 AQE 준비 흔적

| 관찰 | 의미 |
|---|---|
| `AdaptiveSparkPlan` wrapper 존재 | `spark.sql.adaptive.enabled=true` 확인 |
| `plan_id=100`, `plan_id=101` | AQE가 두 Exchange를 별도 `ShuffleQueryStage`로 스케줄할 경계 식별자 |
| `ENSURE_REQUIREMENTS` | 사용자 지정이 아닌 Spark 자동 삽입 Exchange — AQE가 재작성 가능 후보 |

### 관찰 **불가**한 AQE 결정 (이 캡처에서는 공백)

- SMJ → BroadcastHashJoin 전환 여부
- SMJ → ShuffledHashJoin 전환 여부 (`maxShuffledHashJoinLocalMapThreshold`)
- Partition coalesce (200 → 실제 task 수 축소)
- Skew join 분할
- Dynamic `repartition`/`REBALANCE` 조정

이를 관찰하려면 `.show()` 실행 후 `df.queryExecution.executedPlan`
재캡처가 필요하다. 후속 trace 대상.

### EXPLAIN COST의 관찰

EXPLAIN COST가 제공한 통계 (Optimized Logical Plan 기준):

```
RelationV2 (curr, after pruning): sizeInBytes=34.5 MiB, rowCount=5.32E+5
Filter (curr):                    sizeInBytes=32.7 MiB
(row count는 후속에 표시되지 않음 — column histogram 부재)

RelationV2 (prev, after pruning): sizeInBytes=14.2 MiB, rowCount=5.32E+5
Filter (prev):                    sizeInBytes=12.6 MiB

Join Inner:                       sizeInBytes=413.3 TiB   ← fallback 상한
Aggregate:                        sizeInBytes=378.8 TiB
Filter (HAVING):                  sizeInBytes=378.8 TiB
Project:                          sizeInBytes=344.4 TiB
Sort:                             sizeInBytes=344.4 TiB
LocalLimit:                       sizeInBytes=344.4 TiB
GlobalLimit:                      sizeInBytes=1600.0 B, rowCount=20
```

`413.3 TiB` 같은 비현실적 수치는 **Spark의 join cardinality estimator
가 column 레벨 histogram이 없을 때 상한으로 `leftRows × rightRows`를
쓰기 때문**. `5.32E+5 × 5.32E+5 ≈ 2.83E+11` row × row당 몇 바이트
— 안전한 과추정으로 TiB 스케일이 나온다. 실제 결과는 이보다 수십만
배 작다. 이 숫자는 AQE가 실제 shuffle 크기를 보고 나면 무관해진다.
**Statistics가 "가장 비관적 상한"이라는 점을 잊지 않는 것이 학습
포인트** — Java 개발자가 처음에 가장 놀랄 만한 숫자.

참조: [05-cost-and-aqe/leveraging-statistics.md](../05-cost-and-aqe/leveraging-statistics.md).

### 관찰 요점

- **이 캡처는 AQE의 준비만 보여주고 결정은 보여주지 않는다**. case
  파일 전반에 "AQE가 SMJ를 BHJ로 바꾸었다"같은 주장이 나오면 안
  된다 — 실제 실행 후 plan을 다시 캡처해야만 확정 가능.
- cost 수치는 "Catalyst가 보고 있는 데이터"지 "실제 실행 중 측정된
  데이터"가 아니다. AQE가 뒤를 맡기 전까지의 추정치.

---

## 06 Codegen

### 이 trace에서의 처리

Plan 파일 캡처에는 `df.queryExecution.debug.codegen()` 출력이
포함되지 않았다. **생성 Java를 추측으로 기술하지 않는다**는 정책에
따라, 이 섹션은 fused 가능 구역만 plan 관찰 사실로 기록한다.

### WholeStageCodegen이 fuse 가능한 구역 (plan 구조 기반 추정)

SMJ와 Exchange는 codegen 경계 (Sort는 일반적으로 codegen-friendly).
큰 경계는 다음과 같다:

```
BatchScan → Filter → Project → Exchange [경계] Sort → SMJ → Project
                                                              ↓
                                     partial HashAggregate → [경계: 이 쿼리에선 생략]
                                                              ↓
                                     final HashAggregate → Filter → Project
                                                              ↓
                                                    TakeOrderedAndProject
```

- `BatchScan → Filter → Project`: scan 구현체가 codegen을 지원하면
  fuse 가능. Iceberg BatchScan은 V2 API 경로라 fuse 제약이 있을
  수 있음 — 확정은 생성 Java 필요.
- `Sort → SMJ → Project → partial HashAggregate`: 동일 partition
  내에서 처리되므로 fuse 가능성 높음.
- `final HashAggregate → Filter → Project`: shuffle 위에서 fuse
  가능.
- `TakeOrderedAndProject`: 단독 operator. heap-based top-N은 별도.

> 📝 실제 fuse된 영역과 생성 Java는 `df.queryExecution.debug.codegen()`
> 출력 후 `raw/codegen/`에 저장 → 후속 trace로 확정. 이 섹션은
> 경계 추정만.

### 관찰 요점

- Codegen 관찰 없이 "SMJ와 HashAggregate가 fuse되었다"고 단정하지
  않는다. 실제 생성 Java에서 `processNext()` 메서드 하나가 여러
  연산자를 인라인하고 있는지 확인해야 한다.

---

## 07 RDD & Stages

### Exchange 개수 기반 stage 경계 추정

Executed plan의 `Exchange hashpartitioning` 노드 2개:
- `plan_id=100` (curr side, `hashpartitioning(legal_dong#196, 200)`)
- `plan_id=101` (prev side, `hashpartitioning(legal_dong#213, 200)`)

각 Exchange가 **shuffle map stage 하나의 종료점**이 된다. 따라서:

| Stage 추정 | 하는 일 | Task 수 (대략) |
|---|---|---|
| Stage 0 (shuffle map) | curr BatchScan → Filter → Project → Exchange write | Iceberg가 결정하는 scan split 수 |
| Stage 1 (shuffle map) | prev BatchScan → Filter → Project → Exchange write | Iceberg가 결정하는 scan split 수 |
| Stage 2 (result) | 양쪽 shuffle read → Sort → SMJ → Project → partial HashAggregate → (이 경우 final HashAggregate 까지 fuse 가능, 자체 Exchange 없음) → Filter → Project → TakeOrderedAndProject.local | 200 (AQE coalesce 전 기본) |

**정확한 stage 개수와 task 수는 Spark UI에서 확인 권장**. AQE가
활성이므로 실제 실행 시 Stage 2의 task 수가 200에서 coalesce되어
줄어들 수 있고, partial/final 사이에 Spark가 추가 Exchange를
삽입하지 않았는지 최종 확인은 UI로 해야 한다.

### Java 개발자 관점

- **Stage 경계 = shuffle barrier**. Java 병렬 stream에서
  `.collect(Collectors.groupingBy(...))`가 barrier이듯, Exchange가
  "여기서부터는 partition 경계가 완전히 재구성된다"를 뜻함.
- 각 stage의 TaskSet은 내부적으로 **같은 코드 (한 이터레이터
  pipeline)을 실행하는 task 집합**. Java에서 `ForkJoinPool`에
  ForkJoinTask를 N개 제출하는 것과 구조가 같다.
- BatchScan의 task 수는 **Spark가 아니라 Iceberg의 scan planner가
  결정** — `SparkBatch.planInputPartitions()`가 반환하는 수. 다른
  connector는 다른 split 전략을 쓴다.

### 관찰 요점

- 이 trace는 stage를 **plan에서 추정**만 한다. 실제 DAG와 task
  수는 Spark UI (SQL tab → Details for Query, Stages tab)에서
  확정해야 한다. "정확한 stage/task 분석"은 별도 trace 주제로
  분리.

---

## 08 Scheduling

이 trace 범위 밖. DAGScheduler의 stage submission, TaskScheduler의
task 분배, locality preference는 Spark UI의 Event Timeline 또는
driver log가 필요하다 — `.explain()` 출력만으로는 관찰 불가.

> 📝 단계 08에 대한 별도 trace 또는 probe 필요.

## 09 Execution

이 trace 범위 밖. Executor 메모리 분할, shuffle write/read 실물,
Tungsten off-heap 사용, 실제 BatchScan의 Parquet read 동작은 Spark
UI의 metric (executor memory, shuffle bytes, GC time)과 Iceberg
scan metric이 필요.

> 📝 단계 09에 대한 별도 trace 또는 probe 필요.

---

## 이 쿼리가 드러낸 Wiki 갭

다음 trace에서 자연스럽게 이어갈 후보들을 단계별로 그룹핑한다.
플래그: [관찰됨] 이 쿼리의 plan에서 직접 목격, [후속 페이지 후보]
skeleton 필요, [소스 ingest 필요] 해당 단계의 Spark 또는 Iceberg
소스 확보 필요.

### 단계 01 — Parsing

- [관찰됨] `'UnresolvedRelation [multi-part identifier]`의 구조 —
  `[db, schema, table]` 3-part naming이 Nessie/Iceberg catalog와
  어떻게 대응되는지.
- [후속 페이지 후보] `wiki/01-parsing/unresolved-logical-plan.md`
  — `'` prefix 노드 종류들의 총 카탈로그 (Relation, Attribute,
  Function, Having, Alias).
- [소스 ingest 필요] `sql/catalyst/.../parser/AstBuilder.scala`,
  `SparkSqlParser`.

### 단계 02 — Analysis

- [관찰됨] synthetic attribute (`avg(deal_amount#210)#238`)가
  HAVING 때문에 추가되는 현상.
- [관찰됨] TypeCoercion에 의한 `cast(5 as bigint)`, `cast(0 as
  double)` 삽입.
- [후속 페이지 후보] `wiki/02-analysis/type-coercion.md`,
  `wiki/02-analysis/resolve-aggregate-functions.md`,
  `wiki/02-analysis/subquery-alias-and-exprid.md`.
- [소스 ingest 필요] `sql/catalyst/.../analysis/Analyzer.scala` +
  `CheckAnalysis.scala`.

### 단계 03 — Logical Optimization

- [관찰됨] 규칙 최소 5개가 이 쿼리 하나에서 동시 발현
  (`ColumnPruning`, `PushDownPredicates`, `InferFiltersFromConstraints`,
  `LikeSimplification`, `ConstantFolding`).
- [후속 페이지 후보] `wiki/03-logical-optimization/column-pruning.md`,
  `.../infer-filters-from-constraints.md`,
  `.../like-simplification.md`, `.../constant-folding.md`.
- [소스 ingest 필요] `sql/catalyst/.../optimizer/Optimizer.scala` +
  규칙별 정의 파일 (`Joins.scala`, `PushDownPredicates.scala`
  등).

### 단계 04 — Physical Planning

- [관찰됨] `TakeOrderedAndProject`의 fused `LIMIT + ORDER BY` —
  local top-N + global top-N 패턴.
- [관찰됨] DataSourceV2Strategy에 의한 `RelationV2` → `BatchScan`
  변환 (Spark 내부 영역).
- [후속 페이지 후보] `wiki/04-physical-planning/take-ordered-and-project.md`,
  `.../data-source-v2-strategy.md` (Spark 측 V2 변환).
- [소스 ingest 필요] `sql/core/.../datasources/v2/DataSourceV2Strategy.scala`,
  `sql/core/.../limit.scala` (TakeOrderedAndProjectExec).
- [소스 ingest 필요] **iceberg-spark-runtime-3.5_2.12:1.7.1** 의
  `SparkScanBuilder`, `SparkBatch`, `SparkPartitioningAwareScan`,
  manifest 평가 로직. `batch-scan-iceberg.md`가 대기 중.

### 단계 05 — Cost & AQE

- [관찰됨] `isFinalPlan=false`의 의미 — `.explain()` 시점에는
  AQE 결정 없음.
- [관찰됨] Column histogram 부재 시 join cost가 413 TiB로 튀는
  fallback estimator.
- [후속 페이지 후보] 별도 페이지보다는 기존 `aqe-overview.md`와
  `leveraging-statistics.md`의 L5 (How to Observe) 섹션을
  이 사례로 강화.
- [별도 trace 필요] **실제 execution 후 `isFinalPlan=true`인 plan
  재캡처** — AQE 결정을 관찰하는 trace. 같은 세 쿼리에 대해
  `.show()` 후 `queryExecution.executedPlan.toString()`을
  `raw/explain/2026-04-19-*-executed.txt`로 저장.

### 단계 06 — Codegen

- [후속 trace 필요] `df.queryExecution.debug.codegen()` 캡처와
  `raw/codegen/2026-04-19-query-A.java` 저장 후 분리된 case 파일.
- [후속 페이지 후보] `wiki/06-codegen/whole-stage-codegen.md`,
  `.../reading-generated-java.md` (후자는 `_probes/`).

### 단계 07 — RDD & Stages

- [후속 페이지 후보] `wiki/07-rdd-and-stages/exchange-and-stage-boundary.md`
  — Exchange 개수 = shuffle map stage 수, plan_id의 역할.
- [소스 ingest 필요] `sql/core/.../exchange/ShuffleExchangeExec.scala`
  + `scheduler/DAGScheduler.scala`의 stage split 로직.

### 단계 08-09

- [별도 trace 필요] Spark UI 스냅샷과 metric 기반 trace. plan
  문자열만으로는 이 단계를 다룰 수 없음.

---

## 개인 학습 노트

<!-- 이 공간은 Song이 직접 채운다. Claude는 수정 금지. -->
<!--
    이 쿼리를 돌려본 후 깨달은 것:
    - `legal_dong#196` vs `#213`이 self-join의 정체성 구분 방법임을
      알고 나니 EXPLAIN이 덜 무서워졌다.
    - ...
-->

---

## Trace 메타

- 생성 명령: `/trace query-A-self-join-growth`
- 소요 추정: Plan 5개 단계 × 9 파이프라인 단계 분석
- 생성된 skeleton: 4개 (analyzer-overview, predicate-pushdown,
  hash-aggregate-partial-final, batch-scan-iceberg)
- 인용한 verified 페이지: `04-physical-planning/join-strategy-hints.md`
- 다음 우선 trace: 같은 쿼리 `.show()` 후 executed plan 재캡처 (AQE
  결정 관찰) 또는 `debug.codegen()` 캡처 (단계 06).
