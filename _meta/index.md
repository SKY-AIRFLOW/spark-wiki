# Spark Wiki 카탈로그

파이프라인 단계별로 묶인 최상위 인덱스. `/ingest`와 `/trace`가
실제 페이지를 생성함에 따라 이 목록이 성장한다.

## 00 — Overview

- [wiki/00-overview/execution-pipeline.md](../wiki/00-overview/execution-pipeline.md) — Scaffolding. SQL → Executor 10개 화살표.

## 01 — Parsing

_비어 있음._

## 02 — Analysis

- [wiki/02-analysis/analyzer-overview.md](../wiki/02-analysis/analyzer-overview.md) — Unresolved → Analyzed LogicalPlan 변환, exprId 부여 (tentative, query-A trace 기반).

## 03 — Logical Optimization

- [wiki/03-logical-optimization/predicate-pushdown.md](../wiki/03-logical-optimization/predicate-pushdown.md) — Filter를 Join/Scan 하위로 밀어넣는 최적화, data source 경계 pushdown 포함 (tentative).
- [wiki/03-logical-optimization/cte-inline.md](../wiki/03-logical-optimization/cte-inline.md) — WithCTE/CTERelationDef/CTERelationRef의 구조와 Optimizer에 의한 inline 소거 (tentative, Optimizer.scala ingest 대기).

## 04 — Physical Planning

- [wiki/04-physical-planning/join-strategy-hints.md](../wiki/04-physical-planning/join-strategy-hints.md) — BROADCAST/MERGE/SHUFFLE_HASH/SHUFFLE_REPLICATE_NL hints + auto-broadcast threshold.
- [wiki/04-physical-planning/coalesce-repartition-hints.md](../wiki/04-physical-planning/coalesce-repartition-hints.md) — COALESCE/REPARTITION/REPARTITION_BY_RANGE/REBALANCE + shuffle/file partition configs.
- [wiki/04-physical-planning/storage-partition-join.md](../wiki/04-physical-planning/storage-partition-join.md) — V2 DataSource partitioning으로 Exchange 제거.
- [wiki/04-physical-planning/hash-aggregate-partial-final.md](../wiki/04-physical-planning/hash-aggregate-partial-final.md) — GROUP BY의 partial → final 2단계 HashAggregate 패턴 (tentative).
- [wiki/04-physical-planning/batch-scan-iceberg.md](../wiki/04-physical-planning/batch-scan-iceberg.md) — Iceberg V2 DataSource BatchScan 연산자, filter pushdown 관찰 (tentative, Iceberg 소스 ingest 대기).
- [wiki/04-physical-planning/window-exec.md](../wiki/04-physical-planning/window-exec.md) — Window function 물리 실행, Window strategy (tentative, L3 일부 SparkStrategies.scala:641-655 인용).
- [wiki/04-physical-planning/window-group-limit.md](../wiki/04-physical-planning/window-group-limit.md) — Top-N per Group 최적화, Partial/Final 분할 (tentative, L3 일부 SparkStrategies.scala:657-667 인용; InsertWindowGroupLimit rule은 Optimizer 미-ingest).

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

- [wiki/case/2026-04-19-query-A-self-join-growth.md](../wiki/case/2026-04-19-query-A-self-join-growth.md) — apt_trades self-join (2024 vs 2023) + HashAggregate + TakeOrderedAndProject. Spark 3.5.3 + Iceberg V2. SMJ 선택 근거 EXPLAIN COST로 검증. AQE `isFinalPlan=false` — 런타임 결정 미포함.
- [wiki/case/2026-04-19-query-B-row-number-top5.md](../wiki/case/2026-04-19-query-B-row-number-top5.md) — 서울 자치구별 ROW_NUMBER() Top-5. Spark 3.5의 WindowGroupLimit (Partial/Final 분할 strategy-level) + rangepartitioning 첫 등장 + "Top-N per group" vs "전역 Top-N" 대비 (쿼리 A와 비교 섹션 포함).
- [wiki/case/2026-04-19-query-C-window-moving-avg.md](../wiki/case/2026-04-19-query-C-window-moving-avg.md) — CTE(단일 참조, inline) + group aggregate + AVG() window (ROWS BETWEEN 5 PRECEDING). Exchange 3개 (aggregate key ⊋ window partition key 불일치로 추가 Exchange 불가피) + bounded RowFrame + WindowGroupLimit 미적용 (AVG는 rank-like 아님, 소스 근거). 3쿼리 대조 4개 표 (구조/Exchange/Statistics/WGL).
