---
stage: 03-logical-optimization
title: CTE Inline — WithCTE/CTERelationDef/CTERelationRef 소거
spark_versions: ["3.5"]
sources:
  - raw/explain/2026-04-19-query-C-window-moving-avg.txt
last_verified: 2026-04-19
confidence: tentative
related:
  - 00-overview/execution-pipeline.md
  - 02-analysis/analyzer-overview.md
  - case/2026-04-19-query-C-window-moving-avg.md
configs:
  - spark.sql.optimizer.excludedRules
apis:
  - SQL WITH <name> AS (...) SELECT ...
  - DataFrame.createOrReplaceTempView (유사)
tags: [cte, catalyst, optimizer, inline, substitution]
---

## L1. What & Why

CTE (Common Table Expression, `WITH name AS (...)`)는 SQL 레벨에서 쿼리의
재사용 가능한 subquery를 이름으로 정의하는 구문이다. Catalyst는 이 구문을
plan 상에서 두 종류의 전용 논리 노드로 표현한 뒤 (`CTERelationDef`,
`CTERelationRef`), 논리 최적화 중에 조건이 맞으면 참조 지점에 실제 내용을
**inline**한다.

Inline의 본질은 "named subquery를 일반 subtree로 치환"이다. Inline 후 plan
은 CTE를 쓰지 않은 동등한 쿼리와 구조적으로 같아지며, 이후 규칙 (column
pruning, predicate pushdown 등)이 경계를 넘어 자유롭게 최적화할 수 있다.

Java 개발자 관점에서 CTE inline은 **JIT의 private method inlining**과
정확히 대응된다. 참조가 적고 단순한 helper method는 호출 지점에 inline되어
인라이너가 더 공격적인 최적화 (escape analysis, dead store elimination)를
할 수 있다. Catalyst도 마찬가지로 "inline 후 최적화 경계가 사라지면 더
많은 규칙이 발동"하기 위해 이 변환을 먼저 수행한다.

Inline은 **항상** 안전하지는 않다. CTE가 여러 번 참조되고 재계산 비용이
크면 inline이 되려 성능에 해가 될 수 있다 — 이 경우 Catalyst는 inline을
건너뛰고 참조를 유지, 물리 단계에서 1회 계산 + 재사용 (shuffle reuse 또는
InMemory cache) 전략을 택할 수 있다. 이 결정의 정확한 기준은 소스 ingest
후 기록.

## L2. Inputs & Outputs

### CTE를 표현하는 세 노드 (Analyzed 단계)

- **`WithCTE`**: CTE를 가진 쿼리 전체를 감싸는 루트. 자식은 `CTERelationDef`
  목록 + main query.
- **`CTERelationDef id, resolved`**: CTE의 정의 subtree. `id`는 Analyzer가
  부여한 고유 식별자. `resolved` flag는 내부 resolution 상태.
- **`CTERelationRef id, resolved, [cols], recursive`**: CTE의 참조. main
  query 안에서 CTE 이름이 등장한 지점을 대체. 같은 `id`로 `CTERelationDef`
  를 가리킴.

**입력** (Analyzed LogicalPlan):

```
WithCTE
:- CTERelationDef <id>, <resolved>
:  +- <CTE body subtree>
+- <main query subtree containing
     CTERelationRef <same id>, <resolved>, [col#id, ...], <recursive>>
```

**출력** (Optimized LogicalPlan, inline 적용 시):

```
<main query subtree with CTERelationRef replaced by the CTE body>
```

`WithCTE`, `CTERelationDef`, `CTERelationRef` 세 노드 모두 사라지고, 참조
지점에 body가 그대로 substitute된다. 참조가 여럿이었다면 각 지점에
별개의 복제본이 들어간다.

**쿼리 C 관찰** (Analyzed, 라인 27-40):

```
WithCTE
:- CTERelationDef 0, false
:  +- SubqueryAlias monthly_avg
:     +- Aggregate [agent_gu#377, deal_year#370, deal_month#371],
:          [..., avg(deal_amount#373) AS avg_amount#364,
:                count(1) AS trade_count#365L]
:        +- Filter agent_gu#377 LIKE 서울%
:           +- SubqueryAlias nessie.real_estate.apt_trades
:              +- RelationV2[...] nessie.real_estate.apt_trades ...
+- Sort [agent_gu#377 ASC NULLS FIRST, ...]
   +- Project [..., moving_avg_6m#363]
      +- Project [..., moving_avg_6m#363, moving_avg_6m#363]
         +- Window [avg(avg_amount#364) ...]
            +- Project [agent_gu#377, deal_year#370, deal_month#371,
                        avg_amount#364, trade_count#365L]
               +- SubqueryAlias monthly_avg
                  +- CTERelationRef 0, true,
                       [agent_gu#377, deal_year#370, deal_month#371,
                        avg_amount#364, trade_count#365L], false
```

Optimized (라인 46-50) — 세 노드 모두 사라짐:

```
Sort [agent_gu#377 ASC NULLS FIRST, ...]
+- Window [avg(avg_amount#364) ...]
   +- Aggregate [agent_gu#377, deal_year#370, deal_month#371],
        [..., avg(deal_amount#373) AS avg_amount#364,
              count(1) AS trade_count#365L]
      +- Filter (isnotnull(agent_gu#377) AND StartsWith(agent_gu#377, 서울))
         +- RelationV2[deal_year#370, deal_month#371, deal_amount#373,
                       agent_gu#377] nessie.real_estate.apt_trades
```

- 참조 횟수 = 1회 (main query에 `CTERelationRef 0` 한 번만 등장) → inline
  안전.
- Inline 후 `Aggregate` 노드가 main query의 `Window` 바로 아래에 배치됨.
- Column pruning이 경계를 넘어 작동 → Aggregate가 쓰지 않는 컬럼들까지
  RelationV2 스키마에서 제거 (17 → 4 컬럼).

## L3. Key Mechanisms

> 📝 docs 요약 수준. 소스 코드 ingest 대기 중
> (`sql/catalyst/src/main/scala/org/apache/spark/sql/catalyst/optimizer/InlineCTE.scala`,
> `sql/catalyst/.../analysis/CTESubstitution.scala`,
> `Optimizer.scala`의 해당 rule batch).

Plan 관찰과 공개 docs 기반 잠정 기술 (소스 ingest 전):
- `CTESubstitution` rule (analysis 단계 또는 이른 optimizer 단계)이 SQL
  parser가 만든 `UnresolvedWith` / `UnresolvedRelation`(CTE 이름)을
  `WithCTE` + `CTERelationDef` + `CTERelationRef` 구조로 정규화.
- `InlineCTE` rule (논리 최적화 단계)이 inline 가능성을 검사:
  - 참조 횟수 (단일 참조는 거의 항상 inline).
  - CTE body의 cost (deterministic 여부, side-effect, 재계산 비용).
  - `spark.sql.legacy.*` 호환 플래그.
- Inline 불가능한 경우 `CTERelationDef`/`Ref`가 Optimized plan에 그대로
  남고, 물리 plan이 이를 shuffle reuse 또는 InMemoryRelation으로 실현.

정확한 규칙 이름, inline 결정 조건, recursive CTE 처리는 소스 ingest 후
기록.

## L4. Performance Levers

> 📝 docs 요약 수준. 소스 코드 ingest 대기 중.

잠정 나열:
- `spark.sql.optimizer.excludedRules`로 `InlineCTE` 관련 규칙을 끄면 inline
  강제 중단 가능. 운영 환경에서는 거의 건드리지 않음.
- CTE가 여러 번 참조되고 각 참조 위에 서로 다른 filter가 붙는 경우:
  inline하면 각 복제본마다 최적화가 달라질 수 있어 유리할 수도, 불리할
  수도 있음.
- Recursive CTE (`WITH RECURSIVE`)는 별도 처리 — Spark 3.5는 제한적 지원.

## L5. How to Observe

- `df.queryExecution.analyzed.toString` 또는 `df.explain(mode="extended")`
  → Analyzed Logical Plan 섹션에서 `WithCTE`, `CTERelationDef id, ...`,
  `CTERelationRef id, ..., [cols], recursive` 세 노드 확인.
- `df.queryExecution.optimizedPlan.toString` → 같은 세 노드가 모두 사라지고
  CTE body가 main query에 병합되었는지 확인.
- 세 노드가 **남아 있다면** inline 미적용 — 참조 횟수, 복잡도, deterministic
  여부를 확인.
- Physical plan에서 CTE가 어떻게 실현됐는지: inline되지 않은 경우
  `ReusedExchange` 또는 `InMemoryRelation` 노드 등장 여부 관찰.

> 📝 probe 페이지 미작성. 후속 `/probe 03-logical-optimization/cte-inline`
> 대상 — "1회 참조 vs 다중 참조 CTE의 plan 차이 비교".
