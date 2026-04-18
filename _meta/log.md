# 활동 로그

Append-only. 모든 `/ingest`, `/trace`, `/probe`, `/lint` 실행은 여기에
한 줄 항목을 남긴다.

## [2026-04-19] init | wiki가 파이프라인 skeleton으로 초기화됨
## [2026-04-19] ingest | raw/spark-docs/2026-04-spark-{3.5.6,4.1.1}-sql-performance-tuning.md | stages: [04, 05, 09] | pages: 7 (join-strategy-hints, coalesce-repartition-hints, storage-partition-join, aqe-overview, aqe-skew-join, leveraging-statistics, in-memory-columnar-cache) | all tentative pending source-code ingest
## [2026-04-19] ingest | raw/spark-source/v3.5.6/.../SparkStrategies.scala + .../optimizer/joins.scala (JoinSelection object + JoinSelectionHelper trait, 논리 단위) | commit 303c18c74664f161b9b969ac343784c088b47593 | stages: [04] | pages: 1 (join-strategy-hints: tentative → verified, spark_versions ["3.5", "4.1"] → ["3.5"]) | config-index: +2 rows (preferSortMergeJoin, shuffle.hashJoinFactor — 소스가 새로 드러낸 항목)
