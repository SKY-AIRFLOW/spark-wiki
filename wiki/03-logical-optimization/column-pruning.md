---
stage: 03-logical-optimization
title: Column Pruning
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
  - 03-logical-optimization/predicate-pushdown.md
  - 03-logical-optimization/cte-inline.md
  - 04-physical-planning/batch-scan-iceberg.md
  - case/2026-04-19-query-A-self-join-growth.md
  - case/2026-04-19-query-B-row-number-top5.md
  - case/2026-04-19-query-C-window-moving-avg.md
configs:
  - spark.sql.optimizer.excludedRules
apis:
  - DataFrame.select
  - SQL SELECT projection
tags: [optimizer, catalyst, projection, pruning]
---

## L1. What & Why

`ColumnPruning`은 plan 트리를 위에서 아래로 훑어 **부모가 참조하지 않는
attribute**를 자식 노드의 projection/aggregate/window 표현식에서 제거하고,
그 효과를 leaf relation까지 전파하는 규칙이다. 결과적으로 RelationV2/Scan
노드의 출력 컬럼 목록이 좁아져 read amplification이 감소한다.

이 최적화의 가치는 컬럼 지향 (columnar) 저장 포맷에서 가장 크다 — Iceberg/
Parquet/ORC는 read 시 필요한 컬럼만 디스크에서 가져올 수 있기 때문에,
plan에서 컬럼 수가 줄면 그대로 I/O 감소로 이어진다. 이 wiki의 case A/B/C
모두 같은 17 컬럼 테이블 `apt_trades`에서 시작해 각각 5/6/4 컬럼으로
좁혀진다.

규칙 정의는 `Optimizer.scala:874` (Catalyst 코어). Catalyst defaultBatches의
"Operator Optimization" rule set에 포함되어 fixed-point batch 안에서
다른 규칙들 (PushDownPredicates, CollapseProject 등)과 함께 반복 적용된다.

## L2. Inputs & Outputs

**입력**: Analyzed `LogicalPlan` 또는 partially optimized plan. Project,
Aggregate, Expand, Generate, Window, Union, WithCTE 등 attribute를
producing/consuming하는 노드들이 대상.

**출력**: 같은 의미의 plan이지만 다음이 좁아진 상태:
- `Project`의 `projectList`에서 부모가 참조하지 않는 attribute 제거
- `Aggregate`의 `aggregateExpressions`에서 미참조 항목 제거
- `Window`의 `windowExpressions`에서 미참조 window 함수 제거
- 자식이 producing하는 출력 중 부모가 안 쓰는 것 위에 명시적
  `Project(filteredOutput)` 삽입 (`prunedChild`)
- 결과적으로 leaf `RelationV2`/`HiveTableRelation` 등의 출력 컬럼 목록
  축소

### case별 관찰 — 17 컬럼 테이블에서의 축소

| case | 쿼리 성격 | RelationV2 입력 컬럼 | pruning 후 | 사용 컬럼 |
|---|---|---|---|---|
| A | self-join + WHERE | 17 | 5 (× curr/prev) | `apt_seq, deal_year, deal_amount, legal_dong, agent_gu` |
| B | ROW_NUMBER + WHERE | 17 | 6 | `apt_name, deal_year, deal_amount, area_sqm, agent_gu, deal_date` |
| C | CTE + Window AVG | 17 | 4 | `deal_year, deal_month, deal_amount, agent_gu` |

case C의 4 컬럼 축소는 CTE inline 직후의 효과 — `WithCTE`/`Def`/`Ref`가
사라져 ColumnPruning이 outer query의 참조와 inner Aggregate/Filter/Relation
사이를 직접 가로질러 동작한 결과다. CTE inline과 ColumnPruning의 협업이
이 압축의 메커니즘.

## L3. Key Mechanisms

### object 시그니처와 호출 구조

`Optimizer.scala:874-876`:

```scala
object ColumnPruning extends Rule[LogicalPlan] {

  def apply(plan: LogicalPlan): LogicalPlan = removeProjectBeforeFilter(
    plan.transformWithPruning(AlwaysProcess.fn, ruleId) {
      ...
    })
```

두 단계: (1) `transformWithPruning`으로 PartialFunction 매치 + 변환, (2)
결과 plan을 `removeProjectBeforeFilter`로 후처리. 후자는 이 규칙이 임시로
삽입한 redundant `Project`를 정리한다 (자세한 내용 후술). `AlwaysProcess.fn`
은 TreePattern 가지치기를 끄고 모든 노드를 처리 — column pruning은 plan
전체에 영향을 주는 규칙이라 가지치기가 부적합.

### PartialFunction 케이스 — 7개 그룹

`Optimizer.scala:879-978`에 16개 정도의 매치 케이스. 역할별로 묶으면:

| 그룹 | 라인 | 역할 |
|---|---|---|
| Project on Project | 879-880 | 자식 Project의 `projectList`를 부모 references로 필터 |
| Project on Aggregate | 881-883 | `aggregateExpressions`를 references로 필터 |
| Project on Expand | 884-891 | Expand의 `output` + `projections` 양쪽 좁히기 |
| Project on Window | 951-954 | 미참조 window 표현식 제거, 모두 미사용이면 Window 자체 우회 |
| Project on Union | 934-948 | 첫 child 기준으로 좁힌 후 모든 children에 동일 projection 강제 |
| Project on WithCTE / SetOperation / Distinct / LeafNode | 931-932, 957-965 | 의미 보존상 prune 불가, 그대로 통과 |
| 자식이 producing 노드일 때 (Aggregate/Expand/Generate/MergeRows/DeserializeToObject) | 899-921 | `prunedChild`로 child 출력 위에 명시적 Project 삽입 |

핵심 헬퍼 `prunedChild` (`Optimizer.scala:982-987`):

```scala
private def prunedChild(c: LogicalPlan, allReferences: AttributeSet) =
  if (!c.outputSet.subsetOf(allReferences)) {
    Project(c.output.filter(allReferences.contains), c)
  } else {
    c
  }
```

자식의 모든 output이 부모 references에 이미 포함되면 no-op, 그렇지 않으면
필요한 attribute만 추리는 `Project`로 감싼다. 이 `Project`는 후처리에서
종종 다시 제거된다 (`removeProjectBeforeFilter`).

### removeProjectBeforeFilter — 후처리

`Optimizer.scala:994-` (apply의 두 번째 단계):

```scala
private def removeProjectBeforeFilter(plan: LogicalPlan): LogicalPlan = plan transformUp {
  case p1 @ Project(_, f @ Filter(e, p2 @ Project(_, child)))
    if p2.outputSet.subsetOf(child.outputSet) &&
      // We only remove attribute-only project.
      ...
```

`ColumnPruning`이 top-down으로 삽입한 redundant Project (`Project → Filter
→ Project → child` 패턴) 중 attribute-only Project (표현식 없이 컬럼
선택만)를 bottom-up으로 제거한다. 이유는 코드 주석:

> The Project before Filter is not necessary but conflict with
> PushPredicatesThroughProject, so remove it.

즉 `PushPredicateThroughNonJoin` (predicate-pushdown 규칙)이 Project를
가로질러 Filter를 push할 때 이 임시 Project가 방해되므로 미리 정리.
이 두 규칙이 같은 fixed-point batch에 있으므로 매 iteration마다 협업.

### 이중 wiring — Catalyst와 SparkOptimizer 양쪽

Catalyst 측 (`Optimizer.scala:88-92` defaultBatches operator rule set):

```scala
val operatorOptimizationRuleSet = Seq(
  PushProjectionThroughUnion,
  PushProjectionThroughLimit,
  ...
  ColumnPruning,
  ...
)
```

이 rule set이 `Batch("Operator Optimization before Inferring Filters",
fixedPoint, ...)` (local 144)와 `Batch("Operator Optimization after
Inferring Filters", fixedPoint, ...)` (local 149) 두 batch에서 사용된다 —
`InferFiltersFromConstraints` 전후로 두 번 fixed-point.

SparkOptimizer 측 (`SparkOptimizer.scala:99`):

```scala
Batch("Extract Python UDFs", Once,
  ...
  // The eval-python node may be between Project/Filter and the scan node, which breaks
  // column pruning and filter push-down. Here we rerun the related optimizer rules.
  ColumnPruning,
  LimitPushDown,
  PushPredicateThroughNonJoin,
  ...) :+
```

Python UDF가 plan에 eval-python 노드를 끼워 넣으면 이미 적용된 column
pruning이 깨질 수 있어, 이 batch에서 다시 한 번 적용. Catalyst vs
SparkOptimizer 비교 표는
[predicate-pushdown.md L4](predicate-pushdown.md#l4-performance-levers)
참조.

## L4. Performance Levers

- **`spark.sql.optimizer.excludedRules`** — `org.apache.spark.sql.catalyst.optimizer.ColumnPruning`
  을 제외하면 plan 전체가 RelationV2의 모든 컬럼을 끝까지 들고 다닌다.
  Iceberg/Parquet 환경에서 read 비용이 즉시 폭발 — 디버깅 외에는 절대
  비활성화하지 않는다.
- **DataFrame `.drop()` vs `.select()`** — explicit projection은 plan에
  직접 `Project` 노드로 등장해 ColumnPruning의 입력이 된다. `.drop()`
  도 동등한 `Project`로 바뀌며, ColumnPruning은 둘을 구분하지 않는다.
  반대로 `df.toDF()`처럼 컬럼을 명시적으로 안 좁히는 호출은 plan에 모든
  attribute를 남기지만, 부모가 안 쓰면 결국 ColumnPruning이 같은 결과로
  좁힌다 — implicit pruning.
- **Nested column pruning은 별도** — struct/array 안의 일부 field만
  쓰는 경우는 `SchemaPruning` rule (`SparkOptimizer.scala:45-53`의
  `earlyScanPushDownRules`)이 담당. 이 규칙은 V2 data source에 대해
  scan 시점에 nested field-level pruning을 적용한다. 이 페이지의
  `ColumnPruning`은 top-level attribute만 다룬다.
- **`NestedColumnAliasing`** (`Optimizer.scala:967`)이 ColumnPruning
  내부에서 별도 case로 호출되어 일부 nested 케이스를 처리하지만, 풀
  nested pruning은 SchemaPruning 영역. 후속 ingest 후보.

> 📝 V2 SchemaPruning과 ColumnPruning의 정확한 협업 (어느 시점에
> 어느 규칙이 무엇을 pruning하는지) — 후속 ingest 시 보강.

## L5. How to Observe

- `df.queryExecution.optimizedPlan` — Optimized LogicalPlan 출력. 가장
  확실한 검증은 leaf `RelationV2[...]` 노드의 컬럼 리스트를 보는 것:
  ```
  RelationV2[deal_year#370, deal_month#371, deal_amount#373, agent_gu#377]
    nessie.real_estate.apt_trades
  ```
  case C의 이 4컬럼 출력이 17→4 축소의 직접 증거.
- `df.explain(mode="formatted")` — Physical plan 섹션의 각 노드별
  `Output: [...]` + scan 노드의 `ReadSchema: struct<...>`. ReadSchema는
  Iceberg/Parquet에 실제로 요청되는 스키마 — 컬럼 수 = pruning 후 컬럼 수.
- 비활성화 실험: `spark.conf.set("spark.sql.optimizer.excludedRules",
  "org.apache.spark.sql.catalyst.optimizer.ColumnPruning")` 후 같은 쿼리
  재실행 → RelationV2의 컬럼 수가 원본 17개 그대로 유지되는지 확인.
  실행 시간/I/O 차이도 함께 측정 가능.
- case별 컬럼 수 검증 (raw explain에서):
  - case A line 77: `BatchScan ...[apt_seq#188, deal_year#190, deal_amount#193, legal_dong#196, agent_gu#197]` → 5 컬럼
  - case B line 61: `[apt_name#297, deal_year#301, deal_amount#304, area_sqm#305, agent_gu#308, deal_date#313]` → 6 컬럼
  - case C line 62: `[deal_year#370, deal_month#371, deal_amount#373, agent_gu#377]` → 4 컬럼

> 📝 probe 페이지 미작성. 후속 `/probe 03-logical-optimization/column-pruning`
> 후보 — "ColumnPruning 비활성화 시 ReadSchema 차이 + Iceberg actual
> bytes-read 비교".
