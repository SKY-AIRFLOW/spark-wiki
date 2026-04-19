---
stage: 03-logical-optimization
title: Predicate Pushdown
spark_versions: ["3.5"]
sources:
  - raw/explain/2026-04-19-query-A-self-join-growth.txt
last_verified: 2026-04-19
confidence: tentative
related:
  - 00-overview/execution-pipeline.md
  - 04-physical-planning/join-strategy-hints.md
  - case/2026-04-19-query-A-self-join-growth.md
configs:
  - spark.sql.optimizer.excludedRules
apis:
  - DataFrame.filter
  - DataFrame.where
  - SQL WHERE
  - SQL HAVING
tags: [optimizer, catalyst, predicate, pushdown]
---

## L1. What & Why

Predicate Pushdown은 `Filter` 노드를 plan 트리의 **소비자 쪽(위)에서
공급자 쪽(아래)으로 밀어넣는** 최적화의 총칭이다. 파이프라인 단계
03 (Catalyst Logical Optimization)에서 규칙 기반으로 수행된다. 효과는
두 계층에 걸친다:

1. **논리 plan 내부**: `Filter` 가 `Join` 위에 있을 때, 해당 조건이
   한쪽 child의 속성만 참조하면 그 child로 밀어내려 join 입력 row
   수를 축소.
2. **Data source 경계**: scan 노드 바로 위의 `Filter`는 data source가
   지원하는 한 스캔 자체의 `filters=[...]`로 전달되어 물리적 read
   비용을 줄인다 (파일 skip, Iceberg partition pruning, Parquet row
   group pruning).

이 최적화가 없으면 쿼리는 전체 테이블을 join 후 WHERE를 적용하게
되고, self-join처럼 양쪽 카디널리티가 큰 경우 중간 결과가 폭발한다.

## L2. Inputs & Outputs

**입력**: Analyzed `LogicalPlan`의 `Filter(condition, child)` 노드
— child 타입별 처리가 다르다:
- `Join`: equi-join 조건 외 조건들을 각 side로 분배
- `Aggregate`: grouping 키만 참조하는 조건은 아래로, aggregate
  expression 참조 조건은 그대로 유지
- `Project`: alias를 되돌려 아래로
- `Relation` / `RelationV2`: source API (`SupportsPushDownFilters`,
  `SupportsPushDownV2Filters`)가 수락하는 조건은 relation의 메타데이터에
  부착되어 scan 시 사용

**출력**: 같은 의미의 `LogicalPlan`이지만 `Filter`가 트리의 더 깊은
위치로 이동한 상태. 일부 조건은 `Filter` 노드에서 사라지고 `RelationV2`
의 `pushedFilters` (또는 BatchScan의 `filters=[...]`) 리스트로 이동.

**쿼리 A 관찰** (Optimized Plan):
- 원래 WHERE `((curr.deal_year=2024) AND (prev.deal_year=2023)) AND
  curr.agent_gu LIKE '서울%'` → Join 위 하나의 `Filter`
- 최적화 후: Join의 **양쪽 child 아래로 각각** 분배됨
  - curr 쪽: `Filter (isnotnull(deal_year) AND isnotnull(agent_gu) AND
    deal_year=2024 AND StartsWith(agent_gu, 서울) AND
    isnotnull(legal_dong))`
  - prev 쪽: `Filter (isnotnull(deal_year) AND deal_year=2023 AND
    isnotnull(legal_dong))`
- 나아가 Physical의 `BatchScan`에 `filters=[deal_year IS NOT NULL,
  agent_gu IS NOT NULL, deal_year=2024, agent_gu LIKE '서울%',
  legal_dong IS NOT NULL, groupedBy=]`로 전체가 data source까지 전달.

이 관찰은 "filter가 Optimized plan에서 사라졌다"가 아니라 "filter
조건이 scan node의 메타데이터로 통합되었다"가 정확한 기술임을
상기시킨다.

## L3. Key Mechanisms

> 📝 docs 요약 수준. 소스 코드 ingest 대기 중 (`sql/catalyst/src/main/scala/org/apache/spark/sql/catalyst/optimizer/Optimizer.scala` 내 `PushDownPredicates`, `PushPredicateThroughJoin`, `PushPredicateThroughNonJoin` 배치, 그리고 `sql/catalyst/.../datasources/v2/V2ScanRelationPushDown.scala`).

쿼리 A에서 관찰된 부산 효과들 (소스 ingest 전 잠정 기술):
- `InferFiltersFromConstraints`로 보이는 규칙이 `legal_dong IS NOT
  NULL`을 추론해 join nullsafe성 보장에 기여.
- `LikeSimplification`으로 `LIKE '서울%'` → `StartsWith(agent_gu,
  서울)` 재작성.
- `ConstantFolding`으로 `cast(5 as bigint)` → `5`, `cast(100 as
  double)` → `100.0`.

규칙 정의 위치와 정확한 적용 순서, fix-point 동작은 소스 ingest 후
기록.

## L4. Performance Levers

> 📝 docs 요약 수준. 소스 코드 ingest 대기 중.

- 데이터 소스가 filter pushdown을 지원해야 data source 경계 넘어
  의미가 있다 (Iceberg V2는 지원, 일부 JDBC는 타입별 제한).
- `spark.sql.optimizer.excludedRules`로 특정 규칙을 끌 수 있다.
  디버깅 외 운영 환경에서는 거의 변경하지 않는다.
- Join 조건과 filter가 어떻게 결합되느냐에 따라 한쪽으로만 전파될
  수도 있고 양쪽으로 inference될 수도 있다 (equi-join key의
  transitivity).

구체 수치와 규칙별 기여도는 후속 ingest.

## L5. How to Observe

- `df.queryExecution.optimizedPlan` — Optimized LogicalPlan 출력.
  `Filter` 노드의 위치가 Analyzed와 비교해 어디로 이동했는지 관찰.
- `df.explain(mode="formatted")` — Physical plan의 scan 노드 섹션에
  `PushedFilters` 또는 Iceberg의 `filters=[...]` 리스트 확인.
- EXPLAIN COST로 filter 전후 `Statistics(sizeInBytes=...)` 감소 관찰.

> 📝 probe 페이지 미작성. 후속 `/probe 03-logical-optimization` 대상.
