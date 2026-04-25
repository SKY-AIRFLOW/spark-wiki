---
stage: 04-physical-planning
title: EnsureRequirements
spark_versions: ["3.5"]
sources:
  - raw/spark-source/v3.5.6/sql/core/src/main/scala/org/apache/spark/sql/execution/exchange/EnsureRequirements.scala
  - raw/spark-source/v3.5.6/sql/catalyst/src/main/scala/org/apache/spark/sql/catalyst/plans/physical/partitioning.scala
last_verified: 2026-04-26
confidence: verified
related:
  - 04-physical-planning/partitioning-compatibility.md
  - 04-physical-planning/join-strategy-hints.md
  - 04-physical-planning/hash-aggregate-partial-final.md
  - 04-physical-planning/window-exec.md
  - 04-physical-planning/storage-partition-join.md
  - case/2026-04-19-query-A-self-join-growth.md
  - case/2026-04-19-query-B-row-number-top5.md
  - case/2026-04-19-query-C-window-moving-avg.md
configs:
  - spark.sql.shuffle.partitions
  - spark.sql.maxSinglePartitionBytes
  - spark.sql.requireAllClusterKeysForDistribution
  - spark.sql.sources.v2.bucketing.enabled
  - spark.sql.sources.v2.bucketing.pushPartValues.enabled
  - spark.sql.sources.v2.bucketing.partiallyClusteredDistribution.enabled
apis:
  - DataFrame.groupBy
  - DataFrame.join
  - Window.partitionBy
  - DataFrame.orderBy
tags: [physical-planning, exchange, distribution, partitioning, shuffle]
---

# EnsureRequirements

## L1. What & Why

`EnsureRequirements`는 `Rule[SparkPlan]`이다. `SparkStrategies`가 logical
plan을 physical plan으로 변환하고 난 직후의 plan은 각 노드가 자신의
`requiredChildDistribution: Seq[Distribution]`과
`requiredChildOrdering: Seq[Seq[SortOrder]]`을 **선언**만 한 상태이며,
`Exchange`/`Sort` 노드는 아직 plan tree에 들어 있지 않다.

`EnsureRequirements`는 이 plan tree를 한 번 더 순회하면서, 각 노드의 자식이
가진 `outputPartitioning`이 부모가 요구하는 `Distribution`을 만족하는지
검사한다. 만족하지 않으면 `ShuffleExchangeExec` 또는 `BroadcastExchangeExec`를
삽입한다. 같은 방식으로 `outputOrdering`이 부족하면 `SortExec`도 삽입한다.

이 rule이 없으면 — 예를 들어 `SortMergeJoin`은 좌/우 자식이 같은 키로
hash partition되어 있고 정렬되어 있다는 **불변식**을 가정하지만, 그것을
보장하는 노드는 plan 어디에도 없게 된다. `EnsureRequirements`가 이 gap을
메우는 단계다.

> 인용: `raw/spark-source/v3.5.6/sql/core/src/main/scala/org/apache/spark/sql/execution/exchange/EnsureRequirements.scala:57-67` (rule 정의 + `ensureDistributionAndOrdering` 시그니처).

## L2. Inputs & Outputs

**입력:** `SparkPlan` (이미 `SparkStrategies` 변환 후, **Exchange/Sort
미삽입** 상태). 각 physical operator는 다음 세 필드를 노출한다:

- `outputPartitioning: Partitioning` — 이 노드가 출력하는 데이터의 분배 형태.
- `requiredChildDistribution: Seq[Distribution]` — 이 노드가 자식들에게 요구하는 분배.
- `requiredChildOrdering: Seq[Seq[SortOrder]]` — 이 노드가 자식들에게 요구하는 정렬.

**출력:** `SparkPlan` with `ShuffleExchangeExec` / `BroadcastExchangeExec` /
`SortExec` 노드가 필요한 위치에 삽입된 상태.

## L3. Key Mechanisms

핵심 메서드는 `ensureDistributionAndOrdering`(line 62-218). 안에 4개의
메커니즘이 차례로 적용된다.

### (1) 단일 자식 distribution check (line 71-80)

```scala
var children = originalChildren.zip(requiredChildDistributions).map {
  case (child, distribution) if child.outputPartitioning.satisfies(distribution) =>
    child                                                          // ← 그대로 통과
  case (child, BroadcastDistribution(mode)) =>
    BroadcastExchangeExec(mode, child)                              // ← BroadcastExchange
  case (child, distribution) =>
    val numPartitions = distribution.requiredNumPartitions
      .getOrElse(conf.numShufflePartitions)
    ShuffleExchangeExec(distribution.createPartitioning(numPartitions),
                        child, shuffleOrigin)                       // ← ShuffleExchange
}
```

세 case 파일에 등장한 7개 Exchange 중 **5개가 이 9줄에서 발생**한다 (case A
2개는 multi-child 경로; (2) 참조). `satisfies` 호출의 의미와 false 반환
조건은 [partitioning-compatibility.md](partitioning-compatibility.md)에서
별도로 다룬다 — 본 페이지는 호출 흐름까지만 보여준다.

`distribution.createPartitioning(numPartitions)`이 default partition 수의
원천이며, 그 값은 `conf.numShufflePartitions` (즉 `spark.sql.shuffle.partitions`,
default 200)에서 온다. `Distribution`이 `requiredNumPartitions`를 `Some(_)`로
지정하면 그 값이 우선 사용된다 (예: `BroadcastDistribution.requiredNumPartitions = Some(1)`).

### (2) Multi-child co-partitioning (line 99-205)

자식 둘 이상이 모두 `ClusteredDistribution`을 요구하는 경우 (대표적으로
`SortMergeJoin`, `ShuffledHashJoin`). 양쪽을 무조건 shuffle하지 않고, 한쪽
spec을 채택해 다른 쪽만 re-shuffle하는 것이 목표다.

핵심 흐름:

- 각 자식의 `outputPartitioning.createShuffleSpec(...)`을 만들고 (line 104-105),
  `canCreatePartitioning`과 `isCompatibleWith` 체크를 통해 한쪽 spec을 채택해
  다른 쪽을 re-shuffle한다 (line 138-160).
- 동수 partition을 추구한다. 코드 주석이 직접 설명:

  > "in scenarios such as: `HashPartitioning(5) <-> HashPartitioning(6)` while
  > `spark.sql.shuffle.partitions` is 10, we'll only re-shuffle the left side
  > and make it `HashPartitioning(6)`." — `EnsureRequirements.scala:130-133`

- 양쪽 모두 `ShuffleExchangeLike`이거나 partitioning을 자체 생성하지 못하면
  `spark.sql.shuffle.partitions` 기준 minimum parallelism이 적용된다
  (line 134-141).
- **SPJ 경로** — 두 자식이 `KeyGroupedPartitioning`이고 호환되면
  `checkKeyGroupCompatible`(line 167-177, 354-521)을 통해 양쪽 shuffle을 모두
  회피할 수 있다. 본 페이지는 진입점만 인용; 깊은 다이브는
  [storage-partition-join.md](storage-partition-join.md) (강화 예정).

쿼리 A의 SMJ 양쪽 Exchange가 이 경로의 결과다 — 양쪽 자식 모두 partitioning을
자체 생성하지 못하는 상태에서 시작했기 때문에 한쪽 spec을 채택할 후보가
없었고, 양쪽 모두 `ShuffleExchange`가 삽입되었다.

### (3) Ordering check (line 207-215)

```scala
children = children.zip(requiredChildOrderings).map { case (child, requiredOrdering) =>
  if (SortOrder.orderingSatisfies(child.outputOrdering, requiredOrdering)) {
    child
  } else {
    SortExec(requiredOrdering, global = false, child = child)
  }
}
```

`SortMergeJoin`과 `WindowExec`이 자식에 정렬을 요구한다. 이 분기에서
`SortExec(global = false)` (즉 partition-local sort)가 삽입된다. partition
경계를 넘는 global sort는 (1) 단계의 `ShuffleExchange(rangepartitioning, ...)`로
이미 처리되어 있으므로 여기서는 partition 내 정렬만 보장하면 된다.

### (4) Join key reordering (line 220-346)

**보너스 메커니즘.** `SMJ`/`SHJ`에서 자식의 `outputPartitioning` expression
순서가 join key 순서와 다를 때, 양쪽 partition을 새로 shuffle하지 않고
**join key 자체를 재정렬**해서 일치시킨다.

진입점 사다리:

- `reorderJoinPredicates`(line 329-346) — `SortMergeJoinExec`,
  `ShuffledHashJoinExec` 노드 매칭 후 `reorderJoinKeys` 호출.
- `reorderJoinKeys`(line 264-281) — `deterministic` 검사 후
  `reorderJoinKeysRecursively` 위임.
- `reorderJoinKeysRecursively`(line 285-326) — 좌/우 partitioning을
  재귀적으로 풀어가며 best match를 시도. `PartitioningCollection`의
  partitioning 후보들도 처리.
- `reorder`(line 220-262) — 실제 키 재정렬 알고리즘. 일치 불가 시 `None`
  반환.

이 메커니즘은 [join-strategy-hints.md](join-strategy-hints.md)가 다루는
SMJ 선택의 다음 단계다 — strategy가 SMJ를 선택했더라도 `outputPartitioning`과
join key 순서가 미스매치면 추가 shuffle이 발생할 수 있는데, `reorderJoinKeys`가
사전에 회피한다. 사용자가 toggle할 수 있는 config은 없다 (자동 동작).

### Rule 진입점 (참고)

`case class EnsureRequirements(optimizeOutRepartition: Boolean = true,
requiredDistribution: Option[Distribution] = None) extends Rule[SparkPlan]`
(line 57-60). `apply` 메서드가 plan tree를 traverse하면서 각 노드별로
`ensureDistributionAndOrdering`을 호출한다. plan tree traversal 진입점
자체는 본 ingest 사이클의 범위 외 — Gaps 섹션 참조.

## L4. Performance Levers

| Lever | 영향 | 인용 |
|---|---|---|
| `spark.sql.shuffle.partitions` (default 200) | satisfies 실패 시 삽입되는 `ShuffleExchangeExec`의 default partition 수 | `EnsureRequirements.scala:78` |
| `spark.sql.requireAllClusterKeysForDistribution` (default false) | `ClusteredDistribution` 생성 시 capture되어 `satisfies0` 분기 결정 — 상세는 [partitioning-compatibility.md](partitioning-compatibility.md) | `partitioning.scala:92-93` |
| `spark.sql.maxSinglePartitionBytes` | 양쪽 자식이 `SinglePartition`이고 데이터 크기가 이 값 이하면 multi-child shuffle 회피 (`preferSinglePartition`) | `EnsureRequirements.scala:91-95` |
| `spark.sql.sources.v2.bucketing.enabled` 외 2종 | SPJ 경로 게이트 — `KeyGroupedPartitioning` 호환성 검사 | `EnsureRequirements.scala:391, 441` |
| `reorderJoinKeys` 자동 동작 | SMJ/SHJ에서 partition expr 순서가 join key와 어긋나도 추가 shuffle 회피 | `EnsureRequirements.scala:220-346` |

`reorderJoinKeys`는 자동 동작이라 사용자가 직접 켜고 끌 수 없다. 그러나
그 존재를 아는 것은 [join-strategy-hints.md](join-strategy-hints.md)의
SMJ 선택을 이해하는 데 필수다: SMJ는 양쪽 자식 모두 정확한 키 순서로
partition + sort되어 있어야 하지만, `reorderJoinKeys` 덕분에 partition
expression의 순서가 약간 어긋나는 경우는 추가 shuffle 없이 처리된다.

## L5. How to Observe

세 case 파일의 7개 Exchange가 모두 이 페이지의 메커니즘으로 설명된다.

| Case | 위치 | 메커니즘 |
|---|---|---|
| A self-join growth | `Exchange hashpartitioning(legal_dong, 200)` × 2 (SMJ 양쪽) | (2) multi-child co-partitioning, 양쪽 ShuffleExchange |
| B row_number top-5 | `Exchange hashpartitioning(agent_gu, 200)` (window 입력) | (1) 단일 자식 distribution check |
| B row_number top-5 | `Exchange rangepartitioning(agent_gu ASC, price_rank ASC, 200)` (outer ORDER BY) | (1) 단일 자식 distribution check, `OrderedDistribution` → `RangePartitioning` |
| C window moving avg | `Exchange hashpartitioning(agent_gu, deal_year, deal_month, 200)` (HashAggregate 입력) | (1) 단일 자식 distribution check |
| C window moving avg | `Exchange hashpartitioning(agent_gu, 200)` (HashAggregate Final → Window) | (1) 단일 자식 distribution check, **superset 케이스** — [partitioning-compatibility.md](partitioning-compatibility.md) |
| C window moving avg | `Exchange rangepartitioning(agent_gu ASC, deal_year ASC, deal_month ASC, 200)` (outer ORDER BY) | (1) 단일 자식 distribution check |

관찰 방법:

- `df.queryExecution.executedPlan.toString` 출력에서
  `Exchange hashpartitioning(...)` / `Exchange rangepartitioning(...)` /
  `Exchange SinglePartition` 노드를 카운트한다. 각 Exchange 직전·직후 노드의
  `outputPartitioning`과 `requiredChildDistribution`을 비교하면 어느
  메커니즘인지 식별 가능.
- AQE가 활성화된 상태에서는 `isFinalPlan=false` 시점의 Exchange와
  `isFinalPlan=true` 시점이 다를 수 있다. case A/B/C는 모두
  `isFinalPlan=false` 캡처 — runtime 재작성 전 plan을 본 것.
- `ENSURE_REQUIREMENTS` 라벨이 붙은 Exchange는 **사용자가 명시한 repartition
  hint가 아니라 이 rule이 자동 삽입한 것**임을 표시한다 (case A/B/C 모두 해당).
- partition expression 순서가 join key와 자연스럽게 일치해 보이는 이유는
  `reorderJoinKeys` 때문일 수 있다 — raw plan에서는 미스매치였을 수도 있음.
- 상세 probe는 [_probes/explain-modes.md] (예정).

## Version notes

- v3.5.6 commit `303c18c74664f161b9b969ac343784c088b47593` 기준.
- v4.x diff는 후속 ingest 대상. 특히 `ensureDistributionAndOrdering`
  시그니처와 SPJ 경로의 `checkKeyGroupCompatible`은 4.x에서 변경 가능성이
  높은 영역.
- `case class EnsureRequirements`의
  `requiredDistribution: Option[Distribution]` 파라미터(line 59)는 root
  node의 distribution 요구를 외부에서 강제하는 hook이다. 이 ingest에서는
  다루지 않음.

## Gaps

- `EnsureRequirements.apply` 메서드 자체 — plan tree traversal 진입점은
  본 ingest 범위 외.
- `ShuffleSpec` / `isCompatibleWith` 메커니즘 깊은 ingest — 두 자식 spec
  비교 로직 (line 100-160).
- `checkKeyGroupCompatible`(line 354-521) → SPJ partial clustering 깊은
  ingest는 [storage-partition-join.md](storage-partition-join.md) 강화와
  병행.
