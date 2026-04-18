# 페이지 템플릿 참고

이 Wiki에서 올바르게 구조화된 페이지들의 예시입니다. Depth Ladder
(L1-L5)는 모든 단계 페이지에서 필수입니다. 이 예시들을 출발점으로
복사해 사용하세요.

---

## 템플릿: 단계(Stage) 페이지

경로: `wiki/03-logical-optimization/predicate-pushdown.md`

```markdown
---
stage: 03-logical-optimization
title: Predicate Pushdown
spark_versions: ["3.2", "3.3", "3.4", "3.5"]
sources:
  - raw/spark-source/sql/catalyst/org/apache/spark/sql/catalyst/optimizer/Optimizer.scala
  - raw/spark-docs/2024-11-sql-performance-tuning.md
  - raw/books/high-performance-spark-ch5.md
last_verified: 2026-04-19
confidence: verified
related:
  - 03-logical-optimization/rule-based-rules.md
  - 04-physical-planning/strategies.md
  - 09-execution/vectorized-reader.md
  - _probes/explain-modes.md
configs:
  - spark.sql.optimizer.excludedRules
apis:
  - DataFrame.filter
  - WHERE clause
tags: [optimizer, catalyst, predicate]
---

## L1. What & Why

Predicate pushdown은 Catalyst의 규칙 배치로, 필터 조건을 논리적
플랜에서 데이터 소스에 가능한 한 가까이 이동시킵니다. 이 규칙이
없다면 Spark는 전체 데이터셋을 메모리로 읽은 후에 필터링하게 되어
I/O와 shuffle 대역폭을 낭비하게 됩니다. 이 규칙은 논리적 플랜 트리를
변환해 Filter 노드를 Project, Join, Aggregate 노드 아래로 의미상
안전한 위치에 밀어 넣습니다.

## L2. Inputs & Outputs

입력: Filter 노드가 트리의 임의 위치에 있는 Analyzed LogicalPlan.

출력: Filter 노드가 가능한 한 낮게 이동된 Optimized LogicalPlan.
데이터 소스 컬럼을 참조하는 조건자는 Relation에 pushed filter로
부착됩니다 (소스가 PrunedFilteredScan을 지원하는 경우).

API 경계에서의 타입:
  LogicalPlan (추상) 및 Filter(condition, child) 노드
  Relation은 `pushedFilters: Array[Filter]`를 가진
  PrunedFilteredScan이 될 수 있음

## L3. Key Mechanisms

Catalyst optimizer는 "Operator Optimization" 배치에서 관련 규칙들을
실행합니다:

- **PushDownPredicates**: Filter를 Project, Union, Aggregate(필터가
  grouping 컬럼만 참조할 때), Join(조건이 한쪽만 참조할 때) 아래로
  밀어냄.
- **CombineFilters**: 인접한 Filter들을 하나로 병합하여 PushDownPredicates가
  하나의 단위로 이동시킬 수 있게 함.
- **InferFiltersFromConstraints**: join 조건으로부터 새로운 필터를
  합성 (예: `a.x = b.x AND b.x > 5`에서 `a.x > 5`를 유도).
- **PushPredicateThroughNonJoin** / **PushPredicateThroughJoin**:
  주된 두 개의 push-through 규칙. 3.x에서 분리됨.

Data Source V2 (DSv2)의 경우, push된 조건자들은 커넥터에
`SupportsPushDownFilters.pushFilters`를 통해 전달되고, 커넥터는
어떤 필터를 네이티브로 처리했고 어떤 필터를 Spark가 여전히
재평가해야 하는지 응답합니다.

## L4. Performance Levers

1. **커넥터 지원이 가장 중요합니다.** Parquet 소스는 풍부한 필터
   pushdown을 지원합니다 (row-group 수준의 min/max 프루닝 포함).
   JDBC 소스는 SQL WHERE를 push합니다. 일반 RDD 소스는 아무것도
   push하지 않습니다. Song의 환경에서는 Iceberg v2 커넥터가 파티션 및
   통계 기반 file-skipping을 포함한 공격적인 필터 pushdown을 수행합니다.

2. **UDF는 pushdown을 차단합니다.** 비결정적이거나 pushdown-safe하지
   않은 UDF가 포함된 필터는 push되지 **않습니다**. 이는 조용한
   절벽입니다: `df.filter(myUdf(col))`은 `df.filter(col > 5)`보다
   수십 배 많은 데이터를 읽을 수 있습니다. `queryExecution.optimizedPlan`
   으로 검증할 것.

3. **`spark.sql.optimizer.excludedRules`로 특정 규칙을 디버깅용으로
   비활성화할 수 있습니다.** 프로덕션에서는 거의 쓰이지 않지만, "왜
   내 필터가 push되지 않았는가"를 진단할 때 필수적입니다.

4. **DPP (Dynamic Partition Pruning)**는 관련되지만 별도의
   메커니즘으로, AQE를 통해 단계 05에서 활성화되며, join 반대편의
   런타임 통계에 기반해 동적으로 필터를 push할 수 있습니다.
   [05-cost-and-aqe/dynamic-partition-pruning.md] 참고.

## L5. How to Observe

```python
df = spark.sql("""
    SELECT c.name, o.amount
    FROM customers c JOIN orders o ON c.id = o.customer_id
    WHERE o.date > '2026-01-01' AND c.country = 'KR'
""")

# 최적화 이후 필터가 어디에 자리 잡았는지 확인
print(df.queryExecution.optimizedPlan)

# 실제로 소스가 무엇을 받았는지 확인
print(df.queryExecution.executedPlan)
# 다음과 같은 줄을 찾을 것:
#   +- *(1) Filter (isnotnull(date#47))
#   +- *(1) ColumnarToRow
#      +- FileScan parquet [...] PushedFilters: [IsNotNull(date), GreaterThan(date,2026-01-01)]
```

`FileScan` 노드의 `PushedFilters: [...]` 목록이 소스가 실제로 처리한
것에 대한 ground truth입니다. 예상한 필터가 없다면 소스가 그것을
지원하지 않았거나 UDF 절벽이 발동한 것입니다.

전체 워크플로우: [_probes/explain-modes.md]
```

---

## 템플릿: Probe 페이지

경로: `wiki/_probes/reading-codegen-output.md`

```markdown
---
stage: _probes
title: WholeStageCodegen이 생성한 Java 읽기
spark_versions: ["3.0", "3.1", "3.2", "3.3", "3.4", "3.5"]
sources:
  - raw/spark-source/sql/core/.../WholeStageCodegenExec.scala
  - raw/blog/2023-databricks-codegen-deep-dive.md
last_verified: 2026-04-19
confidence: verified
related:
  - 06-codegen/wholestage-codegen.md
  - 06-codegen/generated-code-anatomy.md
tags: [probe, codegen, debugging]
---

## L1. What & Why

WholeStageCodegen은 런타임에 실제 Java 소스 코드를 생성하고 Janino로
컴파일합니다. 이 생성된 코드를 읽는 것이 Executor에서 Spark가 실제로
실행하는 것을 *직접* 볼 수 있는 유일한 방법입니다. Physical plan은
추상이고, 생성된 Java가 현실입니다. 많은 성능 미스터리들 (왜 내
필터가 느린가? 내 UDF가 codegen되었는가?) 이 생성된 Java를 읽음으로써
해결됩니다.

## L2. Inputs & Outputs

입력: physical plan에 `WholeStageCodegenExec` 노드가 포함된
DataFrame (`executedPlan`에서 `*` 접두사가 붙은 연산자로 표시됨).

출력: Java 소스 문자열. WholeStageCodegen 경계당 하나. 각 경계는
`BufferedRowIterator`를 상속하는 하나의 생성된 클래스가 됩니다.

## L3. Key Mechanisms

```python
from pyspark.sql.functions import col, sum as spark_sum

df = (spark.table("orders")
      .filter(col("amount") > 100)
      .groupBy("customer_id")
      .agg(spark_sum("amount")))

# 생성된 코드 블록 모두 출력
df.queryExecution.debug.codegen()
```

블록당 출력 구조:
```
Found N WholeStageCodegen subtrees.
== Subtree 1 / N (maxMethodCodeSize:X; maxConstantPoolSize:Y) ==
*(1) HashAggregate(...)
+- Exchange hashpartitioning(customer_id, ...)
   +- *(2) Filter ...
      +- *(2) Scan parquet ...

Generated code:
/* 001 */ public Object generate(Object[] references) {...}
/* 002 */ final class GeneratedIteratorForCodegenStage1 ...
/* ... */
```

각 subtree가 갖는 것:
- **플랜 요약** (하나의 메서드로 fused된 트리 부분)
- **maxMethodCodeSize** (JVM 제한은 메서드당 8KB 바이트코드;
  초과 시 codegen fallback 강제)
- **maxConstantPoolSize** (JVM 제한은 클래스당 65k 상수)
- **라인 번호가 매겨진 생성된 Java 소스**

## L4. Performance Levers

- **`spark.sql.codegen.wholeStage`** (기본값 true): 마스터 스위치
- **`spark.sql.codegen.maxFields`** (기본값 100): 출력 컬럼 수가
  이보다 많은 연산자는 codegen에서 떨어져 나감
- **`spark.sql.codegen.methodSplitThreshold`**: JVM의 8KB 바이트코드
  한도 아래로 유지하기 위해 긴 메서드를 자동 분할
- **`spark.sql.codegen.hugeMethodLimit`**: fallback을 강제하기 전의
  절대 상한

생성된 코드에서 fallback의 징후:
- 플랜에 연산자가 `*` 접두사 **없이** 나타남
- `InterpretedProjection`, `InterpretedMutableProjection`이 보임
- 로그에 `[WARN] WholeStageCodegenExec: Whole-stage codegen
  disabled for plan`이 찍힘

## L5. How to Observe

최소 재현 예시:

```python
# Spark 셸 또는 노트북
from pyspark.sql.functions import col

df = spark.range(1000).filter(col("id") > 500).selectExpr("id * 2 as doubled")
df.queryExecution.debug.codegen()

# 또는 신중히 읽기 위해 파일로 덤프:
with open("codegen_output.java", "w") as f:
    import sys
    from io import StringIO
    buf = StringIO()
    old_stdout = sys.stdout
    sys.stdout = buf
    df.queryExecution.debug.codegen()
    sys.stdout = old_stdout
    f.write(buf.getvalue())
```

출력을 `raw/codegen/YYYY-MM-DD-<slug>.java`에 저장해 case가 참조할
수 있도록 합니다.

주시할 지점:
- `processNext()` 메서드 — 입력 row당 실행되는 hot loop
- `wholestagecodegen_isNull_N` / `wholestagecodegen_value_N` 변수 —
  fused된 연산자들의 컬럼별 상태를 보여줌
- `if (!(<condition>)) continue;` 패턴 — codegen에 push된 Filter
- `processNext` 내부에 object 할당이 없다는 점 — Tungsten UnsafeRow는
  raw 메모리 위에서 동작하므로 object-free
```

---

## 템플릿: Case 페이지 (/trace로 생성됨)

경로: `wiki/case/2026-04-19-customer-order-agg-trace.md`

```markdown
---
stage: case
title: 고객-주문 집계 — 전체 파이프라인 trace
spark_versions: ["3.5"]
sources:
  - raw/explain/2026-04-19-customer-order-agg.txt
  - raw/codegen/2026-04-19-customer-order-agg.java
last_verified: 2026-04-19
confidence: verified
related:
  - 03-logical-optimization/predicate-pushdown.md
  - 04-physical-planning/join-strategies.md
  - 04-physical-planning/aggregate-strategies.md
  - 05-cost-and-aqe/aqe-overview.md
  - 06-codegen/wholestage-codegen.md
  - 07-rdd-and-stages/stage-boundaries.md
tags: [trace, join, aggregation, codegen]
---

## 쿼리

```sql
SELECT c.name, SUM(o.amount) AS total
FROM customers c JOIN orders o ON c.id = o.customer_id
WHERE o.date > '2026-01-01'
GROUP BY c.name
```

## 파이프라인 요약

| 단계 | 이 쿼리에 일어나는 일 | 페이지 |
|---|---|---|
| 01 parsing | 표준 SELECT가 UnresolvedRelation 노드로 파싱됨 | [01-parsing/parser-antlr.md] |
| 02 analysis | Catalog로 `c.id`, `o.customer_id`, `o.amount`, `o.date`, `c.name`이 해결됨 | [02-analysis/catalog-resolution.md] |
| 03 logical opt | PushDownPredicate가 `o.date > '2026-01-01'`을 Join 아래로 이동시켜 orders scan에 PushedFilter로 부착. ColumnPruning이 읽히는 컬럼을 감소시킴. | [03-logical-optimization/predicate-pushdown.md] |
| 04 physical planning | JoinSelection이 SortMergeJoin을 선택 (양쪽 다 broadcast 기본값 초과). SUM에 HashAggregate 선택. SMJ 앞에 Exchange 삽입. | [04-physical-planning/join-strategies.md] [04-physical-planning/aggregate-strategies.md] |
| 05 AQE | Exchange 이후 런타임 통계: 필터 적용 후 orders 쪽이 broadcast threshold 미만이면 BroadcastHashJoin으로 전환. | [05-cost-and-aqe/aqe-dynamic-join-switch.md] |
| 06 codegen | HashAggregate + Join + Filter + Scan이 2개의 WholeStageCodegen subtree로 fuse됨 (exchange 양쪽에 각각 하나). 생성된 Java: raw/codegen/ 참고. | [06-codegen/wholestage-codegen.md] |
| 07 RDD & stages | 3개 스테이지: scan-customers, scan+filter-orders, join+aggregate. Exchange 경계가 stage 경계를 정의. | [07-rdd-and-stages/stage-boundaries.md] |
| 08 scheduling | broadcast 변형에는 PROCESS_LOCAL 선호; SMJ 변형에는 NODE_LOCAL. | [08-scheduling/locality-levels.md] |
| 09 execution | 전반적으로 Tungsten UnsafeRow 사용; Parquet vectorized reader가 생성된 코드에 배치 공급. | [09-execution/tungsten-memory.md] [09-execution/vectorized-reader.md] |

## 이 trace가 가르치는 것

1. AQE의 join 전환이 가장 큰 런타임 요인 — 이것이 없으면 SMJ
   경로는 컴파일 타임에 확정됨. `05-cost-and-aqe/aqe-dynamic-join-switch.md`
   L4에 관찰 사례 불릿을 추가함.
2. Codegen 경계는 Join이 아닌 Exchange에서 분할됨. 두 개의 별도
   생성된 클래스가 되며 하나의 클래스가 아님.
   `raw/codegen/2026-04-19-customer-order-agg.java`로 검증.
   `06-codegen/wholestage-codegen.md` L3에 명확화 노트로 추가함.
3. `executedPlan`에서 orders에는 PushedFilters가 보이지만 customers에는
   보이지 않음 (customers 쪽에 필터가 없으므로). `predicate-pushdown.md`
   L5 예시가 문서화된 대로 동작함을 확인.

## 드러난 갭

- `05-cost-and-aqe/aqe-dynamic-join-switch.md`에 L5가 없음. 전환을
  실제로 관찰하기 위한 소스 추적 TODO 추가.
- `04-physical-planning/aggregate-strategies.md`가 ObjectHashAggregate와
  HashAggregate 선택을 아직 다루지 않음. Ingest 플래그.
```
