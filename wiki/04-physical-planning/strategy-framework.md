---
stage: 04-physical-planning
title: Strategy framework — 논문 candidates 모델 vs 실제 ladder
spark_versions: ["3.5"]
sources:
  - raw/spark-source/v3.5.6/sql/catalyst/src/main/scala/org/apache/spark/sql/catalyst/planning/QueryPlanner.scala
  - raw/spark-source/v3.5.6/sql/core/src/main/scala/org/apache/spark/sql/execution/SparkPlanner.scala
  - raw/spark-source/v3.5.6/sql/core/src/main/scala/org/apache/spark/sql/execution/SparkStrategies.scala
references:
  - "Armbrust, M. et al. (2015). Spark SQL: Relational Data Processing in Spark. SIGMOD §4.3 — Physical Planning."
last_verified: 2026-04-29
confidence: verified
related:
  - 00-overview/execution-pipeline.md
  - 04-physical-planning/join-strategy-hints.md
  - 05-cost-and-aqe/aqe-overview.md
  - 05-cost-and-aqe/leveraging-statistics.md
  - _probes/explain-modes.md
configs:
  - spark.sql.shuffle.partitions
  - spark.sql.cbo.enabled
  - spark.sql.adaptive.enabled
apis:
  - df.queryExecution.executedPlan
  - df.queryExecution.optimizedPlan
  - df.explain
tags: [physical-planning, query-planner, strategies, paper-vs-impl]
---

## L1. What & Why

Physical planning 단계는 Optimized Logical Plan을 SparkPlan tree로 변환한다.
Armbrust 2015 논문 §4.3은 "각 logical 노드에 대해 strategy들이 다수의 후보
SparkPlan을 만들고, planner가 cost를 비교해 1개를 고른다"는 모델을 제시한다.
Spark 3.5.6의 인터페이스(`GenericStrategy.apply: Seq[PhysicalPlan]`,
`QueryPlanner.plan: Iterator[PhysicalPlan]`)는 이 모델의 형태를 보존하지만,
**실제 strategy 구현은 cost를 비교하지 않는다** — 각 strategy는 휴리스틱
ladder를 거쳐 단일 plan만 반환하고 (또는 비어있는 Seq), `QueryPlanner.plan()`은
이들을 그저 `flatMap`으로 평탄화해 Iterator로 내놓는다. `SparkPlanner.prunePlans`도
입력을 그대로 통과시킨다. 진짜 cost 기반 결정은 세 갈래로 흩어져 있다.

## L2. Inputs & Outputs

**입력**: Optimized LogicalPlan 한 그루.

**출력**: SparkPlan 한 그루 (`QueryExecution`이 `Iterator`의 첫 element만 사용).

인터페이스 시그니처와 실제 행동이 어긋나는 세 지점:

| 위치 | 공식 시그니처 | 실제 동작 |
| --- | --- | --- |
| `GenericStrategy.apply` (QueryPlanner.scala:38) | `Seq[PhysicalPlan]` | 매칭되지 않으면 빈 Seq, 매칭되면 길이 1짜리 단일 Seq (4건 예시: SparkStrategies.scala:245-252, 262-270, 275-276, 286) |
| `QueryPlanner.plan` (QueryPlanner.scala:59-95) | `Iterator[PhysicalPlan]` | candidates를 `flatMap`으로 평탄화만; cost 비교 코드 없음 (line 63) |
| `SparkPlanner.prunePlans` (SparkPlanner.scala:62-66) | `Iterator → Iterator` | 입력 그대로 통과 (TODO 주석으로 직접 인정) |

## L3. Key Mechanisms

### 3.1 GenericStrategy 인터페이스

`GenericStrategy`는 logical 노드 매칭과 SparkPlan 생성을 묶은 추상화다.
v3.5.6의 시그니처 (`QueryPlanner.scala:29-39`):

```scala
abstract class GenericStrategy[PhysicalPlan <: TreeNode[PhysicalPlan]] extends Logging {
  protected def planLater(plan: LogicalPlan): PhysicalPlan
  def apply(plan: LogicalPlan): Seq[PhysicalPlan]
}
```

핵심 두 메서드:

- `apply` — logical 노드를 받아 0개 이상의 physical 후보를 반환. **반환 타입이
  `Seq`라는 점이 인터페이스 약속**: 다수 후보 가능성을 열어둠.
- `planLater` — 자식 logical 노드를 후속 처리에 위임하는 placeholder를 만든다.
  strategy는 자기 책임 노드만 처리하고, 자식들은 `PlanLater(child)` 형태의
  플레이스홀더로 비워둔다.

위 docstring(`QueryPlanner.scala:24-28`)도 "returns a list of `PhysicalPlan`s
that can be used for execution"이라고 표현하고 있다. 즉 **인터페이스 텍스트는
candidates 모델을 그대로 반영**한다.

### 3.2 QueryPlanner.plan() 흐름

`QueryPlanner` 추상 클래스의 docstring(`QueryPlanner.scala:41-54`)에는 결정적
TODO가 박혀 있다:

> 📌 "TODO: RIGHT NOW ONLY ONE PLAN IS RETURNED EVER...
>     PLAN SPACE EXPLORATION WILL BE IMPLEMENTED LATER."
> — `QueryPlanner.scala:50-51` (Spark 3.5.6)

대문자 강조까지 동원한 이 문장이 본 페이지의 가장 강한 증거다. 인터페이스의
형태는 candidates를 약속하지만, **실제로는 단 하나의 plan만 흘러간다**.

본문 흐름 (`QueryPlanner.scala:59-95`):

```scala
def plan(plan: LogicalPlan): Iterator[PhysicalPlan] = {
  // Collect physical plan candidates.
  val candidates = strategies.iterator.flatMap(_(plan))           // line 63

  // The candidates may contain placeholders marked as [[planLater]],
  // so try to replace them by their child plans.
  val plans = candidates.flatMap { candidate =>
    val placeholders = collectPlaceholders(candidate)
    if (placeholders.isEmpty) {
      Iterator(candidate)
    } else {
      placeholders.iterator.foldLeft(Iterator(candidate)) {
        case (candidatesWithPlaceholders, (placeholder, logicalPlan)) =>
          val childPlans = this.plan(logicalPlan)                  // 재귀
          candidatesWithPlaceholders.flatMap { candidateWithPlaceholders =>
            childPlans.map { childPlan =>
              candidateWithPlaceholders.transformUp {
                case p if p.eq(placeholder) => childPlan
              }
            }
          }
      }
    }
  }

  val pruned = prunePlans(plans)                                   // line 92
  assert(pruned.hasNext, s"No plan for $plan")                     // line 93
  pruned                                                           // line 94
}
```

세 가지 사실:

1. **line 63**: `strategies.iterator.flatMap(_(plan))`은 모든 strategy의
   apply 결과를 평탄화한다. 만약 두 strategy가 같은 logical 노드에 대해
   각각 한 후보를 반환한다면 candidates는 길이 2짜리 Iterator가 될 것이다 —
   인터페이스상 가능. 그러나 이 함수 어디에도 candidates를 비교·정렬·선별하는
   코드는 없다. 그저 `flatMap`으로 모은다.
2. **line 67-90**: placeholder 채우기는 `transformUp`으로 in-place 치환할 뿐
   "어떤 자식 plan이 더 싼가"를 묻지 않는다.
3. **line 94**: `prunePlans`의 결과를 `Iterator`로 그대로 반환. 호출자
   (`QueryExecution.sparkPlan`)는 보통 `.next()`를 한 번만 호출해 첫 후보를
   가져간다. 즉 strategy 등록 순서와 각 strategy의 첫 후보가 사실상 결정.

### 3.3 SparkStrategies의 단일 Seq 패턴

`JoinSelection` (사이클 1에서 verified) 한 곳만 봐도 strategy 구현이 단일
plan만 반환하는 패턴이 명확히 드러난다 (`SparkStrategies.scala:216`):

```scala
def apply(plan: LogicalPlan): Seq[SparkPlan] = plan match {
  case j @ ExtractEquiJoinKeys(...) =>
    def createBroadcastHashJoin(...) = {
      ...
      buildSide.map { buildSide =>
        Seq(joins.BroadcastHashJoinExec(...))                       // :245-252
      }
    }
    def createShuffleHashJoin(...) = {
      ...
      buildSide.map { buildSide =>
        Seq(joins.ShuffledHashJoinExec(...))                        // :262-270
      }
    }
    def createSortMergeJoin() = {
      if (RowOrdering.isOrderable(leftKeys)) {
        Some(Seq(joins.SortMergeJoinExec(...)))                     // :275-276
      } else None
    }
    def createCartesianProduct() = {
      if (joinType.isInstanceOf[InnerLike] && ...) {
        Some(Seq(joins.CartesianProductExec(...)))                  // :286
      } else None
    }
    ...
}
```

네 곳 모두 `Seq(SingleOperator)` 패턴 — **multi-element Seq를 만들지 않는다**.
이어지는 `createJoinWithoutHint`는 candidates를 enumerate하는 대신 `.orElse`
사슬로 ladder를 구성한다 (`SparkStrategies.scala:292-306`):

```scala
def createJoinWithoutHint() = {
  createBroadcastHashJoin(false)
    .orElse(createShuffleHashJoin(false))
    .orElse(createSortMergeJoin())
    .orElse(createCartesianProduct())
    .getOrElse {
      // This join could be very slow or OOM
      ...
      Seq(joins.BroadcastNestedLoopJoinExec(...))                    // :303-304
    }
}
```

**`Option.orElse`는 첫 `Some`을 만나면 멈춘다**. 즉 후보들을 모아서 비교하는
구조가 아니라, 우선순위 순으로 시도해 첫 통과 옵션을 채택하는 구조다. 이것이
이 Wiki에서 "ladder"라고 부르는 패턴이다.

코드 주석(`SparkStrategies.scala:218-236`)도 "we follow these rules **one by
one**"이라고 명시한다 — 동시 비교가 아니라 순차 통과.

### 3.4 SparkPlanner의 strategy 등록 순서

`SparkPlanner`는 `QueryPlanner`의 구체 구현이며, 등록할 strategy 목록과 순서를
정한다 (`SparkPlanner.scala:33-48`):

```scala
override def strategies: Seq[Strategy] =
  experimentalMethods.extraStrategies ++
    extraPlanningStrategies ++ (
    LogicalQueryStageStrategy ::
    PythonEvals ::
    new DataSourceV2Strategy(session) ::
    FileSourceStrategy ::
    DataSourceStrategy ::
    SpecialLimits ::
    Aggregation ::
    Window ::
    WindowGroupLimit ::
    JoinSelection ::
    InMemoryScans ::
    SparkScripts ::
    BasicOperators :: Nil)
```

순서 자체가 의미를 갖는다:

1. **외부 hook 우선** — `experimentalMethods.extraStrategies`(라이브러리/사용자
   런타임 등록)과 `extraPlanningStrategies`(서브클래스 override hook,
   `SparkPlanner.scala:54`)가 모든 내장보다 앞.
2. **AQE/소스 커넥터** — `LogicalQueryStageStrategy`는 AQE가 이미 stage로 변환한
   자식 노드 처리. 이어서 V2/File/V1 소스 커넥터.
3. **연산 종류별 strategy** — `SpecialLimits`, `Aggregation`, `Window`,
   `WindowGroupLimit`, `JoinSelection`, `InMemoryScans`, `SparkScripts`.
4. **catch-all** — `BasicOperators`는 Project, Filter 등 일반 연산자 처리. 마지막에
   둠으로써 더 구체적인 strategy가 먼저 매칭될 기회를 갖는다.

`QueryPlanner.plan()`의 `flatMap(_(plan))`(line 63)과 결합하면, **각 logical
노드에 대해 위 순서로 `apply()`를 호출하고, 빈 Seq가 아닌 첫 결과의 첫 후보가
사실상 채택된다**. 다음 strategy의 결과는 Iterator에 남아 있을 뿐 소비되지 않는다.

`numPartitions`(`SparkPlanner.scala:31`)는 `conf.numShufflePartitions` —
즉 `spark.sql.shuffle.partitions`를 노출. exchange 삽입 시 기본 파티션 수로 사용.

### 3.5 prunePlans는 비활성화되어 있다

`QueryPlanner.plan()`이 마지막에 호출하는 `prunePlans`(`QueryPlanner.scala:92,
104`)는 추상이다. 자식 클래스가 plan space를 좁힐 hook. `SparkPlanner`의 구현
(`SparkPlanner.scala:62-66`):

> 📌 "TODO: We will need to prune bad plans when we improve plan space exploration
>     to prevent combinatorial explosion."
> — `SparkPlanner.scala:62-66` (Spark 3.5.6)

```scala
override protected def prunePlans(plans: Iterator[SparkPlan]): Iterator[SparkPlan] = {
  // TODO: We will need to prune bad plans when we improve plan space exploration
  //       to prevent combinatorial explosion.
  plans
}
```

함수 본문이 `plans` 한 줄. **즉 prune할 코드가 없다**. TODO 주석은 "plan space
exploration이 개선되면 prune이 필요해질 것"이라고 적고 있는데, 이는 다음
사실의 다른 표현이다:

> 현재 SparkStrategies는 multi-candidate를 만들지 않으므로 prune할 대상도 없다.

3.2의 TODO ("ONLY ONE PLAN IS RETURNED EVER")와 3.5의 TODO ("when we improve
plan space exploration")는 같은 사실의 두 면 — Spark 코어 팀 스스로 v3.5.6
시점에서 candidates 모델이 "구현 미완"임을 코드 안에 기록해 두었다.

### 3.6 그렇다면 cost는 어디서 들어오는가

논문 모델의 "cost 비교"는 사라진 것이 아니라 다른 단계로 분산되었다.
세 갈래:

1. **Strategy 내부의 통계 휴리스틱.** `JoinSelection`의 `getBroadcastBuildSide`,
   `getShuffleHashJoinBuildSide`(SparkStrategies.scala:240, 257), 그리고
   `canBroadcastBySize`/`canBuildLocalHashMap` 같은 헬퍼는 logical plan의
   `Statistics`(byte size 추정)를 보고 단일 결정을 내린다 — 후보를 만들어
   비교하는 것이 아니라, 통계를 본 뒤 이 strategy가 적용될지/build side가
   어디인지를 휴리스틱으로 정한다. 결정 자체는 strategy 내부에서 종료된다.
2. **Logical CBO** (`spark.sql.cbo.enabled`, 기본 false). physical 단계 이전,
   `Optimizer`의 `CostBasedJoinReorder` 등이 통계와 cardinality 추정을 사용해
   logical plan의 join 순서를 바꾼다. 즉 "어떤 plan 모양이 나오는가"는 바뀌지만,
   physical strategy 단계 자체에는 cost 비교가 없다. (후속 ingest 후보 →
   `03-logical-optimization/cbo-join-reorder.md`)
3. **AQE 사후 재선택** (`spark.sql.adaptive.enabled`, 기본 true since 3.2).
   stage 경계에서 실제 shuffle 통계를 본 뒤 broadcast로 강등하거나 skew를
   분리한다 (`OptimizeSkewedJoin`, `DynamicJoinSelection`). **이것이
   "candidates를 비교해 더 좋은 것을 고른다"는 논문 모델에 가장 근접한 동작**
   이지만, 그것은 단계 04(physical planning)가 아니라 단계 05(cost-and-aqe)
   에서 발생한다. 자세한 내용은 `05-cost-and-aqe/aqe-overview.md` 참고.

요약: **단계 04의 strategy framework 안에는 cost 비교 모델이 없다**.
인터페이스가 그것을 허용할 뿐, 그 자리는 비어 있고, 실제 cost 결정은
strategy 내부 휴리스틱(국소) / logical CBO(외부 앞) / AQE(외부 뒤)로 흩어졌다.

## L4. Performance Levers

이 단계의 동작에 영향을 주는 lever는 strategy 등록 hook과, "어디서 cost가
들어오는가"의 세 갈래에 대응하는 설정이다:

- **`spark.sql.shuffle.partitions`** (기본 200). `SparkPlanner.numPartitions`
  (`SparkPlanner.scala:31`)를 통해 strategy들이 exchange 삽입 시 사용하는 기본
  파티션 수. AQE coalesce가 켜져 있으면 사후 조정 — 그러나 strategy 단계의
  initial 파티션 수는 이 설정으로 결정.
- **`spark.sql.cbo.enabled`** (기본 false). 켜져 있으면 단계 03의
  `CostBasedJoinReorder`가 통계 기반으로 join 순서를 변경. 단계 04 strategy
  ladder가 보는 logical plan의 모양 자체가 달라진다.
- **`spark.sql.adaptive.enabled`** (기본 true since 3.2). 켜져 있으면 단계 04의
  결과가 그대로 실행되지 않고, `AdaptiveSparkPlanExec`가 wrapping해 stage
  경계에서 사후 재선택을 수행.
- **`experimentalMethods.extraStrategies`** — `SparkSession`의 hook. Iceberg,
  Delta, Hudi 같은 라이브러리가 자체 strategy를 등록하는 통로
  (`SparkPlanner.scala:34`). 모든 내장보다 앞에 시도되므로 매칭만 되면 우선권.
- **`extraPlanningStrategies` override** (`SparkPlanner.scala:54`).
  `SparkPlanner`를 직접 상속하는 서브클래스(예: `HiveSessionStateBuilder`의
  planner)가 strategy를 끼워넣는 hook. 기본 Nil.

후속 ingest 후보 (verified로 끌어올리려면 raw 코드가 필요한 영역):

| 후보 | 이유 |
| --- | --- |
| `CostBasedJoinReorder` (단계 03) | logical CBO의 실제 cost 함수가 어떻게 정의되는지 — `Statistics`/`EstimationUtils` 인용 필요 |
| AQE `OptimizeSkewedJoin` / `DynamicJoinSelection` (단계 05) | 사후 재선택의 진짜 cost 비교 위치 |
| `LogicalQueryStageStrategy` | strategies 리스트 첫 자리에 등록되는 AQE bridge — 단계 04와 단계 05의 경계 |
| `JoinSelectionHelper` | `getBroadcastBuildSide` 등 strategy 내부 휴리스틱의 본체 |

## L5. How to Observe

이 단계는 logical plan과 physical plan의 *경계*를 만드는 단계이므로, 두 plan을
나란히 출력하면 strategy ladder의 결과를 직접 확인할 수 있다.

### 5.1 optimized vs executed plan 비교

```scala
val df = spark.range(1000).join(spark.range(1000), "id")
println(df.queryExecution.optimizedPlan.numberedTreeString)
println(df.queryExecution.sparkPlan.numberedTreeString)       // strategy 적용 직후
println(df.queryExecution.executedPlan.numberedTreeString)    // EnsureRequirements 등 후처리 + AQE wrap 포함
```

`optimizedPlan`에는 `Join` (LogicalPlan), `sparkPlan`에는
`BroadcastHashJoinExec`/`SortMergeJoinExec` 등 — 즉 **이 차이가 strategy
ladder가 한 일**이다. `sparkPlan`과 `executedPlan`의 차이는 EnsureRequirements
규칙이 삽입한 `ShuffleExchangeExec` + AQE wrapping(`AdaptiveSparkPlanExec`).

### 5.2 EXPLAIN 모드별 가시성

```scala
df.explain("formatted")  // physical plan tree만, 노드별 인덱스
df.explain("extended")   // Parsed → Analyzed → Optimized → Physical 4단계
df.explain("cost")       // Statistics 추정 노출 (CBO가 보는 값)
```

`extended`로 보면 단계 04의 입력(Optimized)과 출력(Physical) 사이의 전환이
명확하다. 자세한 EXPLAIN 모드 비교는 `_probes/explain-modes.md` 참고
(probe 페이지 작성 예정 — 현재 placeholder).

### 5.3 AQE on/off로 strategy ladder 분리

```scala
// strategy ladder 결과만 보고 싶을 때
spark.conf.set("spark.sql.adaptive.enabled", "false")
val rawPhysical = df.queryExecution.executedPlan
// 이때는 AdaptiveSparkPlanExec wrapping이 없고, JoinSelection이 고른 operator가 그대로 root

// AQE 사후 재선택 효과를 보려면
spark.conf.set("spark.sql.adaptive.enabled", "true")
df.collect()  // 실행해야 stage 경계에서 재선택이 일어남
val adaptivePhysical = df.queryExecution.executedPlan
// AdaptiveSparkPlanExec(initialPlan=..., currentPhysicalPlan=...) 형태
```

`AdaptiveSparkPlanExec`의 `initialPlan`이 단계 04 strategy ladder의 결과,
`currentPhysicalPlan`이 단계 05 AQE 재선택의 결과. 두 값이 다르면 cost 결정이
사후로 미뤄졌다는 직접 증거다.

### 5.4 strategy 등록 순서 자체 확인

```scala
val planner = spark.sessionState.planner   // SparkPlanner
planner.strategies.foreach(s => println(s.getClass.getSimpleName))
```

기대 출력 (extraStrategies/extraPlanningStrategies가 없을 때):

```
LogicalQueryStageStrategy
PythonEvals
DataSourceV2Strategy
FileSourceStrategy
DataSourceStrategy
SpecialLimits
Aggregation
Window
WindowGroupLimit
JoinSelection
InMemoryScans
SparkScripts
BasicOperators
```

`SparkPlanner.scala:33-48`의 정의와 정확히 일치해야 한다. Iceberg/Delta가
설치된 환경에서는 앞쪽에 추가 strategy가 끼어든 것을 볼 수 있다.

### 5.5 무엇을 봐야 하는가

5.1-5.4의 4가지 도구로 단계 03 / 04 / 05의 plan을 분리해 본다.
`AdaptiveSparkPlanExec`의 initialPlan과 currentPhysicalPlan 차이가 클수록
cost 결정이 사후로 미뤄진 쿼리다 — 이 페이지 결론의 직접 측정.
