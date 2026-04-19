---
stage: 04-physical-planning
title: HashAggregate — Partial과 Final 2단계 집계
spark_versions: ["3.5"]
sources:
  - raw/explain/2026-04-19-query-A-self-join-growth.txt
last_verified: 2026-04-19
confidence: tentative
related:
  - 00-overview/execution-pipeline.md
  - 04-physical-planning/join-strategy-hints.md
  - 05-cost-and-aqe/aqe-overview.md
  - case/2026-04-19-query-A-self-join-growth.md
configs:
  - spark.sql.execution.useObjectHashAggregateExec
  - spark.sql.shuffle.partitions
apis:
  - DataFrame.groupBy
  - SQL GROUP BY
  - SQL aggregate functions (COUNT, AVG, SUM, ...)
tags: [aggregate, physical-planning, partial-final, shuffle]
---

## L1. What & Why

물리 plan에서 GROUP BY와 aggregate function은 대부분의 경우 **두
개의 HashAggregate 노드**로 실현된다: 입력 데이터 바로 위의
`partial_<func>` 집계(map-side), 그리고 shuffle 뒤의
`<func>` 집계(reduce-side). 이 분할이 파이프라인 단계 04
(Physical Planning)에서 `Aggregation` strategy에 의해 결정된다.

이 구조의 목적은 셔플되는 데이터 양을 줄이는 것이다. Partial 단계가
각 partition 내에서 grouping 키별 부분 집계 상태 (count, sum 등)를
누적한 뒤, final 단계가 키로 shuffle된 부분 상태들을 병합한다.

Java 개발자 관점에서는 정확히 MapReduce의 **Combine + Reduce**
패턴이다. Combine이 mapper 로컬에서 그룹별 중간 상태를 만들고,
Reduce가 모든 mapper의 해당 그룹 상태를 받아 최종 값을 계산한다.
Partial/Final 분할이 없으면 모든 원시 row가 shuffle을 통과하게
되어 네트워크가 병목이 된다.

## L2. Inputs & Outputs

**입력**: Optimized `LogicalPlan`의 `Aggregate(groupingExprs,
aggregateExprs, child)` 노드.

**출력**: 일반적으로 다음 3계층의 `SparkPlan`:

1. Partial `HashAggregate(keys=..., functions=[partial_<f>, ...],
   output=[..., <intermediate state columns>])`
2. `Exchange hashpartitioning(<grouping keys>, N)`
3. Final `HashAggregate(keys=..., functions=[<f>, ...],
   output=[..., <final aggregate columns>])`

**쿼리 A 관찰** (Physical Plan 노드 13-14):

```
(13) HashAggregate                                       # partial
  Keys [2]: [agent_gu#197, legal_dong#196]
  Functions [3]: [partial_count(apt_seq#188),
                  partial_avg(deal_amount#193),
                  partial_avg(deal_amount#210)]
  Aggregate Attributes [5]: [count#253L, sum#254, count#255L,
                             sum#256, count#257L]
  Results [7]: [agent_gu#197, legal_dong#196,
                count#258L, sum#259, count#260L, sum#261, count#262L]

(14) HashAggregate                                       # final
  Keys [2]: [agent_gu#197, legal_dong#196]
  Functions [3]: [count(apt_seq#188),
                  avg(deal_amount#193),
                  avg(deal_amount#210)]
  Results [7]: [agent_gu#197, legal_dong#196, ... trades_2024#182L,
                avg_2024#183, avg_2023#184, growth_pct#185, ...]
```

주목:
- `avg`는 partial 단계에서 `(sum, count)` 쌍으로 누적되고 (노드 13의
  `sum#254, count#253L` 등), final 단계에서 `sum/count`로 나누어
  평균 산출. aggregate buffer의 중간 스키마가 입출력과 다름.
- grouping 키 `(agent_gu, legal_dong)`로 partial → final 사이에
  `Exchange hashpartitioning(legal_dong, 200)` 아닌 — **이 쿼리에서는
  SortMergeJoin 바로 아래에 이미 `legal_dong` 기준 hashpartitioning이
  있어 partial HashAggregate가 join 출력 위에 직접 fuse되고 별도
  `Exchange`가 생성되지 않았다**. partial과 final 사이 shuffle은
  기존 join shuffle에 흡수됨. plan 순서로 이 점 확인 가능.

## L3. Key Mechanisms

> 📝 docs 요약 수준. 소스 코드 ingest 대기 중 (`sql/core/src/main/scala/org/apache/spark/sql/execution/SparkStrategies.scala` 내 `Aggregation` strategy + `sql/core/.../aggregate/AggUtils.scala`).

관찰된 사실 (소스 ingest 전):
- `HashAggregateExec`가 기본. UDT 등 hashable하지 않은 타입이 있으면
  `ObjectHashAggregateExec` 또는 `SortAggregateExec`로 fallback.
- partial 단계의 출력 스키마는 `aggregateFunction.aggBufferAttributes`
  가 결정 (avg → sum, count 쌍; count → count; sum → sum; ...).
- Distinct aggregate는 rewrite되어 추가 단계가 들어감 (이 쿼리는
  해당 없음).

## L4. Performance Levers

> 📝 docs 요약 수준. 소스 코드 ingest 대기 중.

- `spark.sql.shuffle.partitions` (이 캡처: 200): partial→final 사이
  (또는 재사용된 join shuffle)의 parallelism과 partition 크기를 결정.
- `spark.sql.adaptive.coalescePartitions.enabled` (AQE): 런타임에
  작은 partition을 병합해 final aggregate 쪽 task 수를 조정. AQE의
  coalesce가 가장 자주 작동하는 지점 중 하나.
- 높은 cardinality grouping 키 → partial 집계의 효과 감소 (거의
  row 수 그대로 shuffle됨). 이 쿼리의 `(agent_gu, legal_dong)`
  조합은 수십~수백 수준 예상으로 partial 효과 높음.

## L5. How to Observe

- `df.explain(mode="formatted")`에서 연속된 두 HashAggregate 노드를
  확인. 하나는 `functions=[partial_<f>, ...]`, 다른 하나는
  `functions=[<f>, ...]`.
- Spark UI → SQL tab → DAG 시각화에서 "HashAggregate" 두 박스와
  그 사이 Exchange 박스 (또는 앞선 shuffle로부터의 연결)를 관찰.
- partial output 스키마의 `sum/count` 쌍 등 중간 상태 컬럼 확인
  — avg의 내부 표현을 이해하는 단서.

> 📝 probe 페이지 미작성. 후속 `/probe 04-physical-planning` 대상.
