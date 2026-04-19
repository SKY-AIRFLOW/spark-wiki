---
stage: 04-physical-planning
title: WindowExec — Window function 물리 실행
spark_versions: ["3.5"]
sources:
  - raw/explain/2026-04-19-query-B-row-number-top5.txt
  - raw/spark-source/v3.5.6/sql/core/src/main/scala/org/apache/spark/sql/execution/SparkStrategies.scala
last_verified: 2026-04-19
confidence: tentative
related:
  - 00-overview/execution-pipeline.md
  - 04-physical-planning/window-group-limit.md
  - 04-physical-planning/hash-aggregate-partial-final.md
  - case/2026-04-19-query-B-row-number-top5.md
configs: []
apis:
  - DataFrame.withColumn (with Window spec)
  - SQL ROW_NUMBER / RANK / DENSE_RANK / LAG / LEAD / SUM OVER / ...
tags: [window, physical-planning, partition-by, order-by, ranking]
---

## L1. What & Why

`WindowExec`는 SQL의 window function (`ROW_NUMBER()`, `RANK()`,
`SUM(...) OVER (...)`, `LAG(...)` 등)을 plan에서 구체 연산으로
실현하는 물리 노드다. 파이프라인 단계 04에서 `Window` strategy가
논리 `Window` 노드를 `WindowExec`로 변환한다.

Window function의 의미: 같은 PARTITION BY 키를 가진 row들을 한 그룹으로
묶고, 그룹 내부를 ORDER BY에 따라 정렬한 뒤, 각 row에 대해 window frame
범위의 값을 읽어 함수를 계산해 한 컬럼을 덧붙인다. Aggregate와 달리
**row 수가 줄지 않는다** — 입력 row 수 = 출력 row 수. 단지 "해당 row의
파티션 내 순위" 같은 컬럼이 하나 더 붙는다.

이 연산이 분산에서 성립하려면 같은 PARTITION BY 키의 모든 row가 같은
partition에 있어야 한다 — 따라서 WindowExec 상위에 보통
`Exchange hashpartitioning(<partition keys>)`가 삽입되고, 그 위에
`Sort [<partition keys> ASC, <order keys> ...]`가 붙어 frame 계산 전제를
만든다.

## L2. Inputs & Outputs

**입력** (Optimized LogicalPlan):

```
Window [<windowExpression>, ...],
       [<partitionSpec>],
       [<orderSpec>]
```

- `windowExpression`: `<aggregateOrRankFunction>()
  windowspecdefinition(<partitionSpec>, <orderSpec>, <frameSpec>) AS alias#id`
- `partitionSpec`: PARTITION BY 표현식 목록 (비어 있으면 단일 파티션)
- `orderSpec`: ORDER BY 표현식 목록
- `frameSpec`: `specifiedwindowframe(RowFrame|RangeFrame, <start>, <end>)`
  또는 `unspecifiedframe()`

**출력** (Physical Plan):

```
Window [<windowExpression list>],
       [<partitionSpec>],
       [<orderSpec>]
```

위에 요구되는 distribution은 `ClusteredDistribution(partitionSpec)`, ordering은
`partitionSpec ASC :: orderSpec`. 따라서 플래너가 자동으로 상위에
`Exchange hashpartitioning(partitionSpec)` 및 `Sort [partitionSpec ASC ..., orderSpec]`
를 배치한다.

**쿼리 B 관찰** (executed plan, nodes 7-9):

```
Window [row_number() windowspecdefinition(agent_gu#308,
                                          deal_amount#304 DESC NULLS LAST,
                                          specifiedwindowframe(RowFrame,
                                            unboundedpreceding$(),
                                            currentrow$()))
        AS price_rank#296],
       [agent_gu#308],          # partitionSpec
       [deal_amount#304 DESC NULLS LAST]  # orderSpec
+- Sort [agent_gu#308 ASC NULLS FIRST,
         deal_amount#304 DESC NULLS LAST], false, 0
   +- <upstream>
```

- partitionSpec은 `agent_gu#308` — 자치구별 그룹핑.
- orderSpec은 `deal_amount#304 DESC NULLS LAST` — 거래금액 내림차순.
- ROW_NUMBER()의 기본 frame은 `RowFrame UNBOUNDED PRECEDING ~ CURRENT ROW` —
  ROW_NUMBER에게 frame은 사실상 무의미하지만 Analyzer가 항상 채움.

## L3. Key Mechanisms

### Strategy 변환 (verified)

`sql/core/.../execution/SparkStrategies.scala:641-655`에 정의:

```scala
object Window extends Strategy {
  def apply(plan: LogicalPlan): Seq[SparkPlan] = plan match {
    case PhysicalWindow(
      WindowFunctionType.SQL, windowExprs, partitionSpec, orderSpec, child) =>
      execution.window.WindowExec(
        windowExprs, partitionSpec, orderSpec, planLater(child)) :: Nil

    case PhysicalWindow(
      WindowFunctionType.Python, windowExprs, partitionSpec, orderSpec, child) =>
      execution.python.WindowInPandasExec(
        windowExprs, partitionSpec, orderSpec, planLater(child)) :: Nil

    case _ => Nil
  }
}
```

두 분기:
- **SQL/Scala UDF path**: `execution.window.WindowExec`
- **Python UDF path**: `execution.python.WindowInPandasExec` (vectorized Arrow 기반)

`PhysicalWindow` extractor는 logical `Window` 노드를 분해해 function
type을 판별.

### WindowExec 내부 동작

> 📝 docs 요약 수준. 소스 코드 ingest 대기 중
> (`sql/core/src/main/scala/org/apache/spark/sql/execution/window/WindowExec.scala`,
> `WindowFunctionFrame.scala`, `WindowFunctionType.scala`).

Plan 관찰로 확정된 사실만:
- 입력은 partitionSpec + orderSpec 기준 이미 정렬된 row stream (plan이
  상위에 Sort를 반드시 둠).
- 한 partition의 row들을 순차 순회하며 각 row마다 window frame 범위의
  데이터에 function 적용.
- ROW_NUMBER / RANK / DENSE_RANK 같은 rank-like function은 frame과 무관하게
  group-local counter만 증가.
- SUM/AVG 등 aggregate-over-window는 `WindowFunctionFrame`이 sliding 또는
  growing window buffer를 유지.

정확한 buffer 관리, frame 타입별 분기, spill 동작은 소스 ingest 후 기록.

## L4. Performance Levers

> 📝 docs 요약 수준. 소스 코드 ingest 대기 중.

- `spark.sql.shuffle.partitions`: partitionSpec 기반 Exchange의 partition 수.
- Skew: 특정 partition 키에 row가 몰리면 해당 Exchange partition의 WindowExec
  task가 straggler가 된다. AQE skew 처리는 주로 join용이라 window에는 제한적.
- 좁은 partition key (e.g. `agent_gu` 25개) → shuffle partition 수(200) 대비
  partition 수 부족 → 다수 shuffle partition이 빈 채 유휴.
- Window function이 **여러 개**일 때 같은 PARTITION BY 키를 공유하면 단일
  WindowExec에 합쳐짐 (single shuffle + single sort). 다른 키를 쓰면 별도
  WindowExec + 추가 shuffle.
- Rank-like function에 `rank <= N` filter가 붙으면 `WindowGroupLimit`이 앞에
  삽입되어 shuffle 전후 volume을 줄인다 — 별도 페이지 참조:
  [wiki/04-physical-planning/window-group-limit.md](window-group-limit.md).

## L5. How to Observe

- `df.explain(mode="formatted")`:
  - Physical plan에 `Window [...]` 노드 확인.
  - 바로 아래에 `Sort [<partitionSpec> ASC ..., <orderSpec>]`가 있는지.
  - Sort 아래에 `Exchange hashpartitioning(<partitionSpec>)`가 있는지.
- `WindowFunctionType.Python` 분기가 탔는지: plan 노드가 `WindowInPandasExec`
  로 뜨면 그쪽. UDF가 Pandas UDF면 Arrow 경로.
- Spark UI → SQL tab의 DAG에서 `Window` 박스 + 그 아래 Sort + Exchange 삼단
  구조 확인.

> 📝 probe 페이지 미작성. 후속 `/probe 04-physical-planning/window` 대상.
