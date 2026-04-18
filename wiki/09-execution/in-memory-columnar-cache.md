---
stage: 09-execution
title: In-Memory Columnar Cache
spark_versions: ["3.5", "4.1"]
sources:
  - raw/spark-docs/2026-04-spark-3.5.6-sql-performance-tuning.md
  - raw/spark-docs/2026-04-spark-4.1.1-sql-performance-tuning.md
last_verified: 2026-04-19
confidence: tentative
related:
  - 00-overview/execution-pipeline.md
configs:
  - spark.sql.inMemoryColumnarStorage.compressed
  - spark.sql.inMemoryColumnarStorage.batchSize
apis:
  - DataFrame.cache
  - DataFrame.unpersist
  - spark.catalog.cacheTable
  - spark.catalog.uncacheTable
tags: [cache, memory, columnar, storage, tungsten]
---

## L1. What & Why

Spark SQL은 DataFrame과 등록된 테이블을 columnar in-memory 포맷으로
캐시한다. 이는 row 지향인 `RDD.cache()` 경로와 구별된다. Columnar
layout은 컬럼별 압축 코덱 선택(dictionary, RLE, integer packing)과
컬럼 부분집합 scan을 가능하게 한다 — 캐시된 20개 컬럼 중 2개만 쓰는
쿼리는 그 2개만 읽는다. 이것이 중요한 이유는 동일 데이터셋에 대해
서로 다른 projection으로 반복 발행되는 analytic 쿼리가 많기 때문이다.
compute 비용을 한 번 지불하고 이후 쿼리를 메모리에서 서빙하면 실행
시간에서 큰 이득이 생긴다 — executor 메모리만 여유 있다면.

## L2. Inputs & Outputs

입력: `.cache()` / `spark.catalog.cacheTable(name)` 호출 후 실행을
강제하는 action으로 materialize 된 DataFrame 또는 테이블 참조.

출력: executor별로 `CachedBatch` columnar 포맷에 저장된 cached
partition들. 컬럼별로 압축된 배열. 이후 쿼리 실행 시, cached relation이
physical plan에서 원본 scan을 대체한다. 명시적 `.unpersist()` /
`uncacheTable()` 또는 메모리 압박 시 LRU로 eviction 가능.

## L3. Key Mechanisms

> 📝 docs 요약 수준. 소스 코드 ingest 대기 중
> (`sql/core/src/main/scala/org/apache/spark/sql/execution/columnar/InMemoryRelation.scala`,
> `CachedBatch.scala`, `ColumnarBatch.scala`).

docs가 설명하는 columnar layout 기본:

- 캐시된 데이터는 row 배치(기본 10,000 row/batch)로 구성되며, 각 배치는
  컬럼 지향 배열로 저장된다.
- 각 컬럼은 독립적으로 압축 코덱이 배정되며, 코덱은 해당 컬럼의
  통계에 기반해 선택된다(`compressed = true`일 때).
- cached relation을 scan할 때는 필요한 컬럼의 배열만 읽음 — 전체 row
  버퍼는 아님.

캐싱을 지배하는 두 설정:

| Config | Default | 의미 |
|---|---|---|
| `spark.sql.inMemoryColumnarStorage.compressed` | true | 데이터 통계에 기반해 컬럼별 압축 코덱 자동 선택 |
| `spark.sql.inMemoryColumnarStorage.batchSize` | 10000 | columnar batch당 row 개수 |

docs 기준 API 진입점은 두 가지:

- `spark.catalog.cacheTable("name")` — 이름으로 등록된 테이블/뷰를
  캐시. 대응 unpersist: `spark.catalog.uncacheTable("name")`.
- `dataFrame.cache()` — 특정 DataFrame 인스턴스를 캐시. 대응
  unpersist: `dataFrame.unpersist()`.

캐시는 lazy다: `.cache()` 호출 자체가 아니라, 그 뒤의 첫 action
시점에 캐시가 채워진다.

## L4. Performance Levers

1. **`batchSize = 10000`은 압축 품질과 OOM 위험의 트레이드오프다.** 큰
   batch는 압축이 좋아지고(컬럼 블록당 값이 많아짐) vectorized-reader
   오버헤드를 amortize 한다. 작은 batch는 wide row(컬럼이 많음 →
   batch당 총 바이트 증가) 또는 빡빡한 메모리에서 더 안전.

2. **`compressed = true`가 기본으로 적절**하다. 측정 결과 압축 CPU
   오버헤드가 메모리 절감을 초과하는 경우가 아니라면 유지. 그런
   경우는 드물다 — analytic 컬럼은 대체로 잘 압축된다.

3. **캐시는 lazy — 의존하려면 materialization을 강제하라.** 흔한
   버그: `.cache()` 직후 캐시가 warm이라고 가정하고 비싼 쿼리를
   실행. `.cache()` 후의 첫 action은 compute 비용과 cache-build 비용을
   모두 지불한다. pre-warm:
   ```python
   df.cache()
   df.count()   # materialization 트리거
   ```

4. **`cacheTable(name)`과 `cache()`는 서로 다른 대상을 캐시한다.**
   `cacheTable`은 등록된 view/테이블의 logical plan을 캐시하고,
   `cache()`는 특정 DataFrame 인스턴스를 캐시한다. 같은 물리 데이터가
   두 경로로 접근되면, Spark의 plan 동등성 판정이 일치시키지 못하는
   한 캐시 상태를 공유하지 않는다.

5. **메모리를 명시적으로 해제하려면 unpersist.** LRU eviction도
   동작하지만, 명시적 `.unpersist()`가 예측 가능하고 일회성 쿼리를
   위해 아직 뜨거운 캐시가 밀려 나가는 상황을 피한다.

## L5. How to Observe

> TODO: `queryExecution.executedPlan` 또는 SQL UI 관찰 사례 필요.

```python
df.cache()
df.count()  # materialization 트리거
spark.catalog.isCached("my_table")  # 테이블로 등록된 경우
df.storageLevel  # StorageLevel 확인 (예: MEMORY_AND_DISK)
```

Spark UI → Storage 탭: executor별 메모리 사용량과 함께 cached RDD를
나열.

```python
# physical plan 확인 — cached relation은 InMemoryTableScan으로 나타남
df.queryExecution.executedPlan
# 기대: InMemoryTableScan [col1, col2]
#          +- InMemoryRelation [col1, col2, col3, ...]
#             +- <original scan>
```

## Version notes

- `spark.sql.inMemoryColumnarStorage.compressed`: since 1.0.1.
- `spark.sql.inMemoryColumnarStorage.batchSize`: since 1.1.1.
- 설정 방식 문구 차이 (docs 의역, 원문은 `sources:`의 raw/ 참조):
  - 3.5.6:
    > in-memory 캐시 설정은 `SparkSession`의 `setConf` 메서드 또는 SQL `SET key=value`로 가능.
  - 4.1.1:
    > in-memory 캐시 설정은 `spark.conf.set` 또는 SQL `SET key=value`로 가능.
  동작상 차이 없음 — `setConf`와 `spark.conf.set`은 등가 진입점.
- 섹션 제목 변경: "Caching Data In Memory" (3.5.6) → "Caching Data"
  (4.1.1). 동일 내용.
