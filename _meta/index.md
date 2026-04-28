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

- [wiki/03-logical-optimization/column-pruning.md](../wiki/03-logical-optimization/column-pruning.md) — 부모가 미참조하는 attribute를 자식 projection/aggregate/window에서 제거 + leaf RelationV2 컬럼 축소. 7-group PartialFunction 분류 (16 케이스 압축) + 이중 wiring (Catalyst + SparkOptimizer Extract Python UDFs 재실행). case A/B/C 17→5/6/4 컬럼 축소 (verified, Optimizer.scala 소스).
- [wiki/03-logical-optimization/cte-inline.md](../wiki/03-logical-optimization/cte-inline.md) — WithCTE/CTERelationDef/CTERelationRef 구조의 inline 소거. 2단계 처리 (CTESubstitution analysis + InlineCTE logical opt) + shouldInline 3가지 OR 분기 + 3-pass 알고리즘. case C refCount==1 inline 메커니즘 verified (verified, InlineCTE.scala + CTESubstitution.scala 소스).
- [wiki/03-logical-optimization/infer-filters-from-constraints.md](../wiki/03-logical-optimization/infer-filters-from-constraints.md) — child constraints에서 부모 Filter/Join에 새 filter inference. conf 게이트 `spark.sql.constraintPropagation.enabled` + Once batch가 fixedPoint 사이에 끼임 → 다음 fixedPoint의 PushDownPredicates 입력. case A `legal_dong IS NOT NULL` transitive nullability (verified, 얕게).
- [wiki/03-logical-optimization/infer-window-group-limit.md](../wiki/03-logical-optimization/infer-window-group-limit.md) — `Filter(rank ≤ N) over Window(rank-like-fn)` 패턴에 `WindowGroupLimit` 노드 삽입. 두 갈래 출력 (RowNumber + 빈 partitionSpec → Limit / 그 외 → WindowGroupLimit) + limit ≤ 0 dead-code + 동반 5규칙 batch + Catalyst defaultBatches 부재 (SparkOptimizer 전용). case B 적용 + case C 미적용(AVG는 rank-like 아님) 양면 검증 (verified, InferWindowGroupLimit.scala + SparkOptimizer.scala 소스).
- [wiki/03-logical-optimization/like-simplification.md](../wiki/03-logical-optimization/like-simplification.md) — LIKE 'pattern'을 5종 단순 표현식 (StartsWith/EndsWith/Contains/EqualTo + prefix%suffix 결합)으로 재작성. 5개 정규식 패턴 + simplifyLike 5분기 + prefix%suffix Length invariant. case A/B/C 모두 `LIKE '서울%'` → `StartsWith` 관찰 (verified, 얕게).
- [wiki/03-logical-optimization/predicate-pushdown.md](../wiki/03-logical-optimization/predicate-pushdown.md) — Filter를 Join/Scan 하위로 밀어넣는 dispatcher (CombineFilters + PushPredicateThroughNonJoin + PushPredicateThroughJoin orElse 체인). PushPredicateThroughJoin.split() 3-튜플 분류. **Catalyst defaultBatches vs SparkOptimizer 비교 표 — 다른 03 페이지가 참조하는 hub** (verified, Optimizer.scala + SparkOptimizer.scala 소스).

## 04 — Physical Planning

- [wiki/04-physical-planning/join-strategy-hints.md](../wiki/04-physical-planning/join-strategy-hints.md) — BROADCAST/MERGE/SHUFFLE_HASH/SHUFFLE_REPLICATE_NL hints + auto-broadcast threshold (verified, JoinSelection 소스).
- [wiki/04-physical-planning/ensure-requirements.md](../wiki/04-physical-planning/ensure-requirements.md) — Exchange/Sort 자동 삽입 rule. 4 메커니즘 (단일 자식 distribution check / multi-child co-partitioning / ordering check / reorderJoinKeys). 세 case의 7개 Exchange를 메커니즘별로 매핑 (verified, EnsureRequirements.scala 소스).
- [wiki/04-physical-planning/partitioning-compatibility.md](../wiki/04-physical-planning/partitioning-compatibility.md) — `Partitioning.satisfies(Distribution)` 호환성 매트릭스 (Distribution 6종 × Partitioning 6종, 36 셀 라인 인용). HashPartitioningLike의 subset/superset 분기 + RangePartitioning prefix 매칭 + 쿼리 C "structural 한계" 일반화 (verified, partitioning.scala 소스).
- [wiki/04-physical-planning/strategy-framework.md](../wiki/04-physical-planning/strategy-framework.md) — Spark 3.5.6의 physical planner가 Armbrust 2015 논문의 "candidates → cost 비교" 모델이 아니라 "단일 plan + 휴리스틱 ladder" 모델로 동작함을 검증. 결정적 증거 2건 blockquote (`QueryPlanner.scala:50-51` TODO "ONLY ONE PLAN IS RETURNED EVER" + `SparkPlanner.scala:62-66` TODO prunePlans no-op) + cost 세 갈래 분리 (strategy 내부 휴리스틱 / logical CBO / AQE 사후 재선택) (verified, QueryPlanner.scala + SparkPlanner.scala + SparkStrategies.scala 소스).
- [wiki/04-physical-planning/coalesce-repartition-hints.md](../wiki/04-physical-planning/coalesce-repartition-hints.md) — COALESCE/REPARTITION/REPARTITION_BY_RANGE/REBALANCE + shuffle/file partition configs.
- [wiki/04-physical-planning/storage-partition-join.md](../wiki/04-physical-planning/storage-partition-join.md) — V2 DataSource partitioning으로 Exchange 제거.
- [wiki/04-physical-planning/hash-aggregate-partial-final.md](../wiki/04-physical-planning/hash-aggregate-partial-final.md) — GROUP BY의 partial → final 2단계 HashAggregate 패턴 (tentative).
- [wiki/04-physical-planning/batch-scan-iceberg.md](../wiki/04-physical-planning/batch-scan-iceberg.md) — Iceberg V2 DataSource BatchScan 연산자, filter pushdown 관찰 (tentative, Iceberg 소스 ingest 대기).
- [wiki/04-physical-planning/window-exec.md](../wiki/04-physical-planning/window-exec.md) — Window function 물리 실행, Window strategy (tentative, L3 일부 SparkStrategies.scala:641-655 인용).
- [wiki/04-physical-planning/window-group-limit.md](../wiki/04-physical-planning/window-group-limit.md) — Top-N per Group 최적화, Partial/Final 분할 (tentative, L3 일부 SparkStrategies.scala:657-667 인용; 짝이 되는 Catalyst rule은 [03/infer-window-group-limit](../wiki/03-logical-optimization/infer-window-group-limit.md) verified 작성됨).

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
