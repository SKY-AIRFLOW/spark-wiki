# Config Index

`spark.sql.*` 및 관련 설정 → 파이프라인 단계 + 이를 다루는 페이지.
`/lint`가 단계 페이지의 `configs:` frontmatter와 이 테이블을 교차
검증한다. 행 순서는 단계 → 설정 이름 알파벳순.

| Config | Default | Stage | Pages |
|---|---|---|---|
| `spark.sql.cbo.enabled` | false | 03 | [strategy-framework](../../wiki/04-physical-planning/strategy-framework.md) |
| `spark.sql.constraintPropagation.enabled` | true | 03 | [infer-filters-from-constraints](../../wiki/03-logical-optimization/infer-filters-from-constraints.md) |
| `spark.sql.legacy.ctePrecedencePolicy` | EXCEPTION | 03 | [cte-inline](../../wiki/03-logical-optimization/cte-inline.md) |
| `spark.sql.optimizer.dynamicPartitionPruning.enabled` | true | 03 | [predicate-pushdown](../../wiki/03-logical-optimization/predicate-pushdown.md) |
| `spark.sql.optimizer.excludedRules` | (none) | 03 | [predicate-pushdown](../../wiki/03-logical-optimization/predicate-pushdown.md), [cte-inline](../../wiki/03-logical-optimization/cte-inline.md), [column-pruning](../../wiki/03-logical-optimization/column-pruning.md), [like-simplification](../../wiki/03-logical-optimization/like-simplification.md), [infer-filters-from-constraints](../../wiki/03-logical-optimization/infer-filters-from-constraints.md) |
| `spark.sql.optimizer.topKSortFallbackThreshold` | (verify) | 03 | [infer-window-group-limit](../../wiki/03-logical-optimization/infer-window-group-limit.md) |
| `spark.sql.optimizer.windowGroupLimitThreshold` | 1000 | 03 | [infer-window-group-limit](../../wiki/03-logical-optimization/infer-window-group-limit.md) |
| `spark.sql.autoBroadcastJoinThreshold` | 10485760 (10 MB) | 04 | [join-strategy-hints](../../wiki/04-physical-planning/join-strategy-hints.md) |
| `spark.sql.broadcastTimeout` | 300 | 04 | [join-strategy-hints](../../wiki/04-physical-planning/join-strategy-hints.md) |
| `spark.sql.join.preferSortMergeJoin` | true | 04 | [join-strategy-hints](../../wiki/04-physical-planning/join-strategy-hints.md) |
| `spark.sql.shuffle.hashJoinFactor` | 3 | 04 | [join-strategy-hints](../../wiki/04-physical-planning/join-strategy-hints.md) |
| `spark.sql.files.maxPartitionBytes` | 134217728 (128 MB) | 04 | [coalesce-repartition-hints](../../wiki/04-physical-planning/coalesce-repartition-hints.md) |
| `spark.sql.files.maxPartitionNum` | None | 04 | [coalesce-repartition-hints](../../wiki/04-physical-planning/coalesce-repartition-hints.md) |
| `spark.sql.files.minPartitionNum` | Default Parallelism | 04 | [coalesce-repartition-hints](../../wiki/04-physical-planning/coalesce-repartition-hints.md) |
| `spark.sql.files.openCostInBytes` | 4194304 (4 MB) | 04 | [coalesce-repartition-hints](../../wiki/04-physical-planning/coalesce-repartition-hints.md) |
| `spark.sql.maxSinglePartitionBytes` | 128 MB (verify) | 04 | [ensure-requirements](../../wiki/04-physical-planning/ensure-requirements.md) |
| `spark.sql.requireAllClusterKeysForCoPartition` | true | 04 | [storage-partition-join](../../wiki/04-physical-planning/storage-partition-join.md) |
| `spark.sql.requireAllClusterKeysForDistribution` | false | 04 | [partitioning-compatibility](../../wiki/04-physical-planning/partitioning-compatibility.md), [ensure-requirements](../../wiki/04-physical-planning/ensure-requirements.md) |
| `spark.sql.shuffle.partitions` | 200 | 04 | [coalesce-repartition-hints](../../wiki/04-physical-planning/coalesce-repartition-hints.md), [ensure-requirements](../../wiki/04-physical-planning/ensure-requirements.md), [strategy-framework](../../wiki/04-physical-planning/strategy-framework.md) |
| `spark.sql.sources.parallelPartitionDiscovery.parallelism` | 10000 | 04 | [coalesce-repartition-hints](../../wiki/04-physical-planning/coalesce-repartition-hints.md) |
| `spark.sql.sources.parallelPartitionDiscovery.threshold` | 32 | 04 | [coalesce-repartition-hints](../../wiki/04-physical-planning/coalesce-repartition-hints.md) |
| `spark.sql.sources.v2.bucketing.allowCompatibleTransforms.enabled` | false | 04 | [storage-partition-join](../../wiki/04-physical-planning/storage-partition-join.md) |
| `spark.sql.sources.v2.bucketing.allowJoinKeysSubsetOfPartitionKeys.enabled` | false | 04 | [storage-partition-join](../../wiki/04-physical-planning/storage-partition-join.md) |
| `spark.sql.sources.v2.bucketing.enabled` | true | 04 | [storage-partition-join](../../wiki/04-physical-planning/storage-partition-join.md), [ensure-requirements](../../wiki/04-physical-planning/ensure-requirements.md) |
| `spark.sql.sources.v2.bucketing.partiallyClusteredDistribution.enabled` | false | 04 | [storage-partition-join](../../wiki/04-physical-planning/storage-partition-join.md), [ensure-requirements](../../wiki/04-physical-planning/ensure-requirements.md) |
| `spark.sql.sources.v2.bucketing.pushPartValues.enabled` | true | 04 | [storage-partition-join](../../wiki/04-physical-planning/storage-partition-join.md), [ensure-requirements](../../wiki/04-physical-planning/ensure-requirements.md) |
| `spark.sql.sources.v2.bucketing.shuffle.enabled` | false | 04 | [storage-partition-join](../../wiki/04-physical-planning/storage-partition-join.md) |
| `spark.sql.adaptive.advisoryPartitionSizeInBytes` | 64 MB | 05 | [aqe-overview](../../wiki/05-cost-and-aqe/aqe-overview.md) |
| `spark.sql.adaptive.autoBroadcastJoinThreshold` | (matches `spark.sql.autoBroadcastJoinThreshold`) | 05 | [aqe-overview](../../wiki/05-cost-and-aqe/aqe-overview.md) |
| `spark.sql.adaptive.coalescePartitions.enabled` | true | 05 | [aqe-overview](../../wiki/05-cost-and-aqe/aqe-overview.md) |
| `spark.sql.adaptive.coalescePartitions.initialPartitionNum` | (none) | 05 | [aqe-overview](../../wiki/05-cost-and-aqe/aqe-overview.md) |
| `spark.sql.adaptive.coalescePartitions.minPartitionSize` | 1MB | 05 | [aqe-overview](../../wiki/05-cost-and-aqe/aqe-overview.md) |
| `spark.sql.adaptive.coalescePartitions.parallelismFirst` | true | 05 | [aqe-overview](../../wiki/05-cost-and-aqe/aqe-overview.md) |
| `spark.sql.adaptive.customCostEvaluatorClass` | (none) | 05 | [aqe-overview](../../wiki/05-cost-and-aqe/aqe-overview.md) |
| `spark.sql.adaptive.enabled` | true (default since 3.2.0) | 05 | [aqe-overview](../../wiki/05-cost-and-aqe/aqe-overview.md), [strategy-framework](../../wiki/04-physical-planning/strategy-framework.md) |
| `spark.sql.adaptive.forceOptimizeSkewedJoin` | false | 05 | [aqe-skew-join](../../wiki/05-cost-and-aqe/aqe-skew-join.md) |
| `spark.sql.adaptive.localShuffleReader.enabled` | true | 05 | [aqe-overview](../../wiki/05-cost-and-aqe/aqe-overview.md) |
| `spark.sql.adaptive.maxShuffledHashJoinLocalMapThreshold` | 0 | 05 | [aqe-overview](../../wiki/05-cost-and-aqe/aqe-overview.md) |
| `spark.sql.adaptive.optimizeSkewsInRebalancePartitions.enabled` | true | 05 | [aqe-overview](../../wiki/05-cost-and-aqe/aqe-overview.md) |
| `spark.sql.adaptive.optimizer.excludedRules` | (none) | 05 | [aqe-overview](../../wiki/05-cost-and-aqe/aqe-overview.md) |
| `spark.sql.adaptive.rebalancePartitionsSmallPartitionFactor` | 0.2 | 05 | [aqe-overview](../../wiki/05-cost-and-aqe/aqe-overview.md) |
| `spark.sql.adaptive.skewJoin.enabled` | true | 05 | [aqe-skew-join](../../wiki/05-cost-and-aqe/aqe-skew-join.md) |
| `spark.sql.adaptive.skewJoin.skewedPartitionFactor` | 5.0 | 05 | [aqe-skew-join](../../wiki/05-cost-and-aqe/aqe-skew-join.md) |
| `spark.sql.adaptive.skewJoin.skewedPartitionThresholdInBytes` | 256MB | 05 | [aqe-skew-join](../../wiki/05-cost-and-aqe/aqe-skew-join.md) |
| `spark.sql.inMemoryColumnarStorage.batchSize` | 10000 | 09 | [in-memory-columnar-cache](../../wiki/09-execution/in-memory-columnar-cache.md) |
| `spark.sql.inMemoryColumnarStorage.compressed` | true | 09 | [in-memory-columnar-cache](../../wiki/09-execution/in-memory-columnar-cache.md) |
