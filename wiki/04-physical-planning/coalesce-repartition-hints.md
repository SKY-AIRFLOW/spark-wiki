---
stage: 04-physical-planning
title: Coalesce, Repartition, and Rebalance Hints
spark_versions: ["3.5", "4.1"]
sources:
  - raw/spark-docs/2026-04-spark-3.5.6-sql-performance-tuning.md
  - raw/spark-docs/2026-04-spark-4.1.1-sql-performance-tuning.md
last_verified: 2026-04-19
confidence: tentative
related:
  - 00-overview/execution-pipeline.md
  - 04-physical-planning/join-strategy-hints.md
  - 05-cost-and-aqe/aqe-overview.md
configs:
  - spark.sql.shuffle.partitions
  - spark.sql.files.maxPartitionBytes
  - spark.sql.files.openCostInBytes
  - spark.sql.files.minPartitionNum
  - spark.sql.files.maxPartitionNum
  - spark.sql.sources.parallelPartitionDiscovery.threshold
  - spark.sql.sources.parallelPartitionDiscovery.parallelism
apis:
  - SQL /*+ COALESCE */
  - SQL /*+ REPARTITION */
  - SQL /*+ REPARTITION_BY_RANGE */
  - SQL /*+ REBALANCE */
  - DataFrame.coalesce
  - DataFrame.repartition
  - DataFrame.repartitionByRange
tags: [hint, partition, repartition, coalesce, rebalance, shuffle]
---

## L1. What & Why

이 hint들은 plan의 특정 지점에서 Spark의 기본 partition 수를 사용자가
직접 오버라이드하게 해준다 — shuffle 없이 partition을 병합(`COALESCE`),
shuffle로 재분배(`REPARTITION`, `REPARTITION_BY_RANGE`), 또는 AQE 기반
크기 보정을 동반하는 출력 partition 재조정(`REBALANCE`). 이들이
존재하는 이유는 기본값 `spark.sql.shuffle.partitions = 200`이 대부분
데이터셋에 맞지 않기 때문이고, 출력 시점의 small-file vs. straggler-task
트레이드오프가 shuffle 설정만으로는 표현되지 않기 때문이다.

## L2. Inputs & Outputs

입력: hint가 붙은 LogicalPlan 서브트리. SQL `/*+ ... */` 또는
DataFrame `.hint(...)`에서 오거나, 동등하게 `.repartition()` /
`.coalesce()` / `.repartitionByRange()` 호출에서 생성된다.

출력: 아래 중 하나의 operator 노드가 삽입된 LogicalPlan. 이후 단계에서
physical plan 노드로 lowering 된다:

| Hint / API | Logical operator | Physical operator | Shuffle? |
|---|---|---|---|
| `COALESCE(n)` / `.coalesce(n)` | `Repartition(numPartitions, shuffle=false)` | `CoalesceExec` | 없음 |
| `REPARTITION(n)` / `.repartition(n)` | `Repartition(numPartitions, shuffle=true)` | `ShuffleExchangeExec` | 있음 |
| `REPARTITION(cols)` / `.repartition(cols)` | `RepartitionByExpression` | `ShuffleExchangeExec` (hash) | 있음 |
| `REPARTITION_BY_RANGE(cols)` | `RepartitionByExpression` (range) | `ShuffleExchangeExec` (range) | 있음 |
| `REBALANCE` / `REBALANCE(cols)` | `RebalancePartitions` | `ShuffleExchangeExec` + AQE rebalance rule | 있음 |

## L3. Key Mechanisms

> 📝 docs 요약 수준. 소스 코드 ingest 대기 중
> (`sql/catalyst/src/main/scala/org/apache/spark/sql/catalyst/analysis/ResolveHints.scala`,
> `sql/core/src/main/scala/org/apache/spark/sql/execution/exchange/ShuffleExchangeExec.scala`,
> `AdaptiveSparkPlanExec`의 AQE rebalance 처리).

docs가 규정하는 hint parameter 형태:

- `COALESCE(n)`: 정수 하나, partition 개수만.
- `REPARTITION(n)`, `REPARTITION(col)`, `REPARTITION(n, col)`,
  `REPARTITION` (인자 없음): 개수, 컬럼, 둘 다, 또는 둘 다 없음.
- `REPARTITION_BY_RANGE(col)`, `REPARTITION_BY_RANGE(n, col)`: 컬럼
  필수, partition 개수 선택적. 컬럼 범위에 따라 partition 간 데이터를
  정렬한다 — downstream consumer가 range order로부터 이득을 보는
  경우에 유용(예: sort-merge 친화적 쓰기).
- `REBALANCE`, `REBALANCE(n)`, `REBALANCE(col)`, `REBALANCE(n, col)`:
  모든 인자 형태 지원. `REPARTITION`과 달리 AQE가 runtime 크기 통계에
  기반해 결과 partition을 split/merge 할 수 있으며, 이는 이 hint를
  [05-cost-and-aqe/aqe-overview.md]의 rebalance-skew rule과 결합시킨다.

hint 없이 partition 모양을 결정하는 보조 설정들:

- `spark.sql.shuffle.partitions` (200): shuffle 출력 partition 기본 개수.
- `spark.sql.files.maxPartitionBytes` (128 MB): file-scan partition당
  목표 바이트 수. leaf에서의 병렬성 지배.
- `spark.sql.files.openCostInBytes` (4 MB): 작은 파일을 함께 묶기 위해
  사용하는 파일 열기 비용 추정. 값이 클수록 큰 partition 선호.
- `spark.sql.files.minPartitionNum` (default parallelism): split file
  partition 수의 하한 제안. since 3.1.0.
- `spark.sql.files.maxPartitionNum` (none): 상한 제안. since 3.5.0.
  설정 시 초기 split 개수가 이를 초과하면 Spark가 재조정.
- `spark.sql.sources.parallelPartitionDiscovery.threshold` (32),
  `parallelism` (10000): 파일 목록 조회를 Spark job으로 병렬화할지
  순차 수행할지 제어.

## L4. Performance Levers

1. **`shuffle.partitions = 200`은 보통 틀린 값이다.** 이 기본값은 작은
   데모에 맞춘 타협이다. 프로덕션에서는 클러스터에 맞춰 튜닝하거나,
   AQE의 post-shuffle coalescing에 맡겨 런타임에 크기를 맞추도록
   한다. [05-cost-and-aqe/aqe-overview.md] 참고.

2. **출력 쓰기에는 REPARTITION보다 REBALANCE.** `REBALANCE`는 AQE의
   rebalance-skew rule(`spark.sql.adaptive.optimizeSkewsInRebalancePartitions.enabled`)
   과 통합되어 skew 출력 partition을 런타임에 분할할 수 있다.
   `REPARTITION`은 partition 개수를 고정하고 이 적응성을 포기한다.

3. **`files.maxPartitionBytes`는 scan 단계 task 개수를 결정한다.** 소수의
   매우 큰 파일에 대한 CPU-bound scan에는 값을 낮춰 병렬성을 높이고,
   많은 작은 파일에 대한 I/O-bound scan에는 값을 올려 task 수를 줄이고
   vectorized-reader의 amortization을 개선한다.

4. **COALESCE는 shuffle 하지 않지만 upstream을 병목화할 수 있다.**
   200 → 10 partition으로 coalesce 한다는 건 downstream task 10개가
   각각 20× 데이터를 처리한다는 뜻이다. coalesce가 독립 operator로
   남으면 upstream stage는 여전히 200 task로 실행되지만, shuffle barrier
   없이 fuse 되면 upstream 병렬성도 10으로 떨어진다. coalesce가 독립
   operator인지 upstream을 collapse 하는지는 `executedPlan`에서
   확인하라.

## L5. How to Observe

> TODO: `queryExecution.executedPlan` 또는 SQL UI 관찰 사례 필요.

```python
df.hint("repartition", 50).queryExecution.executedPlan
# 기대: Exchange hashpartitioning(..., 50) 형태

df.coalesce(5).queryExecution.executedPlan
# 기대: Coalesce 5 (Exchange 없음)

df.hint("rebalance").queryExecution.executedPlan
# 기대: Exchange rebalancepartitioning(...) (AQE rule 효과도 포함)
```

SQL 동등 구문:

```sql
SELECT /*+ COALESCE(3) */ * FROM t;
SELECT /*+ REPARTITION(3, c) */ * FROM t;
SELECT /*+ REPARTITION_BY_RANGE(3, c) */ * FROM t;
SELECT /*+ REBALANCE(c) */ * FROM t;
```

## Version notes

- `spark.sql.files.minPartitionNum`: since 3.1.0.
- `spark.sql.files.maxPartitionNum`: since 3.5.0 — partition 설정 중
  가장 최신.
- `REBALANCE` hint: Spark 릴리즈 히스토리상 3.2.0 도입; ingest된 두
  docs(3.5.6, 4.1.1)는 이미 정착된 기능으로 서술.
- 3.5.6은 partition 설정을 "Other Configuration Options"에서 join
  설정과 함께 묶어 서술; 4.1.1은 "Tuning Partitions"라는 전용
  섹션으로 분리. 동일한 설정, 구성만 변경.
- `spark.sql.files.minPartitionNum`의 docs 문구는 버전 간 동일하지만,
  4.1.1에서 fallback이 `spark.sql.leafNodeDefaultParallelism`임을
  명확히 함.
