---
stage: 04-physical-planning
title: Join Strategy Hints
spark_versions: ["3.5"]
sources:
  - raw/spark-docs/2026-04-spark-3.5.6-sql-performance-tuning.md
  - raw/spark-docs/2026-04-spark-4.1.1-sql-performance-tuning.md
  - raw/spark-source/v3.5.6/sql/core/src/main/scala/org/apache/spark/sql/execution/SparkStrategies.scala
  - raw/spark-source/v3.5.6/sql/catalyst/src/main/scala/org/apache/spark/sql/catalyst/optimizer/joins.scala
last_verified: 2026-04-19
confidence: verified
related:
  - 00-overview/execution-pipeline.md
  - 04-physical-planning/coalesce-repartition-hints.md
  - 04-physical-planning/storage-partition-join.md
  - 05-cost-and-aqe/aqe-overview.md
  - 05-cost-and-aqe/leveraging-statistics.md
configs:
  - spark.sql.autoBroadcastJoinThreshold
  - spark.sql.broadcastTimeout
  - spark.sql.join.preferSortMergeJoin
  - spark.sql.shuffle.hashJoinFactor
apis:
  - DataFrame.hint
  - SQL /*+ BROADCAST */
  - SQL /*+ MERGE */
  - SQL /*+ SHUFFLE_HASH */
  - SQL /*+ SHUFFLE_REPLICATE_NL */
tags: [join, hint, broadcast, physical-planning]
---

## L1. What & Why

Join strategy hint는 특정 relation에 대해 Spark의 physical planner가
어느 join 알고리즘을 선호하도록 지시하는, 사용자가 직접 제공하는
지침이다. 크기/통계 기반 기본 선택을 오버라이드한다. 단계 04의
`JoinSelection` strategy는 5개의 join operator 중 하나를 고르며,
선택은 relation 크기, equi-join key 유무, join 타입, 그리고
`spark.sql.*` 설정값에 따라 결정된다. hint는 통계가 누락되었거나 잘못된
경우의 탈출구다 — hint가 없으면 `ANALYZE`가 한 번도 실행되지 않은
테이블에 대한 join은 자동 broadcast를 trigger할 수 없고, 조용히
sort-merge로 떨어진다. 역으로 hint가 있어도 join 타입이 해당 strategy의
build side를 허용하지 않으면 **조용히 fallback**된다 — 정확한 조건은
L3·L4 참고.

## L2. Inputs & Outputs

**입력**: `ExtractEquiJoinKeys` 또는 비-equi `logical.Join`으로 매칭되는
Optimized LogicalPlan 노드. 선택적으로 `JoinHint(leftHint, rightHint)`
메타데이터가 부착됨 (SQL `/*+ ... */` → `ResolveHints`가 analysis 단계에서
주입, 또는 DataFrame `.hint("...")` 호출).

**출력**: 5개 physical operator 중 하나:

- `joins.BroadcastHashJoinExec` (SparkStrategies.scala:245-252)
- `joins.ShuffledHashJoinExec` (SparkStrategies.scala:262-269)
- `joins.SortMergeJoinExec` (SparkStrategies.scala:275-276)
- `joins.CartesianProductExec` (SparkStrategies.scala:286)
- `joins.BroadcastNestedLoopJoinExec` (SparkStrategies.scala:303-304, 374-376 — 비-equi 경로의 최종 fallback 포함)

`JoinSelection.apply`의 top-level pattern match는 3개 case로 구성
(SparkStrategies.scala:216-410):

1. `case j @ ExtractEquiJoinKeys(...)` — equi-join 경로 (237-316)
2. `case j @ ExtractSingleColumnNullAwareAntiJoin(...)` — null-aware anti
   join 특례. 무조건 `BroadcastHashJoinExec(isNullAwareAntiJoin=true)`
   (318-320)
3. `case logical.Join(...)` — 비-equi join 경로 (338-407)

## L3. Key Mechanisms

### 3-1. 4개 hint 이름과 내부 표현

사용자 API의 hint 이름은 `JoinHint.{leftHint, rightHint}` 내부에
`strategy` 필드로 저장되며, 실제 식별자는 다음과 같다
(joins.scala:425-481):

| 사용자 API | 내부 식별자 | 확인 메서드 |
|---|---|---|
| `BROADCAST` / `BROADCASTJOIN` / `MAPJOIN` | `BROADCAST` | `hintToBroadcastLeft/Right` |
| `MERGE` | `SHUFFLE_MERGE` | `hintToSortMergeJoin` |
| `SHUFFLE_HASH` | `SHUFFLE_HASH` | `hintToShuffleHashJoinLeft/Right` |
| `SHUFFLE_REPLICATE_NL` | `SHUFFLE_REPLICATE_NL` | `hintToShuffleReplicateNL` |

이 외에 내부적으로 동작하는 추가 strategy도 존재:
- `PREFER_SHUFFLE_HASH`: `hintToPreferShuffleHashJoin*`가 체크
  (joins.scala:457-467)
- `NO_BROADCAST_HASH`, `NO_BROADCAST_AND_REPLICATION`: broadcast 거부
  (joins.scala:433-447, 487-493)

### 3-2. Equi-join 경로 (SparkStrategies.scala:237-316)

`JoinSelection.apply`는 `ExtractEquiJoinKeys` 매칭 후 5개 보조 메서드를
정의한다:

- `createBroadcastHashJoin(onlyLookingAtHint: Boolean)` (239-254):
  `getBroadcastBuildSide(...)`로 build side를 받고 `Some(Seq(BHJ))`
  또는 `None` 반환.
- `createShuffleHashJoin(onlyLookingAtHint: Boolean)` (256-271).
- `createSortMergeJoin()` (273-280): `RowOrdering.isOrderable(leftKeys)`
  가 true일 때만 `Some(Seq(SMJ))`.
- `createCartesianProduct()` (282-290): `InnerLike` + `!hintToNotBroadcastAndReplicate`
  일 때만.
- `createJoinWithoutHint()` (292-306): 우선순위 순으로 시도 후 최종
  fallback인 `BroadcastNestedLoopJoinExec`.

**핵심 결정 블록** (SparkStrategies.scala:308-316):

```scala
// SparkStrategies.scala:308-316 (GitHub 원본 라인; local 파일은 +8)
if (hint.isEmpty) {
  createJoinWithoutHint()
} else {
  createBroadcastHashJoin(true)
    .orElse { if (hintToSortMergeJoin(hint)) createSortMergeJoin() else None }
    .orElse(createShuffleHashJoin(true))
    .orElse { if (hintToShuffleReplicateNL(hint)) createCartesianProduct() else None }
    .getOrElse(createJoinWithoutHint())   // <-- "hint 무시되는 fallback"의 본체
}
```

이 `.getOrElse(createJoinWithoutHint())`가 docs에서 "hint는
best-effort"라고 표현한 동작의 **정확한 소스 증거**다. hint가 붙어
있어도 네 번의 `.orElse`가 모두 `None`을 반환하면 hint가 전혀 없는
것처럼 `createJoinWithoutHint()`로 떨어진다. 경고·에러 없이 조용히.

#### hint 경로의 우선순위

순서는 `.orElse` 체인에 고정됨: **BHJ → SMJ(hint 있을 때만) → SHJ →
Cartesian(SHUFFLE_REPLICATE_NL hint 있을 때만)**. docs가 말하는
`BROADCAST` > `MERGE` > `SHUFFLE_HASH` > `SHUFFLE_REPLICATE_NL`
우선순위는 이 체인으로 구현된다.

#### no-hint 경로의 우선순위 (createJoinWithoutHint, 292-306)

```scala
// SparkStrategies.scala:292-306
createBroadcastHashJoin(false)
  .orElse(createShuffleHashJoin(false))
  .orElse(createSortMergeJoin())
  .orElse(createCartesianProduct())
  .getOrElse {
    // 최종 fallback: BroadcastNestedLoopJoin (OOM 위험)
    val requiredBuildSide = getBroadcastNestedLoopJoinBuildSide(hint)
    val buildSide = requiredBuildSide.getOrElse(getSmallerSide(left, right))
    Seq(joins.BroadcastNestedLoopJoinExec(...))
  }
```

BHJ → SHJ → SMJ → Cartesian → BroadcastNestedLoop(최후수단).

### 3-3. 비-equi join 경로 (SparkStrategies.scala:338-407)

equi-key가 없으면 BHJ·SHJ·SMJ 전부 불가 (hash/merge 구현은 키가 필요).
남은 선택지는 `BroadcastNestedLoopJoinExec` 또는 `CartesianProductExec`.

- `desiredBuildSide` 결정 (340-348): `InnerLike`·`FullOuter`는 작은 쪽,
  그 외는 `canBuildBroadcastLeft(joinType)`에 따라 `BuildLeft`/`BuildRight`.
- `checkHintNonEquiJoin(hint)` (339, 본체 207-214): SHJ·MERGE·
  PREFER_SHUFFLE_HASH hint가 있으면 `hintErrorHandler.joinHintNotSupported(...)`
  로 경고 — 이 hint들은 비-equi에서 기술적으로 적용 불가하므로 사용자에게
  피드백을 남긴다.
- hint 경로: `createBroadcastNLJoin(true)` → Cartesian(SHUFFLE_REPLICATE_NL 일 때) →
  `createJoinWithoutHint()` (401-407).

### 3-4. `JoinSelectionHelper` 핵심 메서드

`JoinSelection extends JoinSelectionHelper`이므로 아래 trait 메서드들이
직접 호출 가능 (joins.scala:294-543).

#### `canBroadcastBySize(plan, conf)` (joins.scala:369-377)

```scala
// joins.scala:369-377
def canBroadcastBySize(plan: LogicalPlan, conf: SQLConf): Boolean = {
  val autoBroadcastJoinThreshold = if (plan.stats.isRuntime) {
    conf.getConf(SQLConf.ADAPTIVE_AUTO_BROADCASTJOIN_THRESHOLD)
      .getOrElse(conf.autoBroadcastJoinThreshold)
  } else {
    conf.autoBroadcastJoinThreshold
  }
  plan.stats.sizeInBytes >= 0 && plan.stats.sizeInBytes <= autoBroadcastJoinThreshold
}
```

- 런타임 stats (AQE 단계)에서는 `spark.sql.adaptive.autoBroadcastJoinThreshold`를
  우선 사용, 미설정 시 정적 `autoBroadcastJoinThreshold`로 fallback.
- 비-runtime에서는 정적 `autoBroadcastJoinThreshold`만.
- 통계가 없을 때 Spark는 `sizeInBytes`로 `spark.sql.defaultSizeInBytes`
  (기본 Long.MaxValue)를 채워 넣는다 → 두 번째 조건
  (`sizeInBytes <= threshold`)에서 실패. 이것이 docs 각주 "ANALYZE TABLE
  필요"의 작동 경로.

#### join 타입별 build side 허용 매트릭스

소스에서 직접 도출 (joins.scala:379-406):

| join 타입 | BroadcastHash Left | BroadcastHash Right | ShuffledHash Left | ShuffledHash Right |
|---|:---:|:---:|:---:|:---:|
| `Inner` (InnerLike) | ✓ | ✓ | ✓ | ✓ |
| `Cross` (InnerLike) | ✓ | ✓ | ✓ | ✓ |
| `LeftOuter` | ✗ | ✓ | ✓ | ✓ |
| `RightOuter` | ✓ | ✗ | ✓ | ✓ |
| `FullOuter` | ✗ | ✗ | ✓ | ✓ |
| `LeftSemi` | ✗ | ✓ | ✗ | ✓ |
| `LeftAnti` | ✗ | ✓ | ✗ | ✓ |
| `ExistenceJoin` | ✗ | ✓ | ✗ | ✓ |

**중요 귀결**:
- `FullOuter` join에서 `BROADCAST` hint는 **양쪽 다 거부됨** (build side
  없음) → `createBroadcastHashJoin(true)` = `None` → SMJ/SHJ hint 경로로
  이어지거나 최종 `createJoinWithoutHint()`로 fallback.
- `LeftOuter` join에서 `BROADCAST` hint를 **왼쪽**에 걸면 거부됨
  (`canBuildBroadcastLeft(LeftOuter) = false`). 오른쪽에 걸어야 broadcast
  가능. 대칭으로 `RightOuter`의 오른쪽도 거부.
- `LeftSemi`/`LeftAnti`/`ExistenceJoin`은 **오른쪽만** broadcast/SHJ 가능.

#### `getBuildSide` (private, joins.scala:495-511)

양쪽 다 가능 → 작은 쪽. 한쪽만 가능 → 그쪽. 둘 다 불가 → `None`. 이
조합이 hint가 "조용히 거부"되는 가장 흔한 경로.

#### `canBuildLocalHashMapBySize` (private, joins.scala:519-521)

```scala
plan.stats.sizeInBytes < conf.autoBroadcastJoinThreshold * conf.numShufflePartitions
```

기본 10 MB × 200 = 2 GB. SHJ 자동 선택 (hint 없는 경로)의 상한.

#### `muchSmaller` (private, joins.scala:531-533)

```scala
a.stats.sizeInBytes * conf.getConf(SQLConf.SHUFFLE_HASH_JOIN_FACTOR) <= b.stats.sizeInBytes
```

`SHUFFLE_HASH_JOIN_FACTOR` (설정 `spark.sql.shuffle.hashJoinFactor`,
기본 3) — `a` 측이 적어도 이 배수만큼 작아야 SHJ 채택. SHJ는 hash map
build 비용이 sort보다 크므로, 한쪽이 **충분히 작을 때만** 이득.

#### `getShuffleHashJoinBuildSide` — `preferSortMergeJoin` 관문

(joins.scala:321-350) no-hint 경로에서 SHJ를 선택하려면:

```scala
!conf.preferSortMergeJoin && canBuildLocalHashMapBySize(side, conf) && muchSmaller(side, other, conf)
```

`spark.sql.join.preferSortMergeJoin` (기본 **true**)이 켜져 있으면
SHJ 자동 선택은 비활성화. `PREFER_SHUFFLE_HASH` hint나
`forceApplyShuffledHashJoin` (테스트 전용) 우회 경로로만 SHJ 발동
가능.

### 3-5. Analyzer 단계의 hint 처리

`ResolveHints`(`sql/catalyst/.../analysis/ResolveHints.scala`)가 SQL
`/*+ BROADCAST(t) */` 주석과 `DataFrame.hint(...)`를 `JoinHint` 노드로
해결. 이 파일은 아직 ingest 전 — analysis 단계 세부는 후속 ingest
대상.

## L4. Performance Levers

### hint가 무시되는 정확한 조건 (verified)

| 상황 | 무시 여부 | 소스 증거 |
|---|---|---|
| `BROADCAST` hint on `FullOuter` 양쪽 | **무시** (양쪽 build side 거부) | joins.scala:379-391 |
| `BROADCAST` hint on `LeftOuter` 왼쪽 | **무시** | joins.scala:381 (`case _: InnerLike \| RightOuter => true`) |
| `BROADCAST` hint on `RightOuter` 오른쪽 | **무시** | joins.scala:388 (RightOuter 미포함) |
| `BROADCAST` hint on `LeftSemi`/`LeftAnti` 왼쪽 | **무시** | joins.scala:388 (Left* 미포함) |
| `BROADCAST` hint but 크기 > threshold | **적용** (hintOnly=true 경로는 size 검사 생략) | joins.scala:303-307 |
| `MERGE` hint but 키가 unsortable (Map, 일부 UDT 등) | **무시** | SparkStrategies.scala:274 (`RowOrdering.isOrderable(leftKeys)`) |
| `SHUFFLE_HASH`/`MERGE` hint on 비-equi join | **경고 + 무시** | SparkStrategies.scala:207-214 (`checkHintNonEquiJoin`) |
| `SHUFFLE_REPLICATE_NL` hint on non-`InnerLike` | **무시** | SparkStrategies.scala:283 (`joinType.isInstanceOf[InnerLike]` 체크) |
| 단일 `/*+ BROADCAST */` hint에 대해 `canBuildBroadcastLeft/Right` 둘 다 true + 양쪽 크기 비슷 | **작은 쪽을 build side로** | joins.scala:500-503 (`getSmallerSide`) |

모든 경우 조용한 fallback — `createJoinWithoutHint()`가 호출된다.
경고 로그는 `checkHintNonEquiJoin`(비-equi) 및 `checkHintBuildSide`
(잘못된 build side hint)에서만 발생.

### 튜닝 knob

1. **`spark.sql.autoBroadcastJoinThreshold`** (10 MB, since 1.1.0).
   hint 없는 자동 broadcast의 크기 상한. `-1`이면 자동 broadcast 전면
   비활성화. Hive-managed 테이블에서는 `ANALYZE TABLE ... COMPUTE
   STATISTICS noscan`이 필수.

2. **`spark.sql.broadcastTimeout`** (300초, since 1.3.0). driver가
   broadcast relation을 materialize 하는 최대 대기 시간. 초과 시
   `SparkException` 발생.

3. **hint는 `canBroadcastBySize` 검사를 우회한다.** `createBroadcastHashJoin(true)`
   (hintOnly=true 경로)은 `getBroadcastBuildSide`에서 `hintToBroadcastLeft/Right`
   만 확인하고 크기 검사(joins.scala:303-307)는 수행하지 않는다. 즉
   hint가 있으면 테이블이 1 GB든 10 GB든 broadcast를 강제한다 —
   OOM까지 포함해. 단, 위 매트릭스에서 "무시"로 분류된 join 타입
   조합은 hint도 우회할 수 없다 (`canBuildBroadcastLeft/Right`가
   매트릭스를 통과하지 못하면 `getBuildSide`가 `None` 반환).

4. **`spark.sql.join.preferSortMergeJoin`** (기본 **true**). no-hint
   경로에서 SHJ 자동 선택을 막는 스위치. 기본이 `true`이므로 **SHJ는
   거의 hint 없이는 선택되지 않는다** — `PREFER_SHUFFLE_HASH` 또는
   `SHUFFLE_HASH` hint를 명시해야 함. 이 default를 알지 못하면 "왜
   작은 쪽이 broadcast도 안 되고 SMJ로만 돌아가는가?"의 진단이
   어렵다.

5. **`spark.sql.shuffle.hashJoinFactor`** (기본 **3**, Spark 3.3+).
   no-hint SHJ 선택의 크기 불균형 임계값. `small_side × factor ≤
   large_side`가 돼야 자동 SHJ. 한쪽이 다른 쪽의 1/3 이하가 아니면
   자동 SHJ는 발동 안 함 (그래도 hint로는 강제 가능).

6. **AQE 런타임 변환은 별개다.** AQE는 shuffle 완료 후 stats를 받아
   `canBroadcastBySize`를 **runtime 경로**로 다시 평가 (joins.scala:370,
   `plan.stats.isRuntime == true`일 때 `ADAPTIVE_AUTO_BROADCASTJOIN_THRESHOLD`
   참조). 이는 플래닝 단계의 hint와 무관하게 SMJ → BHJ 전환을
   시도한다. 플래닝 단계에서 hint로 이미 BHJ로 결정된 경우는 AQE 변환
   대상이 아니다 — hint가 AQE 전환보다 먼저 적용되므로.
   [05-cost-and-aqe/aqe-overview.md] 참고.

## L5. How to Observe

`executedPlan`의 operator 클래스명이 1차 단서:

```python
print(df.queryExecution.executedPlan)
# 찾는 operator:
#   BroadcastHashJoin  → createBroadcastHashJoin 성공
#   SortMergeJoin      → createSortMergeJoin 성공
#   ShuffledHashJoin   → createShuffleHashJoin 성공
#   BroadcastNestedLoopJoin → 비-equi 경로 또는 최종 fallback
#   CartesianProduct   → createCartesianProduct (비-equi + SHUFFLE_REPLICATE_NL hint)
```

### hint 무시 여부 진단

SQL `/*+ BROADCAST(r) */` 또는 `.hint("broadcast")`를 달았는데
`executedPlan`이 `SortMergeJoin`으로 나온다면, L4 매트릭스 중 해당
행을 확인:

1. **join type 확인**. `df.queryExecution.optimizedPlan`의 `Join [...]`
   라인에서 `Inner`/`LeftOuter`/`RightOuter`/`FullOuter` 등을 확인.
2. build side 결정 확인. `executedPlan`의
   `SortMergeJoin(... , keys, ...)` 앞뒤에 `Sort`/`Exchange` 노드가
   양쪽 대칭으로 있다면 broadcast는 시도조차 되지 않은 것.
3. hint가 analysis 단계에서 보존됐는지: `df.queryExecution.analyzed`
   에서 `JoinHint(strategy=Some(BROADCAST))` 확인. 있으면 analysis
   단계까지는 살아 있음.
4. 경고 로그: `checkHintBuildSide`/`checkHintNonEquiJoin`이
   `hintErrorHandler.joinHintNotSupported`를 호출하면 driver log에
   "join hint not supported" 메시지 기록 (default handler는 logger
   WARN 수준).

### no-hint path 진단

```python
# 왜 BHJ 안 되나?
df.explain(mode="cost")
# 각 노드의 Statistics(sizeInBytes=...) 확인.
# sizeInBytes가 autoBroadcastJoinThreshold보다 큰가?
# sizeInBytes가 Long.MaxValue 근처 (기본값) 이면 ANALYZE 누락.
```

자세한 통계 관찰: [05-cost-and-aqe/leveraging-statistics.md].

### SHJ가 왜 안 되는지

```python
spark.conf.get("spark.sql.join.preferSortMergeJoin")  # true면 자동 SHJ 차단
spark.conf.get("spark.sql.shuffle.hashJoinFactor")    # 기본 3
```

자동 경로로 SHJ가 보고 싶으면 `preferSortMergeJoin=false` + 한쪽이
다른 쪽의 `1/hashJoinFactor` 이하. 또는 `/*+ SHUFFLE_HASH(t) */`
hint로 강제.

## Version notes

- 이 페이지는 **Spark 3.5.6 소스 기준 verified**. 소스 경로:
  `sql/core/.../SparkStrategies.scala` lines 172-411 (`JoinSelection`
  object), `sql/catalyst/.../optimizer/joins.scala` lines 286-536
  (`JoinSelectionHelper` trait).
- 3.5.6 lightweight tag SHA: `303c18c74664f161b9b969ac343784c088b47593`.
- Spark 4.1.1 docs는 hint 이름과 우선순위를 3.5.6과 동일하게 기술하고
  구조만 재편했으므로 **동작 변화 없음으로 추정**되나, 4.1.1 소스
  diff가 ingest되기 전까지 `spark_versions: ["4.1"]` 주장은 보류.
  후속 ingest 대상.
- 3.2–3.4 버전 동작은 별도 ingest 필요. 특히
  `spark.sql.shuffle.hashJoinFactor`는 3.3.0 도입이므로 이전 버전에서는
  `muchSmaller` 로직 자체가 없을 가능성.
- `BroadcastNestedLoopJoinExec`의 `NO_BROADCAST_AND_REPLICATION` hint 지원은
  비교적 최근 추가 — 정확한 도입 버전 확인은 `hints.scala` ingest 시.
- `BROADCASTJOIN`, `MAPJOIN` 동의어는 파서 레벨에서 `BROADCAST`로
  정규화 — 파서 소스 (`SqlBase.g4` 또는 `AstBuilder.scala`) 미ingest.
- 설정 기본값:
  - `spark.sql.autoBroadcastJoinThreshold`: 10 MB (since 1.1.0).
  - `spark.sql.broadcastTimeout`: 300 sec (since 1.3.0).
  - `spark.sql.join.preferSortMergeJoin`: true (since 2.0.0, 추정).
  - `spark.sql.shuffle.hashJoinFactor`: 3 (since 3.3.0, 추정).

## L3 해소된 갭 (이전 tentative 버전 대비)

- "어느 join 타입이 어느 build side를 허용하는지 docs는 명시하지 않음"
  → 해소: L3-4의 매트릭스가 `canBuildBroadcastLeft/Right` +
  `canBuildShuffledHashJoinLeft/Right`로부터 직접 도출됨.
- "hint가 무시되는 정확한 조건" → 해소: L4 첫 표가 소스 증거 포함.
- "hintOnly 경로가 size 검사를 우회하는지" → 해소:
  `getBroadcastBuildSide`의 hintOnly=true 분기가 `hintToBroadcast*`만
  확인 (joins.scala:303-307).

## 남은 tentative 조각

- **analysis 단계의 hint 해결**: `ResolveHints.scala` 아직 ingest 전 —
  SQL `/*+ ... */`가 어떤 규칙으로 `JoinHint` 노드가 되는지, hint 충돌
  해결 세부는 미확인.
- **`hintErrorHandler` 구현**: 기본 handler의 로그 레벨과 사용자
  정의 handler API는 별도 확인 필요 — `conf.hintErrorHandler` 경로
  추적.
- **4.1.1 소스 diff**: 3.5.6과 동일성 확인 필요 (후속 ingest).
