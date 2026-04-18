---
stage: 05-cost-and-aqe
title: Leveraging Statistics for Query Planning
spark_versions: ["4.1"]
sources:
  - raw/spark-docs/2026-04-spark-4.1.1-sql-performance-tuning.md
last_verified: 2026-04-19
confidence: tentative
related:
  - 00-overview/execution-pipeline.md
  - 05-cost-and-aqe/aqe-overview.md
  - 04-physical-planning/join-strategy-hints.md
configs: []
apis:
  - DESCRIBE EXTENDED
  - EXPLAIN COST
  - DataFrame.explain(mode="cost")
  - ANALYZE TABLE
tags: [statistics, cbo, cost-estimate, runtime-stats, analyze-table]
---

## L1. What & Why

Spark의 비용 기반 결정 — join 전략 선택, join 재정렬, broadcast 유발 —
은 각 plan 노드에 대한 optimizer의 row 개수와 크기 추정치에 의존한다.
그 추정치는 세 가지 원천에서 온다: underlying data source(Parquet
footer 통계 등), catalog(`ANALYZE TABLE`로 채워진 Hive Metastore 값),
runtime(AQE가 shuffle을 materialize 하면서 쿼리 실행 중 계산).

통계가 누락되거나 오래되면 진단이 어려운 plan 성능 저하가 발생한다 —
plan이 "그냥 sort-merge로 실행됨"이 되거나 사실 1 GB짜리 dimension
테이블에 "broadcast timeout"이 나는 식이다. 이 페이지는 Spark가 무엇을
알고 있다고 생각하는지를 사용자가 들여다볼 수 있는 inspection API에
관한 것이다.

## L2. Inputs & Outputs

입력: 테이블(catalog 또는 file 기반), 또는 실행 중인 쿼리.

출력: optimization과 physical planning 단계에서 각 plan 노드에 부착
되는 `Statistics` 레코드. 포함 필드:

- Row 개수 (`rowCount`).
- 바이트 크기 (`sizeInBytes`).
- 컬럼별 통계: min, max, null 개수, distinct 개수, histogram
  (가용한 경우).

이 필드들은 단계 03의 Catalyst optimizer(join 재정렬 등 CBO 규칙)와
단계 04의 physical planner(join 전략 선택)가 소비한다.

## L3. Key Mechanisms

> 📝 docs 요약 수준. 소스 코드 ingest 대기 중
> (`sql/catalyst/src/main/scala/org/apache/spark/sql/catalyst/plans/logical/statsEstimation/`,
> `Statistics.scala`, `CostBasedJoinReorder.scala`).

4.1.1 docs에 따른 세 가지 통계 원천:

### Data source statistics

Parquet 파일은 footer에 row-group별 min/max와 row 개수를 담고 있다.
ORC도 stripe 수준 통계를 비슷하게 노출한다. Spark는 scan 시점에 이를
직접 읽는다 — 파일 기반 source는 이 정보를 쓰기 위해 `ANALYZE TABLE`이
필요하지 않다. writer가 관리하므로, 데이터가 오래되어야만 통계도 오래
된다.

### Catalog statistics

metastore(Hive Metastore, Unity Catalog 등)에 저장되며 다음 명령으로
채워진다:

```sql
ANALYZE TABLE my_table COMPUTE STATISTICS;          -- row 개수 + 크기
ANALYZE TABLE my_table COMPUTE STATISTICS NOSCAN;   -- 크기만 (빠름)
ANALYZE TABLE my_table COMPUTE STATISTICS FOR COLUMNS c1, c2;  -- 컬럼 수준
```

이것이 Hive-managed 테이블에서 `spark.sql.autoBroadcastJoinThreshold`
메커니즘이 참조하는 통계이며, 원본 docs 각주는
[04-physical-planning/join-strategy-hints.md]에 정리되어 있다. 기본적으로
stale이다 — insert로 갱신되지 않으므로 수동 재실행 필요.

### Runtime statistics

쿼리 실행 중 AQE가 shuffle 출력을 materialize 하면서 계산된다.
[aqe-overview.md]의 AQE 재계획 규칙들(coalesce, SMJ → BHJ, skew 탐지)이
모두 runtime 통계를 소비한다. 쿼리마다 항상 최신이 보장되는 유일한
통계 원천.

### Inspection APIs

| API | 보여주는 것 |
|---|---|
| `DESCRIBE EXTENDED <table>` | 저장된 catalog 통계: row 개수, 크기, 수집된 컬럼 수준 통계 |
| `EXPLAIN COST <query>` 또는 `DataFrame.explain(mode="cost")` | plan 노드별 optimizer 추정치 |
| SQL UI → SQL 탭 → Details | 쿼리 진행 중 plan 옆에 `Statistics(..., isRuntime=true)`로 표시되는 런타임 통계 |

## L4. Performance Levers

1. **`ANALYZE TABLE`은 Hive-managed 테이블의 auto-broadcast 전제 조건이다.**
   이 없으면 Spark가 테이블 크기를 모르므로
   `spark.sql.autoBroadcastJoinThreshold`가 발동할 수 없다.
   [04-physical-planning/join-strategy-hints.md]의 원본 docs 각주 참고.

2. **Parquet/ORC footer 통계는 공짜다.** 파일 기반 테이블은 사용자
   비용 없이 기본 통계 혜택을 받는다 — Spark가 scan 시점에 읽는다.
   다만 컬럼 통계(distinct 개수, histogram)는 여전히 catalog 쪽에서
   `ANALYZE TABLE ... FOR COLUMNS`가 필요.

3. **`EXPLAIN COST`는 비침습적 진단 도구다.** plan이 의외의 동작을
   보이면 — "왜 broadcast 안 되는가?" — `EXPLAIN COST`로 각 노드가
   produce 한다고 Spark가 생각하는 값을 본다. `Statistics(size=1.0 B)`
   라인은 "추정치 없음"을 의미.

4. **Runtime 통계는 catalog staleness를 우회한다.** AQE의 런타임
   재계획은 `ANALYZE TABLE`의 최신성에 의존하지 않는다. catalog 통계를
   유지하기 운영상 어려운 워크로드에서 AQE 활성화가 타당한 실전적
   이유.

5. **`spark.sql.cbo.enabled`는 기본 off.** CBO(통계를 이용한 규칙 기반
   join 재정렬)는 AQE와 별개의 오래된 기능이며, 4.1.1 "Leveraging
   Statistics" 섹션은 이에 대한 암시적 언급 이상은 다루지 않는다.
   CBO 활성화는 별도의 의사결정이며 고유의 트레이드오프가 있다 — 이
   페이지의 범위 밖.

## L5. How to Observe

> TODO: `queryExecution.executedPlan` 또는 SQL UI 관찰 사례 필요.

저장된 catalog 통계:

```sql
DESCRIBE EXTENDED my_table;
-- "Statistics" 섹션에서 rowCount, sizeInBytes 확인
ANALYZE TABLE my_table COMPUTE STATISTICS NOSCAN;
-- NOSCAN은 크기만 채움 (파일 메타만 읽으므로 빠름)
```

Optimizer가 추정한 plan 노드별 비용:

```sql
EXPLAIN COST SELECT c.name, SUM(o.amount)
FROM customers c JOIN orders o ON c.id = o.customer_id
GROUP BY c.name;
```

출력에는 각 plan 노드에 대한 `Statistics(sizeInBytes=..., rowCount=...)`가
포함된다. `Statistics(sizeInBytes=1.0 B)` 라인은 "추정치 없음"을
의미.

DataFrame 동등 구문:

```python
df.explain(mode="cost")
```

SQL UI에서 runtime 통계 보기:

- Spark UI → SQL/DataFrame 탭 → 실행 중/완료된 쿼리 클릭.
- 특정 plan 노드의 "Details" 펼치기.
- `Statistics(sizeInBytes=..., rowCount=..., isRuntime=true)`를 찾는다.
  `isRuntime=true` 플래그가 런타임 통계와 정적 통계를 구분.

## Version notes

- "Leveraging Statistics" 섹션은 4.1.1 docs에만 존재 — 3.5.6에는 대응
  섹션 없음. 하위 API들(`DESCRIBE EXTENDED`, `EXPLAIN COST`,
  `ANALYZE TABLE`)은 모두 4.1 이전부터 존재: `EXPLAIN COST`는 Spark
  3.0+, `ANALYZE TABLE`은 Spark 2.x+. 4.1 docs는 이들을 하나의 사용자용
  서술로 묶은 것.
- SQL UI의 `isRuntime=true` 태그: AQE 시대의 기능, AQE가 기본 on이
  된 3.2+부터 존재.
- 이 페이지는 4.1 증거만 주장. 3.5에서의 API 동작을 확정하려면 3.5
  `sql-ref-syntax-aux-describe-table.md` 등 관련 docs 또는 소스 ingest
  필요.
