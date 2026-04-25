# API → Stage Index

DataFrame / SQL API → 주로 처리되는 파이프라인 단계 + 관련 페이지.
`/lint`가 단계 페이지의 `apis:` frontmatter와 이 테이블을 교차
검증한다. 행 순서는 단계 → API 이름순.

## User-facing APIs

| API | Stage | Pages |
|---|---|---|
| `DataFrame.coalesce` | 04 | [coalesce-repartition-hints](../../wiki/04-physical-planning/coalesce-repartition-hints.md) |
| `DataFrame.groupBy` | 04 | [ensure-requirements](../../wiki/04-physical-planning/ensure-requirements.md), [partitioning-compatibility](../../wiki/04-physical-planning/partitioning-compatibility.md), [hash-aggregate-partial-final](../../wiki/04-physical-planning/hash-aggregate-partial-final.md) |
| `DataFrame.hint` | 04 | [join-strategy-hints](../../wiki/04-physical-planning/join-strategy-hints.md), [coalesce-repartition-hints](../../wiki/04-physical-planning/coalesce-repartition-hints.md) |
| `DataFrame.join` | 04 | [join-strategy-hints](../../wiki/04-physical-planning/join-strategy-hints.md), [ensure-requirements](../../wiki/04-physical-planning/ensure-requirements.md), [partitioning-compatibility](../../wiki/04-physical-planning/partitioning-compatibility.md) |
| `DataFrame.orderBy` | 04 | [ensure-requirements](../../wiki/04-physical-planning/ensure-requirements.md), [partitioning-compatibility](../../wiki/04-physical-planning/partitioning-compatibility.md) |
| `DataFrame.repartition` | 04 | [coalesce-repartition-hints](../../wiki/04-physical-planning/coalesce-repartition-hints.md) |
| `DataFrame.repartitionByRange` | 04 | [coalesce-repartition-hints](../../wiki/04-physical-planning/coalesce-repartition-hints.md) |
| `Window.partitionBy` | 04 | [ensure-requirements](../../wiki/04-physical-planning/ensure-requirements.md), [partitioning-compatibility](../../wiki/04-physical-planning/partitioning-compatibility.md), [window-exec](../../wiki/04-physical-planning/window-exec.md) |
| SQL `/*+ BROADCAST */` | 04 | [join-strategy-hints](../../wiki/04-physical-planning/join-strategy-hints.md) |
| SQL `/*+ COALESCE */` | 04 | [coalesce-repartition-hints](../../wiki/04-physical-planning/coalesce-repartition-hints.md) |
| SQL `/*+ MERGE */` | 04 | [join-strategy-hints](../../wiki/04-physical-planning/join-strategy-hints.md) |
| SQL `/*+ REBALANCE */` | 04 | [coalesce-repartition-hints](../../wiki/04-physical-planning/coalesce-repartition-hints.md) |
| SQL `/*+ REPARTITION */` | 04 | [coalesce-repartition-hints](../../wiki/04-physical-planning/coalesce-repartition-hints.md) |
| SQL `/*+ REPARTITION_BY_RANGE */` | 04 | [coalesce-repartition-hints](../../wiki/04-physical-planning/coalesce-repartition-hints.md) |
| SQL `/*+ SHUFFLE_HASH */` | 04 | [join-strategy-hints](../../wiki/04-physical-planning/join-strategy-hints.md) |
| SQL `/*+ SHUFFLE_REPLICATE_NL */` | 04 | [join-strategy-hints](../../wiki/04-physical-planning/join-strategy-hints.md) |
| `ANALYZE TABLE` | 05 | [leveraging-statistics](../../wiki/05-cost-and-aqe/leveraging-statistics.md) |
| `DataFrame.explain(mode="cost")` | 05 | [leveraging-statistics](../../wiki/05-cost-and-aqe/leveraging-statistics.md) |
| `DESCRIBE EXTENDED` | 05 | [leveraging-statistics](../../wiki/05-cost-and-aqe/leveraging-statistics.md) |
| `EXPLAIN COST` | 05 | [leveraging-statistics](../../wiki/05-cost-and-aqe/leveraging-statistics.md) |
| `DataFrame.cache` | 09 | [in-memory-columnar-cache](../../wiki/09-execution/in-memory-columnar-cache.md) |
| `DataFrame.unpersist` | 09 | [in-memory-columnar-cache](../../wiki/09-execution/in-memory-columnar-cache.md) |
| `spark.catalog.cacheTable` | 09 | [in-memory-columnar-cache](../../wiki/09-execution/in-memory-columnar-cache.md) |
| `spark.catalog.uncacheTable` | 09 | [in-memory-columnar-cache](../../wiki/09-execution/in-memory-columnar-cache.md) |

## Internal Catalyst classes / methods

내부 SparkPlan rule, Distribution / Partitioning 클래스, satisfies / satisfies0
메서드 등의 위치 인덱스. 사용자 API가 아니지만 plan trace를 읽거나 소스
검증을 할 때 자주 참조된다.

| Class / Method | Stage | Pages |
|---|---|---|
| `EnsureRequirements` (rule) | 04 | [ensure-requirements](../../wiki/04-physical-planning/ensure-requirements.md) |
| `Partitioning.satisfies` (final) | 04 | [partitioning-compatibility](../../wiki/04-physical-planning/partitioning-compatibility.md), [ensure-requirements](../../wiki/04-physical-planning/ensure-requirements.md) |
| `HashPartitioning.satisfies0` (via `HashPartitioningLike`) | 04 | [partitioning-compatibility](../../wiki/04-physical-planning/partitioning-compatibility.md) |
| `RangePartitioning.satisfies0` | 04 | [partitioning-compatibility](../../wiki/04-physical-planning/partitioning-compatibility.md) |
| `ClusteredDistribution` | 04 | [partitioning-compatibility](../../wiki/04-physical-planning/partitioning-compatibility.md), [ensure-requirements](../../wiki/04-physical-planning/ensure-requirements.md) |
| `OrderedDistribution` | 04 | [partitioning-compatibility](../../wiki/04-physical-planning/partitioning-compatibility.md) |
| `JoinSelection` (object) | 04 | [join-strategy-hints](../../wiki/04-physical-planning/join-strategy-hints.md) |
