---
stage: 05-cost-and-aqe
title: AQE — Skew Join Optimization
spark_versions: ["3.5", "4.1"]
sources:
  - raw/spark-docs/2026-04-spark-3.5.6-sql-performance-tuning.md
  - raw/spark-docs/2026-04-spark-4.1.1-sql-performance-tuning.md
last_verified: 2026-04-19
confidence: tentative
related:
  - 00-overview/execution-pipeline.md
  - 05-cost-and-aqe/aqe-overview.md
  - 04-physical-planning/join-strategy-hints.md
configs:
  - spark.sql.adaptive.skewJoin.enabled
  - spark.sql.adaptive.skewJoin.skewedPartitionFactor
  - spark.sql.adaptive.skewJoin.skewedPartitionThresholdInBytes
  - spark.sql.adaptive.forceOptimizeSkewedJoin
apis: []
tags: [aqe, skew, join, data-skew, straggler]
---

## L1. What & Why

Join key의 data skew는 프로덕션 Spark 워크로드에서 latency가 끝없이
늘어지는 가장 흔한 원인이다: 하나의 skew key(null customer-id, 봇
사용자, 레거시 dump 값 등)가 자기의 모든 row를 하나의 shuffle
partition에 몰아넣을 수 있다. 그 partition의 task는 동료들보다 100배
오래 실행되고, stage는 그 task를 기다리며 — 나머지 모든 executor는
놀고 있는다.

AQE의 skew-join 규칙은 partition 크기를 median과 비교해 skew partition을
탐지하고, 이를 더 작은 chunk로 분할한 뒤 반대쪽의 매칭 partition을
복제해 join 의미를 보존한다. 이 기능이 없다면 사용자 수준의 대응책은
key를 직접 salt 하거나 반대쪽을 broadcast 하는 것뿐인데, 둘 다 코드
변경과 데이터 수준의 지식이 필요하다. Skew 탐지는 런타임 기능이며,
AQE가 켜진 상태에서 기본 활성화된다.

## L2. Inputs & Outputs

입력: `SortMergeJoinExec`으로 흘러 들어가는 materialize 된 shuffle
출력. partition별 크기 통계가 `AdaptiveSparkPlanExec`에 제공된 상태.

출력: skew 쪽의 shuffle 입력이 N개 chunk(각각
`advisoryPartitionSizeInBytes`를 목표로 함)로 분할되고, 반대쪽의
매칭 partition이 N번 복제되어 모든 skew chunk가 join 대상 데이터를
갖도록 수정된 physical plan. SortMergeJoin operator 자체는 변하지
않음 — shuffle-read 단계만 재구성됨.

## L3. Key Mechanisms

> 📝 docs 요약 수준. 소스 코드 ingest 대기 중
> (`sql/core/src/main/scala/org/apache/spark/sql/execution/adaptive/OptimizeSkewedJoin.scala`,
> `ShufflePartitionsUtil.scala`).

docs 기준의 skew 판정 조건:

partition이 "skewed"로 판정되려면 **둘 다** 만족:

1. 크기 > `spark.sql.adaptive.skewJoin.skewedPartitionFactor` × median
   partition 크기, **AND**
2. 크기 > `spark.sql.adaptive.skewJoin.skewedPartitionThresholdInBytes`.

두 조건 모두 중요하다: factor만 쓰면 작은 데이터셋의 정상적인 분산을
skew로 오인하고, threshold만 쓰면 작지만 심하게 skew된 join을 놓친다.
절대 threshold(기본 256 MB)는 floor 역할 — shuffle이 매우 작으면
비율적 불균형과 상관없이 skew 처리가 발동하지 않는다.

탐지되면 skew partition은 `spark.sql.adaptive.advisoryPartitionSizeInBytes`
(coalesce 규칙과 공유; [aqe-overview.md] 참고)를 목표로 동일 크기의
chunk로 분할된다. 반대쪽의 해당 partition은 chunk 수만큼 복제된다.

이 규칙은 **SortMergeJoin에만 적용된다**. plan이 ShuffledHashJoin을
쓰는 경우(예: `spark.sql.adaptive.maxShuffledHashJoinLocalMapThreshold`
설정으로 인한) 이 규칙은 동작하지 않음 — skew partition은 하나의
hash-build task가 통째로 처리한다.

`spark.sql.adaptive.forceOptimizeSkewedJoin` (3.3+, 기본 false):
이 플래그 없이는 분할이 추가 shuffle을 유발하는 경우 규칙이 생략된다.
켜면 그 비용을 감수하고라도 최적화를 강제.

## L4. Performance Levers

1. **기본 threshold(5.0× median, 256 MB)는 보수적이다.** 작은
   데이터셋에 대한 key 단위 강한 skew 상황에서는 256 MB floor가 규칙
   발동을 막을 수 있다. `skewedPartitionThresholdInBytes`를 낮추면 더
   작은 크기에서도 분할이 활성화된다.

2. **`forceOptimizeSkewedJoin` (3.3+)은 추가 shuffle을 감수하고 skew를
   해소한다.** straggler가 end-to-end latency를 지배하는데 기본 규칙이
   "fix가 또 다른 shuffle을 만드니 생략"으로 판단하는 경우 활성화 가치
   있음. 활성화 후 전체 shuffle 볼륨을 모니터링할 것.

3. **규칙은 SortMergeJoin 전용.** plan이 ShuffledHashJoin이면 이
   규칙은 dormant — [aqe-overview.md]의 `maxShuffledHashJoinLocalMapThreshold`
   참고. SHJ로 의도적으로 실행되는 skew key join이라면 직접 salting을
   고려.

4. **`advisoryPartitionSizeInBytes`가 분할 granularity를 정한다.**
   post-shuffle coalescing과 공유된다. 낮추면 skew partition이 더 많은
   chunk로 쪼개지고(더 높은 병렬성, 더 많은 총 task), 올리면 chunk가
   더 적고 크게 유지된다.

5. **"skew가 안 고쳐짐"을 디버깅할 때 `skewJoin.enabled`를 먼저 확인.**
   기본 true지만 `spark.sql.adaptive.enabled`가 false이거나 누군가
   명시적으로 껐을 수 있음.

## L5. How to Observe

> TODO: `queryExecution.executedPlan` 또는 SQL UI 관찰 사례 필요.

```python
# 쿼리 실행 후, skew 처리 결과가 반영된 plan 조회
print(df.queryExecution.executedPlan)
# shuffle-read 쪽에서 partition split 징후가 보이는 SortMergeJoin을
# 찾는다 — 정확한 시각화는 AQE 버전별로 다르므로, 실관찰 데이터가
# 수집될 때까지는 placeholder 수준.
```

SQL UI 신호: Jobs / Stages 뷰에서 skew 처리가 발동한 stage는
post-shuffle partition 개수보다 task 개수가 눈에 띄게 많고(분할이
추가 task를 만듦), task 실행 시간이 bimodal이 아니라 대체로 균일해
진다.

규칙이 켜져 있는지 확인:

```python
spark.conf.get("spark.sql.adaptive.enabled")
spark.conf.get("spark.sql.adaptive.skewJoin.enabled")
spark.conf.get("spark.sql.adaptive.skewJoin.skewedPartitionFactor")
spark.conf.get("spark.sql.adaptive.skewJoin.skewedPartitionThresholdInBytes")
```

## Version notes

- `spark.sql.adaptive.skewJoin.enabled`,
  `spark.sql.adaptive.skewJoin.skewedPartitionFactor`,
  `spark.sql.adaptive.skewJoin.skewedPartitionThresholdInBytes`: 모두
  Spark 3.0.0부터.
- `spark.sql.adaptive.forceOptimizeSkewedJoin`: since 3.3.0.
- 기본값과 docs 문구는 3.5.6과 4.1.1 동일.
- 3.5.6 서술은 skew join을 AQE의 "세 가지 주요 기능" 중 하나로 소개;
  4.1.1은 그 framing을 뺐지만 동일한 설정 표와 설명은 유지.
