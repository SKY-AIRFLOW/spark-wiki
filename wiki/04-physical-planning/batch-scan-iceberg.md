---
stage: 04-physical-planning
title: BatchScan — Iceberg V2 DataSource Scan 연산자
spark_versions: ["3.5"]
sources:
  - raw/explain/2026-04-19-query-A-self-join-growth.txt
  - raw/explain/2026-04-19-query-B-row-number-top5.txt
  - raw/explain/2026-04-19-query-C-window-moving-avg.txt
last_verified: 2026-04-19
confidence: tentative
related:
  - 00-overview/execution-pipeline.md
  - 03-logical-optimization/predicate-pushdown.md
  - case/2026-04-19-query-A-self-join-growth.md
configs:
  - spark.sql.catalog.<name>
apis:
  - SparkSession.table
  - SQL SELECT ... FROM <catalog.schema.table>
tags: [iceberg, v2-datasource, batch-scan, scan-node, external-plugin]
---

## L1. What & Why

`BatchScan`은 Spark의 **DataSource V2** 인터페이스 (`Scan`, `Batch`)
를 구현한 외부 connector가 제공하는 scan 연산자를 plan에 나타내는
범용 물리 노드다. 파이프라인 단계 04의 `DataSourceV2Strategy`가
Optimized LogicalPlan의 `RelationV2` → Physical의 `BatchScan`
변환을 담당하며, **실제 scan 동작 (파일 선택, partition pruning,
filter 적용, read format)은 connector 구현체가 결정**한다.

이 Wiki가 다루는 세 쿼리는 모두 Apache Iceberg (`nessie` catalog,
iceberg-spark-runtime-3.5_2.12) 기반 테이블을 읽는다. 따라서 모든
Plan 캡처에서 `BatchScan nessie.real_estate.apt_trades ... (branch=null)`
노드가 leaf로 등장한다.

이 페이지는 두 영역을 구분한다:

1. **Spark 내부 영역** — `RelationV2` → `BatchScan` 변환, V2 API
   계약, pushdown 협상 프로토콜 (Spark 소스).
2. **Iceberg 구현 영역** — `BatchScan`이 실제 어떤 file group을
   읽을지, manifest·snapshot은 어떻게 조회되는지, partition
   pruning과 filter pushdown은 어떻게 되는지 (Iceberg 소스).

영역 2는 전적으로 외부 플러그인이므로 Spark 소스만으로는 답할 수
없다 — 별도 Iceberg 소스 ingest가 필요하다.

## L2. Inputs & Outputs

**입력** (Optimized LogicalPlan):
- `RelationV2[col#id, ...] <table_identifier> <table_identifier>` —
  `DataSourceV2Relation`. 스키마는 projection pruning으로 이미
  축소된 상태.
- Relation 위에 남은 `Filter`의 일부는 pushdown 협상 (Spark 소스의
  `V2ScanRelationPushDown`)을 통해 scan builder에 전달됨.

**출력** (Physical Plan):

```
BatchScan <table_identifier>[col#id, ...] <table_identifier>
  (branch=null)
  [filters=<pushed filters list>,
   groupedBy=<grouping columns for SPJ, 이 세 쿼리는 모두 비어 있음>]
  RuntimeFilters: [<DPP/DDR에서 주입되는 runtime filter>]
```

**쿼리 A curr side 관찰**:

```
BatchScan nessie.real_estate.apt_trades
  [apt_seq#188, deal_year#190, deal_amount#193,
   legal_dong#196, agent_gu#197]
  nessie.real_estate.apt_trades (branch=null)
  [filters=deal_year IS NOT NULL, agent_gu IS NOT NULL,
           deal_year = 2024, agent_gu LIKE '서울%',
           legal_dong IS NOT NULL,
           groupedBy=]
  RuntimeFilters: []
```

주목할 속성:
- **Column pruning 적용됨**: 원 테이블 17 컬럼 중 5개만 읽음.
- **Filter pushdown 적용됨**: 5개 조건이 Iceberg 쪽에 전달됨. 어느
  조건이 파일/manifest 레벨에서 skip에 쓰였는지는 이 plan만으로는
  알 수 없다 — Iceberg metric 필요.
- `branch=null`: Iceberg branch (time-travel/WAP) 사용 없음, main
  snapshot 읽기.
- `groupedBy=`: Storage-Partitioned Join (SPJ) 비활성.
- `RuntimeFilters: []`: Dynamic Partition Pruning (DPP) 또는 AQE
  runtime filter 주입 없음.

## L3. Key Mechanisms

> 📝 Iceberg 소스 ingest 대기 중 (iceberg-spark-runtime-3.5_2.12:1.7.1
> `SparkScanBuilder`, `SparkBatch`, `SparkPartitioningAwareScan`,
> `SparkDistributedDataScan`).

Spark 내부 변환 경로는 어제 ingest한 `SparkStrategies.scala`에 없고
`sql/core/.../datasources/v2/DataSourceV2Strategy.scala`에 있다 —
이 Spark 소스도 아직 ingest 전. 그 전까지는:

- `RelationV2` → `BatchScan` 변환이 `DataSourceV2Strategy` 책임이라는
  사실만 확정.
- Iceberg 측 scan 내부 (file group generation, partition pruning,
  row group skip, position delete 처리)는 전적으로 Iceberg 소스
  영역.

## L4. Performance Levers

> 📝 Iceberg 소스 ingest 대기 중.

소스 ingest 없이 잠정 나열:
- Iceberg side: `read.split.target-size`, `read.split.planning-lookback`,
  partition spec, sort order, metrics mode. 정확한 동작은 Iceberg
  소스 ingest 후.
- Spark side: `spark.sql.sources.v2.bucketing.enabled` (SPJ),
  `spark.sql.files.maxPartitionBytes` (V1 경로, V2에서는 source가
  결정), `spark.sql.optimizer.dynamicPartitionPruning.enabled` (DPP가
  RuntimeFilters에 반영되는 경로).

## L5. How to Observe

- `df.explain(mode="formatted")` scan 섹션:
  - `Output [N]` — projection pruning 결과
  - `<table_identifier> (branch=...) [filters=..., groupedBy=...]` —
    pushdown 결과와 SPJ 상태
  - `RuntimeFilters: [...]` — DPP/AQE runtime filter
- Iceberg metric (Spark UI → Stage detail → metrics):
  `totalPlanningDuration`, `resultDataFiles`, `resultDeleteFiles`,
  `skippedDataFiles`, `skippedDeleteFiles`. 이 지표가 pushdown의
  실제 효과를 드러낸다.

> 📝 probe 페이지 미작성. 후속 `/probe 04-physical-planning/batch-scan`
> 대상 — 특히 skipped vs read file 수 비교.
