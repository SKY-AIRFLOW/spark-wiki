# Spark Wiki 카탈로그

파이프라인 단계별로 묶인 최상위 인덱스. `/ingest`와 `/trace`가
실제 페이지를 생성함에 따라 이 목록이 성장한다.

## 00 — Overview

- [wiki/00-overview/execution-pipeline.md](../wiki/00-overview/execution-pipeline.md) — Scaffolding. SQL → Executor 10개 화살표.

## 01 — Parsing

_비어 있음._

## 02 — Analysis

_비어 있음._

## 03 — Logical Optimization

_비어 있음._

## 04 — Physical Planning

- [wiki/04-physical-planning/join-strategy-hints.md](../wiki/04-physical-planning/join-strategy-hints.md) — BROADCAST/MERGE/SHUFFLE_HASH/SHUFFLE_REPLICATE_NL hints + auto-broadcast threshold.
- [wiki/04-physical-planning/coalesce-repartition-hints.md](../wiki/04-physical-planning/coalesce-repartition-hints.md) — COALESCE/REPARTITION/REPARTITION_BY_RANGE/REBALANCE + shuffle/file partition configs.
- [wiki/04-physical-planning/storage-partition-join.md](../wiki/04-physical-planning/storage-partition-join.md) — V2 DataSource partitioning으로 Exchange 제거.

## 05 — Cost and AQE

- [wiki/05-cost-and-aqe/aqe-overview.md](../wiki/05-cost-and-aqe/aqe-overview.md) — AQE umbrella + coalesce/SMJ→BHJ/SMJ→SHJ/rebalance-skew/advanced customization.
- [wiki/05-cost-and-aqe/aqe-skew-join.md](../wiki/05-cost-and-aqe/aqe-skew-join.md) — Skew join 탐지와 분할 (SMJ 전용).
- [wiki/05-cost-and-aqe/leveraging-statistics.md](../wiki/05-cost-and-aqe/leveraging-statistics.md) — data-source / catalog / runtime 통계 + DESCRIBE EXTENDED / EXPLAIN COST (4.1 only).

## 06 — Codegen

_비어 있음._

## 07 — RDD and Stages

_비어 있음._

## 08 — Scheduling

_비어 있음._

## 09 — Execution

- [wiki/09-execution/in-memory-columnar-cache.md](../wiki/09-execution/in-memory-columnar-cache.md) — 컬럼 기반 캐시 + compressed/batchSize.

## Probes

_비어 있음._

## Cases

_비어 있음._
