---
stage: 05-cost-and-aqe
title: Adaptive Query Execution — Overview
spark_versions: ["3.5", "4.1"]
sources:
  - raw/spark-docs/2026-04-spark-3.5.6-sql-performance-tuning.md
  - raw/spark-docs/2026-04-spark-4.1.1-sql-performance-tuning.md
last_verified: 2026-04-19
confidence: tentative
related:
  - 00-overview/execution-pipeline.md
  - 04-physical-planning/join-strategy-hints.md
  - 04-physical-planning/coalesce-repartition-hints.md
  - 05-cost-and-aqe/aqe-skew-join.md
  - 05-cost-and-aqe/leveraging-statistics.md
configs:
  - spark.sql.adaptive.enabled
  - spark.sql.adaptive.coalescePartitions.enabled
  - spark.sql.adaptive.coalescePartitions.parallelismFirst
  - spark.sql.adaptive.coalescePartitions.minPartitionSize
  - spark.sql.adaptive.coalescePartitions.initialPartitionNum
  - spark.sql.adaptive.advisoryPartitionSizeInBytes
  - spark.sql.adaptive.optimizeSkewsInRebalancePartitions.enabled
  - spark.sql.adaptive.rebalancePartitionsSmallPartitionFactor
  - spark.sql.adaptive.autoBroadcastJoinThreshold
  - spark.sql.adaptive.localShuffleReader.enabled
  - spark.sql.adaptive.maxShuffledHashJoinLocalMapThreshold
  - spark.sql.adaptive.optimizer.excludedRules
  - spark.sql.adaptive.customCostEvaluatorClass
apis: []
tags: [aqe, adaptive, runtime-optimization, coalesce, broadcast-switch, rebalance]
---

## L1. What & Why

Adaptive Query Execution (AQE)은 완료된 shuffle stage에서 수집한 통계를
이용해 런타임에 physical plan을 재최적화하는 기능이다. Spark 3.2.0부터
기본 활성화. 전통적 쿼리 플래닝의 구조적 약점을 해결한다: plan 결정
(join 전략, partition 개수, skew 처리)이 어떤 데이터도 흐르기 전에,
자주 틀리거나 누락된 통계에 기반해 고정된다는 문제. AQE는 각 shuffle
경계에서 개입한다 — map stage가 끝나는 시점에 Spark는 각 출력 partition의
정확한 크기를 알고 downstream을 재계획할 수 있다.

## L2. Inputs & Outputs

입력: `Exchange` operator(shuffle 경계)가 표시된 physical plan. 각
shuffle은 materialization 시점에 map-output 크기 통계가
`AdaptiveSparkPlanExec` wrapper에 제공되도록 감싸진다.

출력: 쿼리 실행 중에 변화하는 physical plan. 각 shuffle이 완료될
때마다 wrapper는 다음을 수행할 수 있다:

- post-shuffle partition을 더 적고 큰 partition으로 coalesce.
- skew partition을 분할 ([05-cost-and-aqe/aqe-skew-join.md] 참고).
- 측정된 크기가 허용하면 `SortMergeJoinExec`을 `BroadcastHashJoinExec`
  또는 `ShuffledHashJoinExec`으로 교체.
- 가능할 때 `Exchange`를 local shuffle reader로 대체.

## L3. Key Mechanisms

> 📝 docs 요약 수준. 소스 코드 ingest 대기 중
> (`sql/core/src/main/scala/org/apache/spark/sql/execution/adaptive/AdaptiveSparkPlanExec.scala`,
> `InsertAdaptiveSparkPlan`, `CoalesceShufflePartitions`,
> `DynamicJoinSelection`, `OptimizeSkewedJoin`, `adaptive` 패키지 전반).

AQE는 `spark.sql.adaptive.enabled`(3.2부터 기본 true)가 지배하는
umbrella 기능이다. 활성화되면 다음 하위 메커니즘들이 동작한다:

### Coalescing post-shuffle partitions

shuffle이 materialize 된 뒤 연속된 작은 출력 partition을 downstream
stage를 위해 더 적고 큰 partition으로 coalesce 한다. 목표 크기는
`spark.sql.adaptive.advisoryPartitionSizeInBytes` (기본 64 MB).

| Config | Default | 역할 |
|---|---|---|
| `spark.sql.adaptive.coalescePartitions.enabled` | true | 이 기능 활성화 |
| `spark.sql.adaptive.coalescePartitions.parallelismFirst` | true | true면 advisory 크기를 무시하고 `minPartitionSize`만 지켜 병렬성을 극대화 |
| `spark.sql.adaptive.coalescePartitions.minPartitionSize` | 1 MB | coalesce 후 최소 크기 |
| `spark.sql.adaptive.coalescePartitions.initialPartitionNum` | (none) | coalesce 이전 초기 partition 개수; 미설정 시 `spark.sql.shuffle.partitions` |
| `spark.sql.adaptive.advisoryPartitionSizeInBytes` | 64 MB | 목표 partition 크기 |

docs는 `advisoryPartitionSizeInBytes`를 실제로 존중하려면
`parallelismFirst = false`로 설정할 것을 권장한다. 버전 간 문구 차이는
"Version notes" 참고.

### Converting sort-merge join to broadcast join

shuffle이 materialize 되고 한쪽의 총 크기가
`spark.sql.adaptive.autoBroadcastJoinThreshold`
(기본값은 `spark.sql.autoBroadcastJoinThreshold`와 동일)보다 작게
나오면, AQE는 join을 broadcast hash join으로 재계획한다. 처음부터
broadcast로 계획한 것만큼 효율적이지는 않지만, 양쪽의 sort를 피하고
shuffle 파일을 local에서 읽을 수 있게 해준다
(`spark.sql.adaptive.localShuffleReader.enabled`).

| Config | Default | 역할 |
|---|---|---|
| `spark.sql.adaptive.autoBroadcastJoinThreshold` | (`spark.sql.autoBroadcastJoinThreshold`와 동일) | 런타임 broadcast 전환의 크기 threshold |
| `spark.sql.adaptive.localShuffleReader.enabled` | true | SMJ → BHJ 전환 후 shuffle 파일을 local에서 읽도록 허용 |

### Converting sort-merge join to shuffled hash join

모든 post-shuffle partition이
`spark.sql.adaptive.maxShuffledHashJoinLocalMapThreshold`보다 작게
나오면, join 선택은 `spark.sql.join.preferSortMergeJoin`을 무시하고
shuffled hash join을 선호한다.

| Config | Default | 역할 |
|---|---|---|
| `spark.sql.adaptive.maxShuffledHashJoinLocalMapThreshold` | 0 | local hash map build를 허용하는 partition당 최대 크기 |

기본값 0은 이 기능이 사실상 opt-in임을 의미 — 활성화하려면 0이 아닌
값(그리고 `advisoryPartitionSizeInBytes` 이상)으로 설정해야 한다.

### Splitting skewed shuffle partitions (via Rebalance)

SMJ skew 최적화([aqe-skew-join.md] 참고)와는 별개로, 이 규칙은
`RebalancePartitions`(=`REBALANCE` hint와 AQE 기반 repartitioning의
뒷단 logical 노드)가 만든 skew partition을 처리한다.

| Config | Default | 역할 |
|---|---|---|
| `spark.sql.adaptive.optimizeSkewsInRebalancePartitions.enabled` | true | rebalance 출력에 대한 skew 분할 활성화 |
| `spark.sql.adaptive.rebalancePartitionsSmallPartitionFactor` | 0.2 | 분할 후 `factor × advisoryPartitionSizeInBytes`보다 작은 partition은 다시 합침 |

### Optimizing skew join

전용 페이지에서 상세히 다룸. [aqe-skew-join.md] 참고. 위의
rebalance-skew 규칙과는 구별됨 — sort-merge join 내부의 skew만
대상으로 한다.

### Advanced customization

| Config | Default | 역할 |
|---|---|---|
| `spark.sql.adaptive.optimizer.excludedRules` | (none) | 비활성화할 AQE optimizer rule 이름을 쉼표로 나열 |
| `spark.sql.adaptive.customCostEvaluatorClass` | (none) | 사용자 정의 cost evaluator의 FQCN; 기본은 `SimpleCostEvaluator` |

## L4. Performance Levers

1. **`advisoryPartitionSizeInBytes`를 실제로 쓰려면 `parallelismFirst = false`.**
   기본 `true`는 병렬성 우선(=`minPartitionSize = 1 MB`만 지킴)으로
   동작해 의도보다 많은, 작은 task를 만든다. 워크로드가 task-overhead-bound
   이거나 downstream 쓰기가 64 MB chunk를 선호하면 `false`로 뒤집어라.
   4.1.1 docs는 이 권고를 "busy cluster에서"로 완화했다(3.5.6은 더
   강하게 권장) — "Version notes" 참고.

2. **`maxShuffledHashJoinLocalMapThreshold`는 기본 off(0).** 양수(그리고
   `advisoryPartitionSizeInBytes` 이상)로 설정하면 SMJ → SHJ 런타임
   전환이 활성화된다. 양쪽이 post-shuffle에서 작지만 broadcast 할
   만큼 작지는 않을 때 SMJ보다 저렴하다.

3. **`adaptive.autoBroadcastJoinThreshold`는 별개 knob.**
   `spark.sql.autoBroadcastJoinThreshold`만 올린다고 AQE 쪽까지 덮인다고
   가정하지 말 것 — adaptive 쪽 threshold는 정적 쪽 값을 기본값으로
   쓰지만, 런타임 전환용으로 독립 튜닝 가능.

4. **`excludedRules`는 외과적 디버깅 수단.** AQE가 놀라운 행동을 하면
   이 설정으로 rule을 하나씩 비활성화해 동작을 분리해 본다. 제외된
   rule은 log에 찍힌다.

5. **`localShuffleReader.enabled`는 true 유지가 권장.** 비활성화하면
   전환 후 BHJ read가 shuffle service를 경유한다.

## L5. How to Observe

> TODO: `queryExecution.executedPlan` 또는 SQL UI 관찰 사례 필요.

```python
# AQE가 켜지면 physical plan에 AdaptiveSparkPlanExec wrapper가 보인다
print(df.queryExecution.executedPlan)
# AdaptiveSparkPlanExec 안의 plan은 shuffle이 materialize 될 때마다
# 바뀌므로, 전환을 보려면 실행 중/후 여러 번 조회한다.
```

SQL UI → SQL/DataFrame 탭 → 해당 쿼리의 graph. AQE 재계획은 실행
중/완료된 쿼리의 plan 이력에 "adaptive" 전환으로 나타난다.

```python
# AQE가 제외한 rule 확인
spark.conf.get("spark.sql.adaptive.optimizer.excludedRules")
```

## Version notes

- `spark.sql.adaptive.enabled`: 설정 표(4.1.1)에는 "since 1.6.0"으로
  적혀 있음 — 인프라는 더 일찍 존재했고, 섹션 도입부에 따르면 3.2.0에서
  기본값이 true로 전환됨.
- 3.5.6 docs는 AQE를 "세 가지 주요 기능 ... post-shuffle partition
  coalescing, sort-merge join → broadcast join 전환, skew join 최적화"로
  소개. 4.1.1은 이 문장을 삭제하고 섹션에 5개 하위 기능 + "Advanced
  Customization"을 나열.
- 4.1.1은 `spark.sql.adaptive.enabled`를 표의 row로 명시; 3.5.6은
  prose로만 언급.
- `parallelismFirst` 권고 문구 (docs 의역, 원문은 `sources:`의 raw/ 참조):
  - 3.5.6:
    > false로 설정해 `advisoryPartitionSizeInBytes`가 지정한 target size를 존중하라.
  - 4.1.1:
    > busy cluster에서는 false로 설정해 리소스 사용 효율을 높이라 (작은 task 과다 방지).
- `spark.sql.adaptive.rebalancePartitionsSmallPartitionFactor`: since 3.3.0.
- `spark.sql.adaptive.forceOptimizeSkewedJoin`([aqe-skew-join.md]에서
  다룸): since 3.3.0.
- 나머지 AQE 설정들은 3.5.6과 4.1.1 모두 동일한 기본값으로 존재.
