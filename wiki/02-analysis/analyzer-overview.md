---
stage: 02-analysis
title: Analyzer — Unresolved에서 Analyzed LogicalPlan으로
spark_versions: ["3.5"]
sources:
  - raw/explain/2026-04-19-query-A-self-join-growth.txt
last_verified: 2026-04-19
confidence: tentative
related:
  - 00-overview/execution-pipeline.md
  - case/2026-04-19-query-A-self-join-growth.md
configs: []
apis:
  - SparkSession.sql
  - DataFrame (lazy)
tags: [analyzer, resolution, exprId, attribute-reference]
---

## L1. What & Why

Analyzer는 Parsing 단계가 내놓은 **Unresolved LogicalPlan**을 받아,
plan 트리의 모든 심볼 (테이블명, 컬럼명, 함수명)을 실제 카탈로그 항목
및 타입 정보에 바인딩해 **Analyzed LogicalPlan**을 산출한다. 파이프라인
순서상 단계 02. 이 단계가 없으면 Optimizer는 아직 `'UnresolvedRelation`
같은 심볼 노드를 가진 plan에서 규칙을 적용할 수 없다 — 어떤 컬럼이
어느 테이블에서 왔는지, 타입이 무엇인지, `count(x)`가 aggregate
function인지 column 참조인지를 알 수 없기 때문.

Analyzer의 주된 산출물은 모든 `AttributeReference`가 **고유
`exprId`** (예: `#188`, `#210`)를 갖는 plan이다. `exprId`는 plan
트리 전체에 걸쳐 "같은 속성"을 추적하는 식별자로, 후속 단계의 규칙
매칭·치환의 기반이 된다.

Java 개발자 관점에서 `exprId`는 JVM의 객체 참조 ID에 해당한다 —
이름이 같아도 서로 다른 참조일 수 있고 (self-join에서 `legal_dong#196`
과 `legal_dong#213`), 이름이 달라도 같은 참조일 수 있다 (alias).

## L2. Inputs & Outputs

**입력**: Unresolved `LogicalPlan` — 루트가 될 수 있는 노드 예시:
- `'UnresolvedRelation [db, schema, table]` — 테이블 참조
- `'UnresolvedAttribute` — 컬럼 참조
- `'UnresolvedFunction name(args)` — 함수 참조
- `'UnresolvedHaving` — HAVING 절 (aggregate 해결 전)
- `'SubqueryAlias name` — 별칭 (아직 내부 해결 전일 수 있음)

프라임 prefix `'`가 "unresolved"의 표식.

**출력**: Analyzed `LogicalPlan` — 모든 참조가 다음으로 교체됨:
- `UnresolvedRelation` → `SubqueryAlias {alias} → RelationV2[col#id, ...]`
  (또는 `HiveTableRelation`, `LogicalRelation` 등 소스에 따라)
- `UnresolvedAttribute` → `AttributeReference(name, dataType, nullable, exprId=#N)`
- `UnresolvedFunction` → 구체적 `AggregateFunction`, `ScalarFunction` 등
- `UnresolvedHaving` → `Filter` (aggregate expression 참조 해결됨)

쿼리 A 관찰: `'UnresolvedRelation [nessie, real_estate, apt_trades]`
가 Analyzed 단계에서 17개 컬럼을 모두 스키마로 가진
`RelationV2[apt_name#186, apt_dong#187, ..., deal_date#202]`로 해결됨.
self-join 양쪽은 같은 `RelationV2`를 가리키지만 **각 컬럼마다 별도
`exprId`**를 부여받음 (`legal_dong#196` vs `legal_dong#213`), 이것이
후속 단계에서 두 참조를 구별하는 근거가 된다.

## L3. Key Mechanisms

> 📝 docs 요약 수준. 소스 코드 ingest 대기 중 (`sql/catalyst/src/main/scala/org/apache/spark/sql/catalyst/analysis/Analyzer.scala`).

Analyzer는 rule batch로 구성되며 대표적으로 `ResolveRelations`,
`ResolveReferences`, `TypeCoercion`, `ResolveAggregateFunctions`,
`ResolveWindowOrder` 등이 있다. 각 규칙의 정확한 동작과 적용 순서는
소스 ingest 후 기록.

## L4. Performance Levers

> 📝 docs 요약 수준. 소스 코드 ingest 대기 중.

Analyzer 자체가 성능 레버를 많이 노출하지는 않지만, 카탈로그 lookup
비용, 캐시 동작, analyzer iteration 상한 등이 지연의 원인이 될 수
있다. 구체 레버는 후속 ingest.

## L5. How to Observe

- `df.queryExecution.analyzed` — Analyzed LogicalPlan 출력
- `df.explain(mode="extended")` — Parsed/Analyzed/Optimized/Physical 4개
  plan 모두 출력
- 관찰 포인트: `AttributeReference`의 `#N` exprId가 등장하기 시작하는지,
  `RelationV2` (또는 소스별 relation)가 구체 스키마를 갖는지.

> 📝 probe 페이지 미작성. 후속 `/probe 02-analysis` 대상.
