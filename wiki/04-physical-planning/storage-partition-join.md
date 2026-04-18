---
stage: 04-physical-planning
title: Storage Partition Join
spark_versions: ["4.1"]
sources:
  - raw/spark-docs/2026-04-spark-4.1.1-sql-performance-tuning.md
last_verified: 2026-04-19
confidence: tentative
related:
  - 00-overview/execution-pipeline.md
  - 04-physical-planning/join-strategy-hints.md
configs:
  - spark.sql.sources.v2.bucketing.enabled
  - spark.sql.sources.v2.bucketing.pushPartValues.enabled
  - spark.sql.requireAllClusterKeysForCoPartition
  - spark.sql.sources.v2.bucketing.partiallyClusteredDistribution.enabled
  - spark.sql.sources.v2.bucketing.allowJoinKeysSubsetOfPartitionKeys.enabled
  - spark.sql.sources.v2.bucketing.allowCompatibleTransforms.enabled
  - spark.sql.sources.v2.bucketing.shuffle.enabled
apis: []
tags: [join, storage-partition-join, spj, datasource-v2, bucketing, iceberg]
---

## L1. What & Why

Storage Partition Join (SPJ)은 Data Source V2 connector가 보고한
partitioning을 이용해 join 전 shuffle을 완전히 생략하는 최적화다. SPJ가
없으면 두 개의 큰 partitioned 테이블에 대한 join은 일치하는 key를
같은 노드에 배치하기 위해 양쪽 모두 `Exchange`가 필요하며, 이 shuffle
비용이 runtime을 지배한다. SPJ는 Hive 스타일 bucketed 테이블에만
적용되던 기존 bucket-join 개념을, 호환 가능한 partition 함수를 노출하는
모든 V2 source(Iceberg의 `bucket()`, `truncate()`, identity partition
등)로 일반화한 것이다. SPJ가 동작하면 physical plan의 join 앞에
`Exchange` 노드가 없다.

## L2. Inputs & Outputs

입력: `Join` 노드를 포함한 Optimized LogicalPlan. 양쪽이 모두
`DataSourceV2Relation`(또는 lowering 후 `BatchScan`)이며 V2 connector가
호환되는 partitioning을 보고한 상태.

출력: join의 upstream `Exchange` 노드가 생략된 physical plan. 양쪽
모두 `BatchScan → ColumnarToRow → Filter → Sort → SortMergeJoin`으로
shuffle 없이 흐른다. 예시 plan은 `raw/spark-docs/2026-04-spark-4.1.1-sql-performance-tuning.md`의
Storage Partition Join 섹션 참고.

## L3. Key Mechanisms

> 📝 docs 요약 수준. 소스 코드 ingest 대기 중
> (`sql/core/src/main/scala/org/apache/spark/sql/execution/datasources/v2/BatchScanExec.scala`,
> `sql/catalyst/.../plans/physical/partitioning.scala`,
> V2 planner의 SPJ 관련 규칙).

SPJ는 다음 조건에서 동작한다:

1. 양쪽 모두 V2 DataSource relation이며 connector가
   `SupportsReportPartitioning`을 구현한다.
2. 보고된 partitioning이 활성화된 feature flag 조건 아래서 호환된다.

7개의 설정이 매칭 기준을 단계적으로 완화한다:

| Config | Default | 역할 |
|---|---|---|
| `spark.sql.sources.v2.bucketing.enabled` | true | 기본 gate: V2가 보고한 partitioning으로 shuffle 제거 시도 |
| `spark.sql.sources.v2.bucketing.pushPartValues.enabled` | true | 한쪽에 있는 partition 값이 다른 쪽에 없는 경우에도 SPJ 허용 |
| `spark.sql.requireAllClusterKeysForCoPartition` | true | join key가 partition key와 정확히, 그리고 같은 순서로 일치하도록 요구 |
| `spark.sql.sources.v2.bucketing.partiallyClusteredDistribution.enabled` | false | partition 크기 skew를 한쪽만 partial clustering으로 처리 |
| `spark.sql.sources.v2.bucketing.allowJoinKeysSubsetOfPartitionKeys.enabled` | false | join key가 partition key의 부분집합인 경우에도 SPJ 허용 |
| `spark.sql.sources.v2.bucketing.allowCompatibleTransforms.enabled` | false | 서로 다르지만 호환되는 partition transform에도 SPJ 허용 |
| `spark.sql.sources.v2.bucketing.shuffle.enabled` | false | 한쪽만 partitioned인 경우 — 다른 쪽을 shuffle 해 맞춤 |

4.1.1에 기술된 의존 관계:

- `pushPartValues.enabled`는 `bucketing.enabled = true`를 요구.
- `partiallyClusteredDistribution.enabled`는 `bucketing.enabled`와
  `pushPartValues.enabled` 모두 요구.
- `allowJoinKeysSubsetOfPartitionKeys.enabled`는 위 둘에 더해
  `requireAllClusterKeysForCoPartition = false`를 요구.
- `allowCompatibleTransforms.enabled`는 `bucketing.enabled`와
  `pushPartValues.enabled`를 요구.

`requireAllClusterKeysForCoPartition`은 특히 중요하다: 기본값 `true`는
partition 컬럼의 부분집합에 대한 join에서는 SPJ가 동작하지 않음을
의미한다. 이를 `false`로 바꾸는 것이 현실적인 Iceberg 쿼리에서 SPJ를
활성화시키는 단일 스위치인 경우가 많다.

## L4. Performance Levers

1. **connector 지원이 전제 조건이다.** SPJ는 partitioning을 보고하는
   V2 DataSource가 필요하다. docs가 예로 드는 Iceberg는 지원하며,
   Hive-bucketed 테이블은 별도 코드 경로를 사용, 일반 Parquet 파일
   scan은 이 기능에 참여하지 않는다.

2. **기본 설정은 가장 엄격한 매칭만 허용한다.** 기본 상태에서
   `requireAllClusterKeysForCoPartition = true`이고 세 개의 `false`
   플래그(`partiallyClusteredDistribution`,
   `allowJoinKeysSubsetOfPartitionKeys`, `allowCompatibleTransforms`,
   `shuffle.enabled`)가 대부분 현실 SPJ 기회를 막는다. 이를 선택적으로
   활성화하는 판단은 워크로드의 join 형태에 따른다.

3. **Iceberg 고유 커플링: `spark.sql.iceberg.planning.preserve-data-grouping`.**
   docs의 예제는 SPJ 설정과 함께 이 값을 설정한다 — Iceberg connector가
   자체 플래그로 partition grouping을 노출해야 Spark가 그것을 사용할
   수 있다.

4. **`partiallyClusteredDistribution.enabled`로 skew 처리.** partition
   크기가 불균일할 때, 이 옵션을 켜면 Spark가 더 큰 쪽을 partial
   clustering 하고 작은 쪽의 split을 복제해 매칭시킨다. 심한 skew
   partition에 대해 full shuffle 없이 SPJ를 유지하고 싶을 때 유용.

## L5. How to Observe

> TODO: `queryExecution.executedPlan` 또는 SQL UI 관찰 사례 필요.

`BatchScan`과 join 사이에 `Exchange` 노드가 **없는지**를 physical
plan에서 확인한다:

```sql
EXPLAIN SELECT * FROM target t INNER JOIN source s
ON t.dep = s.dep AND t.id = s.id
```

SPJ가 동작하지 않을 때 (축약):

```
* SortMergeJoin Inner
  :- * Sort
  :   +- Exchange    -- shuffle
  :      +- * Filter
  :         +- BatchScan
  +- * Sort
      +- Exchange    -- shuffle
         +- * Filter
            +- BatchScan
```

SPJ가 동작할 때:

```
* SortMergeJoin Inner
  :- * Sort
  :   +- * Filter    -- no Exchange
  :      +- BatchScan
  +- * Sort
      +- * Filter    -- no Exchange
         +- BatchScan
```

`BatchScan` 위의 `Exchange` 유무가 ground truth.

## Version notes

- docs 섹션 "Storage Partition Join"은 4.1.1에만 존재(3.5.6에는 없음).
  기능 자체는 더 일찍 존재 — 아래 설정의 "Since Version" 참고 — 하지만
  전용 사용자 docs는 4.1에서 처음 추가됨.
- 설정 도입 히스토리:
  - `spark.sql.sources.v2.bucketing.enabled`: since 3.3.0.
  - `spark.sql.sources.v2.bucketing.pushPartValues.enabled`: since 3.4.0.
  - `spark.sql.requireAllClusterKeysForCoPartition`: since 3.4.0.
  - `spark.sql.sources.v2.bucketing.partiallyClusteredDistribution.enabled`:
    since 3.4.0.
  - `spark.sql.sources.v2.bucketing.allowJoinKeysSubsetOfPartitionKeys.enabled`:
    since 4.0.0.
  - `spark.sql.sources.v2.bucketing.allowCompatibleTransforms.enabled`:
    since 4.0.0.
  - `spark.sql.sources.v2.bucketing.shuffle.enabled`: since 4.0.0.
- 이 페이지가 4.1 증거만 주장하는 이유는 서술 + 의존 규칙이 4.1.1
  docs에만 존재하기 때문. 3.3–4.0에서의 동작을 주장하려면 해당 버전
  docs 또는 소스 코드를 ingest 해야 함.
