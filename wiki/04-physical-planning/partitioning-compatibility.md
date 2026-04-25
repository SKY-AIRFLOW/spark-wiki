---
stage: 04-physical-planning
title: Partitioning compatibility (Distribution × Partitioning matrix)
spark_versions: ["3.5"]
sources:
  - raw/spark-source/v3.5.6/sql/catalyst/src/main/scala/org/apache/spark/sql/catalyst/plans/physical/partitioning.scala
  - raw/spark-source/v3.5.6/sql/core/src/main/scala/org/apache/spark/sql/execution/exchange/EnsureRequirements.scala
last_verified: 2026-04-26
confidence: verified
related:
  - 04-physical-planning/ensure-requirements.md
  - 04-physical-planning/hash-aggregate-partial-final.md
  - 04-physical-planning/window-exec.md
  - 04-physical-planning/storage-partition-join.md
  - 04-physical-planning/coalesce-repartition-hints.md
  - case/2026-04-19-query-C-window-moving-avg.md
  - case/2026-04-19-query-B-row-number-top5.md
  - case/2026-04-19-query-A-self-join-growth.md
configs:
  - spark.sql.requireAllClusterKeysForDistribution
apis:
  - DataFrame.groupBy
  - Window.partitionBy
  - DataFrame.orderBy
  - DataFrame.join
tags: [physical-planning, partitioning, distribution, satisfies, shuffle]
---

# Partitioning compatibility (Distribution × Partitioning matrix)

## L1. What & Why

Spark의 모든 데이터 재분배(shuffle) 결정은 단 한 호출로 환원된다 —
`Partitioning.satisfies(Distribution)`. true이면 자식의 현재 분배를 그대로
사용 가능, false이면 [EnsureRequirements](ensure-requirements.md)가
`ShuffleExchangeExec`를 삽입한다.

이 페이지는 v3.5.6의 모든 `Partitioning` 구현체와 모든 `Distribution`
구현체의 호환성을 한 매트릭스로 정리한다. 각 셀의 판정 로직은
`partitioning.scala`의 정확한 라인에서 온다.

> 인용: `raw/spark-source/v3.5.6/sql/catalyst/.../partitioning.scala:218-220` (`Partitioning.satisfies` 진입점), `:241-245` (기본 `satisfies0`).

## L2. Inputs & Outputs

**입력:** `(Partitioning, Distribution)` 페어. 예: 자식 노드의
`outputPartitioning = HashPartitioning([a, b, c], 200)`,
부모가 요구하는 `requiredChildDistribution = ClusteredDistribution([a])`.

**출력:** `Boolean`.
- `true` → 자식 분배 그대로 사용, Exchange 미삽입.
- `false` → [EnsureRequirements](ensure-requirements.md)가
  `distribution.createPartitioning(numPartitions)`로 새 partitioning을
  만들어 `ShuffleExchangeExec`로 감싸서 자식 위에 삽입.

`Partitioning.satisfies` 자체는 두 단계로 구성된다 (line 218-220):

```scala
final def satisfies(required: Distribution): Boolean = {
  required.requiredNumPartitions.forall(_ == numPartitions) && satisfies0(required)
}
```

1. `requiredNumPartitions` 일치 검사. `Distribution`이 `Some(_)`로 partition
   수를 강제하면 그 값과 자식의 `numPartitions`가 같아야 한다. `None`이면 항상 통과.
2. `satisfies0(required)` 실제 호환성 판정 — 각 `Partitioning` 구현체가 override.

## L3. Key Mechanisms

### Distribution 6종 카탈로그

| Distribution | 의미 | `requiredNumPartitions` | 인용 |
|---|---|---|---|
| `UnspecifiedDistribution` | 분배 요구 없음 | `None` | `partitioning.scala:62-68` |
| `AllTuples` | 모든 행이 단일 partition에 | `Some(1)` | `:74-81` |
| `ClusteredDistribution(clustering, requireAllClusterKeys, requiredNumPartitions)` | 같은 `clustering` 값을 가진 행은 같은 partition에 co-located | `None` (default) | `:90-120` |
| `StatefulOpClusteredDistribution(expressions, _requiredNumPartitions)` | streaming 상태 연산자용 — `HashPartitioning`만 만족 | `Some(_requiredNumPartitions)` | `:144-161` |
| `OrderedDistribution(ordering)` | 인접 partition 간 행 순서 보장 (partition 내 정렬은 무관) | `None` | `:172-184` |
| `BroadcastDistribution(mode)` | 모든 노드에 broadcast | `Some(1)` | `:190-198` |

`ClusteredDistribution.requireAllClusterKeys`는 생성 시점의 SQLConf
`spark.sql.requireAllClusterKeysForDistribution` 값을 capture한다
(`:92-93`). 이 boolean 하나가 호환성 판정을 둘로 가른다 — L4 참조.

### Partitioning.satisfies0 6종 vs Distribution 6종 — 호환성 매트릭스

각 셀: `true` 조건 / `false` 또는 인용 라인.

| Partitioning ↓ \ Distribution → | Unspecified | AllTuples | ClusteredDistribution | StatefulOpClusteredDistribution | OrderedDistribution | BroadcastDistribution |
|---|---|---|---|---|---|---|
| **(Partitioning trait default)** `:241-245` | true | `numPartitions == 1` | false | false | false | false |
| **SinglePartition** `:260-263` | true (`numPartitions == 1`) | true | true | true | true | **false** |
| **HashPartitioningLike** (`HashPartitioning`, `CoalescedHashPartitioning`) `:276-294` | true (super) | super (1 partition일 때) | **분기 — 박스 ① 참조** | length 일치 + semanticEquals (`:280-282`) | false | false |
| **KeyGroupedPartitioning** `:369-387` | true (super) | super | 분기 — 박스 ① 패턴이지만 leaf attribute 사용 (`:378-380`) | false | false | false |
| **RangePartitioning** `:446-480` | true (super) | super | 분기 — HashPartitioningLike와 동일 패턴 (`:468-476`) | false | **prefix 매칭 — 박스 ② 참조** | false |
| **PartitioningCollection** `:524-525` | child 중 하나라도 satisfies (재귀) | (재귀) | (재귀) | (재귀) | (재귀) | (재귀) |
| **BroadcastPartitioning** `:548-552` | true | false | false | false | false | mode 일치 시 true (`:550`) |

`UnknownPartitioning`(`:248`)과 `RoundRobinPartitioning`(`:255`)은 별도
override가 없어 trait default(`:241-245`)에 떨어진다 — `UnspecifiedDistribution`
또는 1-partition `AllTuples`만 만족, 그 외 모두 false. 즉 이 두 partitioning은
거의 항상 Exchange를 유발한다.

`CoalescedHashPartitioning`(`:329-342`)은 AQE coalesce 결과 형태로
`HashPartitioningLike`를 상속하며, `expressions`는 원본 `from.expressions`를
그대로 노출하므로 satisfies0 판정은 `HashPartitioning`과 동일하다.

---

#### 박스 ① — `HashPartitioningLike.satisfies0`의 ClusteredDistribution 분기 (`partitioning.scala:283-290`)

```scala
case c @ ClusteredDistribution(requiredClustering, requireAllClusterKeys, _) =>
  if (requireAllClusterKeys) {
    // Checks `HashPartitioning` is partitioned on exactly same clustering keys of
    // `ClusteredDistribution`.
    c.areAllClusterKeysMatched(expressions)
  } else {
    expressions.forall(x => requiredClustering.exists(_.semanticEquals(x)))
  }
```

`requireAllClusterKeys`(즉 `spark.sql.requireAllClusterKeysForDistribution`,
default `false`)에 따라 의미가 갈린다.

- **`true` 분기** — `areAllClusterKeysMatched`(`:114-119`)는 `expressions.length
  == clustering.length` + 모든 위치에서 `semanticEquals`를 동시에 요구한다.
  즉 **정확히 같은 키 시퀀스만** satisfies.
- **`false` 분기 (default)** — `expressions`의 모든 원소가 `requiredClustering`에
  포함되면 (즉 `expressions ⊆ requiredClustering`) satisfies. 길이 동일 요구
  없음. `expressions`가 strictly smaller여도 OK.

이 분기의 정확한 의미는 다음과 같이 정리된다:

| 관계 | 분기 (false, default) | 분기 (true) |
|---|---|---|
| `expressions == requiredClustering` (정확히 일치, 같은 순서) | satisfies | satisfies |
| `expressions == requiredClustering` (같은 집합, 다른 순서) | satisfies (forall semantic) | **NOT** satisfies (위치 비교) |
| `expressions ⊊ requiredClustering` (partition 키가 distribution 키의 strict subset) | **satisfies** | NOT satisfies |
| `expressions ⊋ requiredClustering` (partition 키가 distribution 키의 strict superset) | **NOT satisfies** | NOT satisfies |
| 교집합만 있음 (둘 다 다른 원소 포함) | NOT satisfies | NOT satisfies |

→ **default 의미: "partition 키가 distribution 키의 subset (또는 동일)이면
satisfies; superset이면 재shuffle"**.

`KeyGroupedPartitioning.satisfies0`(`:369-387`)의 false 분기는 동일 패턴이지만
`expressions.flatMap(_.collectLeaves())`로 leaf attribute를 추출한다 (`:378-380`)
— transform 표현식 (예: `years(ts)`)을 leaf attribute (`ts`)로 풀어 비교하기
위함.

#### 박스 ② — `RangePartitioning.satisfies0`의 OrderedDistribution prefix 매칭 (`partitioning.scala:449-467`)

```scala
case OrderedDistribution(requiredOrdering) =>
  // If `ordering` is a prefix of `requiredOrdering`:
  //   Let's say `ordering` is [a, b] and `requiredOrdering` is [a, b, c]. According to the
  //   RangePartitioning definition, any [a, b] in a previous partition must be smaller
  //   than any [a, b] in the following partition. This also means any [a, b, c] in a
  //   previous partition must be smaller than any [a, b, c] in the following partition.
  //   Thus `RangePartitioning(a, b)` satisfies `OrderedDistribution(a, b, c)`.
  //
  // If `requiredOrdering` is a prefix of `ordering`:
  //   Let's say `ordering` is [a, b, c] and `requiredOrdering` is [a, b]. According to the
  //   RangePartitioning definition, any [a, b, c] in a previous partition must be smaller
  //   than any [a, b, c] in the following partition. If there is a [a1, b1] from a previous
  //   partition which is larger than a [a2, b2] from the following partition, then there
  //   must be a [a1, b1 c1] larger than [a2, b2, c2], which violates RangePartitioning
  //   definition. So it's guaranteed that, any [a, b] in a previous partition must not be
  //   greater(i.e. smaller or equal to) than any [a, b] in the following partition. Thus
  //   `RangePartitioning(a, b, c)` satisfies `OrderedDistribution(a, b)`.
  val minSize = Seq(requiredOrdering.size, ordering.size).min
  requiredOrdering.take(minSize) == ordering.take(minSize)
```

위 코드 주석은 RangePartitioning 정의로부터 prefix 매칭의 정당성을 직접
증명한다. 핵심: **둘 중 어느 한 쪽이 다른 쪽의 prefix이면** satisfies. 정확히
일치할 필요는 없다. 쿼리 B의 outer ORDER BY가 `RangePartitioning(agent_gu,
price_rank)`로 처리될 수 있는 이유는 이 prefix 호환성에서 온다.

`RangePartitioning`의 `ClusteredDistribution` 분기(`:468-476`)는
`HashPartitioningLike`와 동일 패턴 (분기 ①과 동일).

## L4. Performance Levers

### `spark.sql.requireAllClusterKeysForDistribution` (default `false`)

| 설정 | satisfies 의미 | 효과 |
|---|---|---|
| `false` (default) | partition 키가 distribution 키의 **subset (또는 동일)이면 satisfies**. superset이면 재shuffle. | 더 약한 partition이 더 강한 clustering을 자동 만족 — Exchange 회피 가능. |
| `true` | partition 키와 distribution 키가 **정확히 일치 (같은 길이, 같은 순서, semanticEquals)** 해야 satisfies. | 더 엄격. 같은 집합이지만 순서가 다른 경우도 재shuffle을 강제. |

쿼리 C의 두 번째 Exchange (HashAggregate Final → Window 사이의
`Exchange hashpartitioning(agent_gu, 200)`)는 default `false` 상태에서도
재shuffle이 발생한 **superset 케이스**다 (분기 ① 참조). `true`로 변경해도
같은 결과 (양쪽 분기 모두 false).

### 일반화 — 쿼리 C 패턴의 structural 한계

> **HashAggregate의 output partitioning은 hash agg key 전체로 결정된다.
> 하위 Window가 PARTITION BY를 그 키의 strict subset으로 사용하면, 그 결과는
> 항상 superset 케이스 (`expressions ⊋ requiredClustering`)이 되고, 이 조합은
> 어떤 `requireAllClusterKeysForDistribution` 설정에서도 satisfies하지 못해
> 재shuffle이 불가피하다.**

이는 단발 사례가 아닌 **구조적 한계**다. SQL 작성자가 `GROUP BY a, b, c` 후
`OVER (PARTITION BY a)`를 두면 항상 추가 Exchange가 발생한다 — partitioning
재배치 외에는 회피 방법이 없다 (예: aggregate 키를 window 키와 일치시키거나,
window 부분만 별도 query로 분리).

### 보조 lever

- `Distribution.requiredNumPartitions = Some(_)`이면 numPartitions 일치 검사
  (line 219)에서 미스매치 시 satisfies 자동 false. `BroadcastDistribution`,
  `StatefulOpClusteredDistribution`, AllTuples가 이 경로.
- `PartitioningCollection`(line 524-525)은 자식 중 하나라도 satisfies면 true —
  outer/multi-key join에서 한 쪽 키로만 partition된 데이터도 join 자체는
  통과시키는 메커니즘. SPJ 평가의 핵심 입구이기도 하다 (자세한 내용은
  [storage-partition-join.md](storage-partition-join.md), 강화 예정).

## L5. How to Observe

### Probe 제안 (PySpark)

쿼리 C의 superset 케이스를 최소 재현하는 스니펫:

```python
from pyspark.sql import functions as F
from pyspark.sql.window import Window

w = Window.partitionBy("a")  # window는 [a]만 사용

(df
  .groupBy("a", "b", "c")    # aggregate은 [a, b, c]로 partition됨
  .agg(F.sum("v").alias("sv"))
  .withColumn("rn", F.row_number().over(w))
  .explain(mode="formatted"))
```

`HashAggregate(Final)`과 `Window` 사이에 `Exchange hashpartitioning(a, 200)`이
삽입되는 것을 확인한다. partitioning 키 `[a, b, c]`가 distribution 요구 `[a]`의
superset이라 분기 ①의 false 분기에 떨어지는 케이스. 이 Exchange를 회피하려면
window의 partition by를 aggregate 키와 같은 집합 `[a, b, c]`로 맞추면 된다 —
그러면 정확 일치 케이스로 떨어져 양 분기 모두 satisfies.

상세 plan 출력 형식은 [_probes/explain-modes.md] (예정).

### Verified case 인용

[case/2026-04-19-query-C-window-moving-avg.md](../case/2026-04-19-query-C-window-moving-avg.md)는
이 패턴의 ground truth다. 같은 case의 Exchange #2 (`plan_id=507`,
`hashpartitioning(agent_gu, 200)`)가 본 페이지의 박스 ①의 default false 분기로
정확히 설명된다.

[case/2026-04-19-query-B-row-number-top5.md](../case/2026-04-19-query-B-row-number-top5.md)의
outer ORDER BY `Exchange rangepartitioning(agent_gu ASC, price_rank ASC, 200)`은
박스 ②의 prefix 매칭으로 설명된다 (단, 이 case는 prefix 매칭이 아닌
신규 RangePartitioning 생성 — `OrderedDistribution`을 그대로 만족하는 자식이
없어 `OrderedDistribution.createPartitioning`이 새 RangePartitioning을 만든
경로).

[case/2026-04-19-query-A-self-join-growth.md](../case/2026-04-19-query-A-self-join-growth.md)의
SMJ 양쪽 `Exchange hashpartitioning(legal_dong, 200)`은
[ensure-requirements.md](ensure-requirements.md)의 multi-child co-partitioning
경로 — 본 페이지의 satisfies 매트릭스 단독으로는 설명 부족 (양쪽 자식 모두
처음부터 partitioning을 자체 생성하지 못해 비교 대상이 없는 케이스).

## Version notes

- v3.5.6 commit `303c18c74664f161b9b969ac343784c088b47593` 기준.
- v3.5.6 `partitioning.scala`에는 **`HashClusteredDistribution`이 별도 case
  class로 존재하지 않는다**. 과거 버전에는 별도 distribution이 있었으나
  현재는 `ClusteredDistribution` + `HashPartitioningLike.satisfies0`의 조합으로
  통합되었다. 통합 시점(어느 SPARK ticket / 어느 버전)은 후속 ingest 대상
  (Gaps 참조).
- `StatefulOpClusteredDistribution`(line 144-161)의 코드 주석(`:127-142`)은
  "ClusteredDistribution requires relaxed condition and multiple partitionings
  can satisfy the requirement" 라는 표현으로 본 페이지의 default false 분기의
  성격을 직접 명시한다 — verified의 보조 근거.

## Gaps

- `HashClusteredDistribution` 통합 이력 — 어느 SPARK ticket / 어느 Spark
  버전에서 `ClusteredDistribution`로 통합되었는지 추적.
- `ShuffleSpec` 트레이트와 그 구현체 (`SinglePartitionShuffleSpec`,
  `RangeShuffleSpec`, `HashShuffleSpec`, `KeyGroupedShuffleSpec`,
  `ShuffleSpecCollection` 등, line 566-806 일부)의 `isCompatibleWith` 메커니즘
  — multi-child co-partitioning 결정의 핵심이지만 본 ingest 범위 외.
- `KeyGroupedPartitioning` + SPJ partial clustering 깊은 다이브 —
  [storage-partition-join.md](storage-partition-join.md) 강화와 함께.
- `HashPartitioning.partitionIdExpression`(line 317:
  `Pmod(Murmur3Hash(expressions), Literal(numPartitions))`) — 실제 partition
  ID 계산식. shuffle write 단계와 연결되는 다리, 후속 ingest로 단계 09에서 인용.
- **AQE coalesce와 partitioning 호환성** — `CoalescedHashPartitioning`(line
  329-342)이 `HashPartitioningLike`를 상속해 satisfies0를 동일 사용한다는 본
  ingest의 관찰을 후속 페이지로 격상 가능. AQE가 coalesce를 거친 자식의
  outputPartitioning이 그대로 다음 노드의 distribution 요구를 만족할 수 있는지
  여부는 Iceberg + AQE 환경에서 가치 높은 주제. 후속 페이지 후보.
- **frontmatter `last_verified` 필드 retroactive 추가** —
  [join-strategy-hints.md](join-strategy-hints.md)에 누락된 상태. 이번 두
  페이지는 추가됨. 후속 lint pass에서 정리.
