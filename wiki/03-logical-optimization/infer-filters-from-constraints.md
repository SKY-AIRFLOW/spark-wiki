---
stage: 03-logical-optimization
title: Infer Filters From Constraints
spark_versions: ["3.5"]
sources:
  - raw/explain/2026-04-19-query-A-self-join-growth.txt
  - raw/spark-source/v3.5.6/sql/catalyst/src/main/scala/org/apache/spark/sql/catalyst/optimizer/Optimizer.scala
last_verified: 2026-04-26
confidence: verified
related:
  - 03-logical-optimization/predicate-pushdown.md
  - 04-physical-planning/join-strategy-hints.md
  - case/2026-04-19-query-A-self-join-growth.md
configs:
  - spark.sql.constraintPropagation.enabled
  - spark.sql.optimizer.excludedRules
apis:
  - SQL JOIN ON
  - SQL WHERE
  - SQL HAVING
tags: [optimizer, catalyst, constraint, propagation, inference]
---

## L1. What & Why

`InferFiltersFromConstraints`는 plan 트리 자식 노드에서 도출 가능한
**constraints** (`col IS NOT NULL`, equi-join key 동등성 등)를 부모
`Filter`/`Join` 노드에 추가 filter로 inference하는 규칙이다. 의미상 자명한
filter를 명시적으로 plan에 적는 것이며, 이렇게 명시화된 filter가 곧바로
`PushDownPredicates`의 입력이 되어 더 깊은 곳으로 push 가능해진다.
case A의 양쪽 child에 `legal_dong IS NOT NULL`이 자동 추가된 것이
정확히 이 메커니즘 — equi-join key는 양쪽 모두 not-null이라야 매치되므로,
한쪽 제약이 다른 쪽으로 transitive inference된다.

## L2. Inputs & Outputs

입력: `Filter(condition, child)` 또는 `Join(left, right, joinType, condition)`.
출력: 동일 노드이지만 `child.constraints -- (child.constraints ++
existing-predicates)` 만큼 새 filter가 conjunction으로 추가된 상태.

## L3. Key Mechanisms

### object 시그니처

`raw/spark-source/v3.5.6/.../optimizer/Optimizer.scala:1397-1398`:

```scala
object InferFiltersFromConstraints extends Rule[LogicalPlan]
  with PredicateHelper with ConstraintHelper {
```

### conf 게이트

`Optimizer.scala:1400-1406`:

```scala
def apply(plan: LogicalPlan): LogicalPlan = {
  if (conf.constraintPropagationEnabled) inferFilters(plan) else plan
}
```

설정명: `spark.sql.constraintPropagation.enabled` (default `true`). `false`면
no-op.

### inferFilters Filter 케이스

`Optimizer.scala:1408-1417`:

```scala
case filter @ Filter(condition, child) =>
  val newFilters = filter.constraints --
    (child.constraints ++ splitConjunctivePredicates(condition))
  if (newFilters.nonEmpty) {
    Filter(And(newFilters.reduce(And), condition), child)
  } else {
    filter
  }
```

차집합: `filter.constraints`(도출된 모든 제약) - `child.constraints`(이미
보유) - `splitConjunctivePredicates(condition)`(이미 명시된 predicate) =
"추가해야 할 새 filter". 비어 있으면 무변동. Join 케이스 분기
(`Optimizer.scala:1419-`)는 양쪽 child constraints + join condition 사이
transitive 관계 처리 — 본 페이지 범위 밖.

### batch 위치 — Once

`Optimizer.scala:146-148`:

```scala
Batch("Infer Filters", Once,
  InferFiltersFromGenerate,
  InferFiltersFromConstraints) ::
```

이 batch는 "Operator Optimization before Inferring Filters" (fixedPoint)와
"Operator Optimization after Inferring Filters" (fixedPoint) 두 batch 사이에
끼워져 있다 — inference된 새 filter가 다음 fixedPoint batch의
`PushDownPredicates` 입력이 되는 구조. 동반 `InferFiltersFromGenerate`는
`Generate` 노드 (lateral view 등) 전용 별도 규칙, 후속 ingest 후보.

### case A 적용 사례

case A의 join condition `curr.legal_dong = prev.legal_dong`에서 양쪽
attribute가 equi-join 참여 → 매치되려면 둘 다 not-null이 자명. 이 규칙이
양쪽으로 명시화 → 다음 batch의 `PushDownPredicates`가 받아 양쪽 BatchScan
filters까지 전달 (case A line 58, 76, 97의 `isnotnull(legal_dong)`).
[predicate-pushdown.md](predicate-pushdown.md) L2 case A 표 참조.

## L4. Performance Levers

`spark.sql.constraintPropagation.enabled` (default `true`) — `false`로 끄면
join 양쪽의 implicit not-null 정보가 명시화되지 않아 BatchScan filters에
not-null이 빠짐. data source가 null row까지 모두 읽고 후속 단계에서 join이
자체적으로 떨어뜨리게 됨 — read I/O 증가.

> 📝 Join 케이스 분기 (`Optimizer.scala:1419-`) 본문, `InferFiltersFromGenerate`
> 협업, 비활성화 시 실측 성능 영향은 후속 ingest 보강.

## L5. How to Observe

`df.queryExecution.optimizedPlan`에서 자동 추가된 `isnotnull(...)` 등 filter
관찰. case A의 BatchScan filters에 `legal_dong IS NOT NULL`이 등장하지만
원래 SQL에는 없는 조건 — 이 규칙이 만든 것.

> 📝 비교 실험: `spark.conf.set("spark.sql.constraintPropagation.enabled", "false")`
> 후 case A 재실행 → optimizedPlan과 BatchScan filters에서
> `isnotnull(legal_dong)` 사라짐 직접 확인. 후속
> `/probe 03-logical-optimization/infer-filters-from-constraints` 후보.
