---
stage: 03-logical-optimization
title: Predicate Pushdown
spark_versions: ["3.5"]
sources:
  - raw/explain/2026-04-19-query-A-self-join-growth.txt
  - raw/explain/2026-04-19-query-B-row-number-top5.txt
  - raw/explain/2026-04-19-query-C-window-moving-avg.txt
  - raw/spark-source/v3.5.6/sql/catalyst/src/main/scala/org/apache/spark/sql/catalyst/optimizer/Optimizer.scala
  - raw/spark-source/v3.5.6/sql/core/src/main/scala/org/apache/spark/sql/execution/SparkOptimizer.scala
last_verified: 2026-04-26
confidence: verified
related:
  - 00-overview/execution-pipeline.md
  - 03-logical-optimization/column-pruning.md
  - 03-logical-optimization/cte-inline.md
  - 03-logical-optimization/infer-filters-from-constraints.md
  - 03-logical-optimization/like-simplification.md
  - 04-physical-planning/join-strategy-hints.md
  - case/2026-04-19-query-A-self-join-growth.md
  - case/2026-04-19-query-B-row-number-top5.md
  - case/2026-04-19-query-C-window-moving-avg.md
configs:
  - spark.sql.optimizer.excludedRules
  - spark.sql.optimizer.dynamicPartitionPruning.enabled
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

`PushDownPredicates`라는 이름은 단일 알고리즘이 아니라 **세 PartialFunction
의 dispatcher**다. `Optimizer.scala:1706` 정의에서 `CombineFilters`,
`PushPredicateThroughNonJoin`, `PushPredicateThroughJoin`을 `orElse`로
체이닝한다. 한 트리 순회 중 각 노드마다 매치하는 첫 PartialFunction이
적용되며, 이는 각 노드가 한 패스에서 하나의 변환만 받는 보장을 준다.

## L2. Inputs & Outputs

**입력**: Analyzed `LogicalPlan`의 `Filter(condition, child)` 노드
또는 `Join(left, right, joinType, condition)` 노드. child 타입 또는
join 종류에 따라 dispatcher 분기가 결정된다:
- `Join`: equi-join 조건 외 조건들을 각 side로 분배 (`PushPredicateThroughJoin`)
- `Aggregate`, `Project`, `Window`, `Generate`, `Union` 등: PartialFunction
  매치별 처리 (`PushPredicateThroughNonJoin`)
- `Relation` / `RelationV2`: source API (`SupportsPushDownFilters`,
  `SupportsPushDownV2Filters`)가 수락하는 조건은 relation의 메타데이터에
  부착되어 scan 시 사용 (단계 04 경계의 `V2ScanRelationPushDown`, 별도
  ingest 대상)

**출력**: 같은 의미의 `LogicalPlan`이지만 `Filter`가 트리의 더 깊은
위치로 이동한 상태. 일부 조건은 `Filter` 노드에서 사라지고 `RelationV2`
의 `pushedFilters` (또는 BatchScan의 `filters=[...]`) 리스트로 이동.

### 쿼리 A 관찰 (Optimized Plan)

원래 WHERE `((curr.deal_year=2024) AND (prev.deal_year=2023)) AND
curr.agent_gu LIKE '서울%'` → Join 위 하나의 `Filter`. 최적화 후 양쪽
child 아래로 분배:

| Join 양쪽 | Optimizer가 도달한 Filter 조건 |
|---|---|
| curr (left) | `isnotnull(deal_year) AND isnotnull(agent_gu) AND deal_year=2024 AND StartsWith(agent_gu, 서울) AND isnotnull(legal_dong)` |
| prev (right) | `isnotnull(deal_year) AND deal_year=2023 AND isnotnull(legal_dong)` |

이 분배는 정확히 `PushPredicateThroughJoin.split()` (`Optimizer.scala:1917-1925`)
가 condition을 3-튜플 `(canEvaluateInLeft, canEvaluateInRight, haveToEvaluateInBoth)`
로 분류한 결과다. `legal_dong IS NOT NULL`이 양쪽 모두에 등장한 것은
**별도 규칙** `InferFiltersFromConstraints` (`Optimizer.scala:1397`,
[03/infer-filters-from-constraints.md](infer-filters-from-constraints.md))이
직전 batch에서 equi-join key의 not-null 제약을 양쪽으로 inference한
결과를 PushDown이 받은 것이다. 두 규칙의 batch 순서가 메커니즘의 핵심.

나아가 Physical의 `BatchScan`에 `filters=[deal_year IS NOT NULL,
agent_gu IS NOT NULL, deal_year=2024, agent_gu LIKE '서울%',
legal_dong IS NOT NULL, groupedBy=]`로 전체가 data source까지 전달.
이는 단계 04 경계의 `V2ScanRelationPushDown` 동작 결과로, plan 내부
filter는 `StartsWith(agent_gu, 서울)`로 변환됐지만 BatchScan filters는
원형 `LIKE '서울%'`을 유지 — Iceberg V2 connector가 별도 push-down
경로에서 원본 LIKE를 받기 때문.

이 관찰은 "filter가 Optimized plan에서 사라졌다"가 아니라 "filter
조건이 dispatcher에 의해 분배되고 일부는 scan node의 메타데이터로 통합"이
정확한 기술임을 상기시킨다. case B와 C에서도 같은 패턴이 단일 BatchScan
경로로 관찰된다.

## L3. Key Mechanisms

### Dispatcher 구조

`PushDownPredicates`는 단일 변환이 아니라 세 PartialFunction을 `orElse`
체인으로 묶은 dispatcher다.

`raw/spark-source/v3.5.6/.../optimizer/Optimizer.scala:1706`:

```scala
object PushDownPredicates extends Rule[LogicalPlan] {
  def apply(plan: LogicalPlan): LogicalPlan = plan.transformWithPruning(
    _.containsAnyPattern(FILTER, JOIN)) {
    CombineFilters.applyLocally
      .orElse(PushPredicateThroughNonJoin.applyLocally)
      .orElse(PushPredicateThroughJoin.applyLocally)
  }
}
```

각 PartialFunction의 분담:

| PartialFunction | 담당 | 정의 위치 |
|---|---|---|
| `CombineFilters` | 인접 `Filter` 노드 합치기 (`Filter(c1, Filter(c2, x))` → `Filter(c1 ∧ c2, x)`) | 별도 파일, 후속 ingest |
| `PushPredicateThroughNonJoin` | Project/Aggregate/Window/Generate/Union 등 비-Join 노드 통과 | `Optimizer.scala:1722` |
| `PushPredicateThroughJoin` | Join 양쪽으로 condition 분배 | `Optimizer.scala:1908` |

`transformWithPruning(_.containsAnyPattern(FILTER, JOIN))`이 트리 순회를
가속한다 — `FILTER` 또는 `JOIN` 노드가 없는 subtree는 통째로 skip
(TreePattern bitset 기반).

### PushPredicateThroughNonJoin의 첫 매치 케이스

`Optimizer.scala:1722-1734`:

```scala
object PushPredicateThroughNonJoin extends Rule[LogicalPlan] with PredicateHelper {
  def apply(plan: LogicalPlan): LogicalPlan = plan transform applyLocally

  val applyLocally: PartialFunction[LogicalPlan, LogicalPlan] = {
    // SPARK-13473: We can't push the predicate down when the underlying projection output non-
    // deterministic field(s).  Non-deterministic expressions are essentially stateful. ...
    case Filter(condition, project @ Project(fields, grandChild))
      if fields.forall(_.deterministic) && canPushThroughCondition(grandChild, condition) =>
      val aliasMap = getAliasMap(project)
      ...
```

핵심 가드는 `fields.forall(_.deterministic)` — Project의 모든 출력이
결정론적이어야 push 허용. `rand()`, `current_timestamp()` 등 비결정
표현식이 있는 Project 위의 Filter는 push 불가 (input row 순서가
표현식 결과에 영향을 주기 때문). 이 가드는 Aggregate, Window 분기에도
같은 형태로 등장한다.

### PushPredicateThroughJoin의 split()

`Optimizer.scala:1917-1925`:

```scala
private def split(condition: Seq[Expression], left: LogicalPlan, right: LogicalPlan) = {
  val (pushDownCandidates, nonDeterministic) = condition.partition(_.deterministic)
  val (leftEvaluateCondition, rest) =
    pushDownCandidates.partition(_.references.subsetOf(left.outputSet))
  val (rightEvaluateCondition, commonCondition) =
      rest.partition(expr => expr.references.subsetOf(right.outputSet))
  (leftEvaluateCondition, rightEvaluateCondition, commonCondition ++ nonDeterministic)
}
```

조건 분류 알고리즘 (4단계):
1. `partition(_.deterministic)`로 결정/비결정 분리. 비결정은 어디로도 push
   불가 — Join 자리에 그대로 남는다 (`commonCondition`에 합류).
2. 결정론적 후보를 left output set 부분집합 검사로 left-evaluable 분리.
3. 남은 것을 right output set 부분집합 검사로 right-evaluable 분리.
4. 어디에도 안 가는 것 (`commonCondition`)은 양쪽 컬럼을 모두 참조 — Join
   조건으로 유지.

쿼리 A의 `curr.deal_year = 2024`는 left output set 부분집합 → left로
분배. `prev.deal_year = 2023`은 right로. `curr.agent_gu LIKE '서울%'`
(이미 `StartsWith`로 변환된 후) 도 left로. `legal_dong IS NOT NULL`
(양쪽 모두 `InferFiltersFromConstraints`가 추가)은 각 child의 condition
으로 분배 — 결과는 위 L2 표.

### canPushThrough(joinType) 가드

`Optimizer.scala:1927-1930`:

```scala
private def canPushThrough(joinType: JoinType): Boolean = joinType match {
  case _: InnerLike | LeftSemi | RightOuter | LeftOuter | LeftAnti | ExistenceJoin(_) => true
  case _ => false
}
```

`FullOuter` 등은 push 불가 (양쪽 child의 null-safety가 깨질 수 있음).
case A의 self-join은 inner — 통과.

### 동일 규칙의 이중 wiring (batch 위치)

`PushDownPredicates`는 두 위치에서 wiring된다.

**`Optimizer.scala`의 `defaultBatches`** (Catalyst 코어):
- `Batch("Operator Optimization before Inferring Filters", fixedPoint, ...)`
  (local 144) 내 `operatorOptimizationRuleSet`에 포함 (local 92).
- `Batch("Operator Optimization after Inferring Filters", fixedPoint, ...)`
  (local 149) 에서 같은 rule set 재실행 (Infer Filters 후).
- `Batch("Push extra predicate through join", fixedPoint, ...)` (local 151)
  에서 `PushExtraPredicateThroughJoin`과 함께 또 한 번 실행.

**`SparkOptimizer.scala`** (Spark 엔진 마무리):
- `Batch("Pushdown Filters from PartitionPruning", fixedPoint, PushDownPredicates)`
  (local 79-80) — PartitionPruning이 만든 새 Filter를 다시 push.

**의미**: 같은 규칙이 fixed-point batch 내에서도 반복되고, 다른 batch
에서도 재실행된다. 따라서 "PushDownPredicates는 한 번 적용된다"는
표현은 부정확. `InjectRuntimeFilter`, `PartitionPruning` 등이 새 Filter를
만든 후 다시 push down되는 것까지가 전체 동작이다.

## L4. Performance Levers

### Catalyst Optimizer.scala vs SparkOptimizer.scala 비교

이 wiki에서 가장 중요한 단계 03→04 경계 구분.

| 측면 | `Optimizer.scala` (catalyst) | `SparkOptimizer.scala` (sql/core) |
|---|---|---|
| 추상화 | `abstract class Optimizer` | `class SparkOptimizer extends Optimizer` |
| 패키지 | `o.a.s.sql.catalyst.optimizer` | `o.a.s.sql.execution` |
| 역할 | 데이터 소스/엔진 무관 (Catalyst 코어) | Spark 엔진 + Python UDF + V2 push-down 결합 |
| `defaultBatches` 수 | 약 22 | catalyst의 22개 + 12 (= `super.defaultBatches ++ ...`) |
| `PushDownPredicates` wiring | "Operator Optimization" 두 번 + "Push extra predicate through join" | + "Pushdown Filters from PartitionPruning" 재실행 |
| `ColumnPruning` wiring | "Operator Optimization" | + "Extract Python UDFs" batch 재실행 |
| `InferWindowGroupLimit` wiring | ❌ 없음 | "Infer window group limit" batch (`SparkOptimizer.scala:96-101`) |
| `MergeScalarSubqueries` wiring | ❌ 없음 (Catalyst-순수) | "MergeScalarSubqueries" batch |
| `InjectRuntimeFilter` wiring | ❌ 없음 | "InjectRuntimeFilter" batch |
| 데이터 소스 push-down | ❌ (V2ScanRelationPushDown은 SparkOptimizer 책임) | `earlyScanPushDownRules` (`SparkOptimizer.scala:45-53`) |

이 분리는 **"Catalyst가 데이터 소스 무관 / SparkOptimizer가 엔진별
마무리"** 의 결정적 증거. data source v2의 push-down (`V2ScanRelationPushDown`)
은 SparkOptimizer의 `earlyScanPushDownRules`에서 실행되는 별도 단계로,
이 페이지 범위 밖이다 (후속 ingest 후보). 다른 03 단계 페이지가 이
표를 참조한다.

### Lever 목록

- **`spark.sql.optimizer.excludedRules`** — 특정 규칙(`PushDownPredicates`,
  `PushPredicateThroughJoin` 등) 이름을 콤마 구분으로 나열해 비활성화.
  운영 환경에서는 거의 변경하지 않음 — 디버깅 또는 회귀 격리용.
- **`spark.sql.optimizer.dynamicPartitionPruning.enabled`** (default `true`)
  — DPP가 만든 runtime filter가 SparkOptimizer의 "Pushdown Filters from
  PartitionPruning" batch 진입의 입력. `false`면 이 이중 wiring의 한쪽이
  의미가 없어진다.
- **결정성 (deterministic)** — `PushPredicateThroughNonJoin`
  (`Optimizer.scala:1722`) 의 `fields.forall(_.deterministic)` 가드. 사용자
  함수가 비결정이면 push 차단 → 그 위 Filter가 그대로 남아 fewer pruning.
  UDF 작성 시 `nondeterministic = false` (default) 유지가 lever.
- **Data source 지원 여부** — Iceberg V2는 LIKE/StartsWith/IsNull 등 광범위
  지원, JDBC는 타입별 제한, 일부 file format은 동등 비교만 지원.
  지원하지 않는 predicate은 plan에 남고 scan 직후 필터링됨 (correctness OK,
  performance 손실).

### Equi-join key의 transitive 효과

`PushDownPredicates` 자체는 condition을 단순 분배하지만, 직전 batch
"Infer Filters" (`Optimizer.scala:146`)에서 `InferFiltersFromConstraints`가
equi-join key의 transitivity를 활용해 양쪽으로 새 Filter를 추가한다 — 그
새 Filter가 곧바로 PushDown 입력이 된다. 이 두 규칙의 협업이 case A의
"`legal_dong IS NOT NULL`이 양쪽 child에 모두 등장"의 정확한 메커니즘.
[03/infer-filters-from-constraints.md](infer-filters-from-constraints.md)
참조.

## L5. How to Observe

- `df.queryExecution.optimizedPlan` — Optimized LogicalPlan 출력. `Filter`
  노드의 위치가 Analyzed와 비교해 어디로 이동했는지 관찰. 어느 dispatcher
  분기가 적용됐는지는 형태로 추론:
  - `Filter` 아래에 `Project`가 그대로 있고 condition 변화 없음 → NonJoin이
    비결정 가드에서 차단됨
  - `Filter`가 `Join` 양쪽 child 아래로 분배 → `PushPredicateThroughJoin`
    이 split() 적용
  - 인접한 두 `Filter`가 하나로 합쳐짐 → `CombineFilters`
- `df.explain(mode="formatted")` — Physical plan의 scan 노드 섹션에
  `PushedFilters` 또는 Iceberg의 `filters=[...]` 리스트 확인. data source
  경계 push-down 여부.
- EXPLAIN COST로 filter 전후 `Statistics(sizeInBytes=...)` 감소 관찰. 단
  cardinality 추정이 부정확할 수 있음 — 쿼리 A의 join fallback 413 TiB
  사례 참고.
- 비활성화 실험: `spark.conf.set("spark.sql.optimizer.excludedRules",
  "org.apache.spark.sql.catalyst.optimizer.PushDownPredicates")` 후 같은
  쿼리 재실행 → Filter 위치 차이 직접 비교. 좋은 probe 실험.

> 📝 probe 페이지 미작성. 후속 `/probe 03-logical-optimization` 대상 —
> "PushDownPredicates 비활성화 시 case A의 Filter 위치 비교".
