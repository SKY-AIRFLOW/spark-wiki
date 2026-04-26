---
stage: 03-logical-optimization
title: CTE Inline — WithCTE/CTERelationDef/CTERelationRef 소거
spark_versions: ["3.5"]
sources:
  - raw/explain/2026-04-19-query-C-window-moving-avg.txt
  - raw/spark-source/v3.5.6/sql/catalyst/src/main/scala/org/apache/spark/sql/catalyst/optimizer/InlineCTE.scala
  - raw/spark-source/v3.5.6/sql/catalyst/src/main/scala/org/apache/spark/sql/catalyst/optimizer/Optimizer.scala
  - raw/spark-source/v3.5.6/sql/catalyst/src/main/scala/org/apache/spark/sql/catalyst/analysis/CTESubstitution.scala
  - raw/spark-source/v3.5.6/sql/core/src/main/scala/org/apache/spark/sql/execution/SparkOptimizer.scala
last_verified: 2026-04-26
confidence: verified
related:
  - 00-overview/execution-pipeline.md
  - 02-analysis/analyzer-overview.md
  - 03-logical-optimization/column-pruning.md
  - 03-logical-optimization/predicate-pushdown.md
  - case/2026-04-19-query-C-window-moving-avg.md
configs:
  - spark.sql.optimizer.excludedRules
  - spark.sql.legacy.ctePrecedencePolicy
apis:
  - SQL WITH <name> AS (...) SELECT ...
  - DataFrame.createOrReplaceTempView (유사)
tags: [cte, catalyst, optimizer, inline, substitution]
---

## L1. What & Why

CTE (Common Table Expression, `WITH name AS (...)`)는 SQL 레벨에서 쿼리의
재사용 가능한 subquery를 이름으로 정의하는 구문이다. Catalyst는 이 구문을
plan 상에서 두 종류의 전용 논리 노드로 표현한 뒤 (`CTERelationDef`,
`CTERelationRef`), 논리 최적화 중에 조건이 맞으면 참조 지점에 실제 내용을
**inline**한다.

CTE 처리는 **두 단계에 걸쳐 있다**:
1. **단계 02 (Analysis)** — `CTESubstitution` 규칙 (`CTESubstitution.scala:59`):
   parser가 만든 `UnresolvedWith` / CTE 이름의 `UnresolvedRelation`을
   `WithCTE` + `CTERelationDef` + `CTERelationRef` 구조로 정규화 (또는
   LEGACY 모드/Command에서는 즉시 inline).
2. **단계 03 (Logical Optimization)** — `InlineCTE` 규칙 (`InlineCTE.scala:49`):
   분석 후 남은 `CTERelationRef`에 대해 inline 가능 조건을 검사하고,
   가능하면 참조 지점에 body subtree 치환.

이 두 단계 분리는 명시적 invariant이다. `InlineCTE.scala:58`의 한 줄
주석이 검증한다:

> `// CTEs in SQL Commands have been inlined by` `CTESubstitution` `already`

즉 SQL Command (INSERT, CTAS 등)는 분석 단계에서 이미 CTE가 풀린
상태로 들어오고, `InlineCTE`는 일반 쿼리 plan에서 남은 reference만
처리한다. 두 규칙은 의무를 분담하지, 같은 일을 두 번 하지 않는다.

Inline의 본질은 "named subquery를 일반 subtree로 치환"이다. Inline 후
plan은 CTE를 쓰지 않은 동등한 쿼리와 구조적으로 같아지며, 이후 규칙
(column pruning, predicate pushdown 등)이 경계를 넘어 자유롭게 최적화할
수 있다.

Java 개발자 관점에서 CTE inline은 **JIT의 private method inlining**과
정확히 대응된다. 참조가 적고 단순한 helper method는 호출 지점에 inline
되어 인라이너가 더 공격적인 최적화 (escape analysis, dead store
elimination)를 할 수 있다. Catalyst도 마찬가지로 "inline 후 최적화 경계가
사라지면 더 많은 규칙이 발동"하기 위해 이 변환을 먼저 수행한다.

Inline은 **항상** 안전하지는 않다. CTE가 여러 번 참조되고 재계산 비용이
크면 inline이 되려 성능에 해가 될 수 있다 — 이 경우 Catalyst는 inline을
건너뛰고 참조를 유지, SparkOptimizer의 `Replace CTE with Repartition`
batch (`SparkOptimizer.scala:111`)가 shuffle reuse 경로로 실현한다.

## L2. Inputs & Outputs

### CTE를 표현하는 세 노드 (Analyzed 단계)

- **`WithCTE`**: CTE를 가진 쿼리 전체를 감싸는 루트. 자식은 `CTERelationDef`
  목록 + main query.
- **`CTERelationDef id, resolved`**: CTE의 정의 subtree. `id`는 Analyzer가
  부여한 고유 식별자. `resolved` flag는 내부 resolution 상태.
- **`CTERelationRef id, resolved, [cols], recursive`**: CTE의 참조. main
  query 안에서 CTE 이름이 등장한 지점을 대체. 같은 `id`로 `CTERelationDef`
  를 가리킴.

**입력** (Analyzed LogicalPlan):

```
WithCTE
:- CTERelationDef <id>, <resolved>
:  +- <CTE body subtree>
+- <main query subtree containing
     CTERelationRef <same id>, <resolved>, [col#id, ...], <recursive>>
```

**출력** (Optimized LogicalPlan, inline 적용 시):

```
<main query subtree with CTERelationRef replaced by the CTE body>
```

`WithCTE`, `CTERelationDef`, `CTERelationRef` 세 노드 모두 사라지고, 참조
지점에 body가 그대로 substitute된다. 참조가 여럿이고 inline 조건 만족 시
각 지점에 별개의 복제본이 들어간다 — 후술하는 `Project + DeduplicateRelations`
경로 (`InlineCTE.scala:187-198`)로 attribute exprId 충돌이 방지된다.

### 쿼리 C 관찰 (Analyzed, 라인 27-40)

```
WithCTE
:- CTERelationDef 0, false
:  +- SubqueryAlias monthly_avg
:     +- Aggregate [agent_gu#377, deal_year#370, deal_month#371],
:          [..., avg(deal_amount#373) AS avg_amount#364,
:                count(1) AS trade_count#365L]
:        +- Filter agent_gu#377 LIKE 서울%
:           +- SubqueryAlias nessie.real_estate.apt_trades
:              +- RelationV2[...] nessie.real_estate.apt_trades ...
+- Sort [agent_gu#377 ASC NULLS FIRST, ...]
   +- Project [..., moving_avg_6m#363]
      +- Project [..., moving_avg_6m#363, moving_avg_6m#363]
         +- Window [avg(avg_amount#364) ...]
            +- Project [agent_gu#377, deal_year#370, deal_month#371,
                        avg_amount#364, trade_count#365L]
               +- SubqueryAlias monthly_avg
                  +- CTERelationRef 0, true,
                       [agent_gu#377, deal_year#370, deal_month#371,
                        avg_amount#364, trade_count#365L], false
```

Optimized (라인 46-50) — 세 노드 모두 사라짐:

```
Sort [agent_gu#377 ASC NULLS FIRST, ...]
+- Window [avg(avg_amount#364) ...]
   +- Aggregate [agent_gu#377, deal_year#370, deal_month#371],
        [..., avg(deal_amount#373) AS avg_amount#364,
              count(1) AS trade_count#365L]
      +- Filter (isnotnull(agent_gu#377) AND StartsWith(agent_gu#377, 서울))
         +- RelationV2[deal_year#370, deal_month#371, deal_amount#373,
                       agent_gu#377] nessie.real_estate.apt_trades
```

- 참조 횟수 = 1회 (main query에 `CTERelationRef 0` 한 번만 등장) → inline
  안전. `shouldInline`의 첫 분기 `refCount == 1`에서 즉시 true
  (`InlineCTE.scala:75`).
- Inline 후 `Aggregate` 노드가 main query의 `Window` 바로 아래에 배치됨.
- Column pruning이 경계를 넘어 작동 → Aggregate가 쓰지 않는 컬럼들까지
  RelationV2 스키마에서 제거 (17 → 4 컬럼).

## L3. Key Mechanisms

### 두 단계 처리

| 단계 | 규칙 | 입력 | 출력 |
|---|---|---|---|
| 02 Analysis | `CTESubstitution` (`CTESubstitution.scala:59`) | `UnresolvedWith`, CTE 이름의 `UnresolvedRelation` | `WithCTE` + `CTERelationDef` + `CTERelationRef` (CORRECTED 모드) 또는 inline 직접 (LEGACY/Command) |
| 03 Logical Opt | `InlineCTE()` (`InlineCTE.scala:49`, batch wiring `Optimizer.scala:167-168`) | `WithCTE/Def/Ref` 트리 | `shouldInline` 결과에 따라 inline body 또는 그대로 |

### CTESubstitution의 3가지 모드

`CTESubstitution.scala:70-78`:

```scala
val (substituted, firstSubstituted) =
  LegacyBehaviorPolicy.withName(conf.getConf(LEGACY_CTE_PRECEDENCE_POLICY)) match {
    case LegacyBehaviorPolicy.EXCEPTION =>
      assertNoNameConflictsInCTE(plan)
      traverseAndSubstituteCTE(plan, isCommand, Seq.empty, cteDefs)
    case LegacyBehaviorPolicy.LEGACY =>
      (legacyTraverseAndSubstituteCTE(plan, cteDefs), None)
    case LegacyBehaviorPolicy.CORRECTED =>
      traverseAndSubstituteCTE(plan, isCommand, Seq.empty, cteDefs)
  }
```

| 모드 | 의미 | 사용 |
|---|---|---|
| `EXCEPTION` (default 3.x) | 이름 충돌 시 예외, 그 외는 CORRECTED와 동일 — nested CTE의 inner 우선 + `WithCTE` 정규화 | 신규 코드 권장 |
| `LEGACY` | Spark 2.x 호환 — outer CTE 우선 + 즉시 inline | 마이그레이션 임시 호환 |
| `CORRECTED` | 이름 충돌 무시, nested의 inner 우선, `WithCTE` 정규화 | EXCEPTION의 충돌 검사 없는 버전 |

`spark.sql.legacy.ctePrecedencePolicy` 설정으로 선택. case C는 이름 충돌이
없으므로 EXCEPTION/CORRECTED 어느 쪽으로도 같은 결과 — `WithCTE` 노드가
Analyzed plan에 등장한다 (case C raw explain line 27).

`isCommand` 분기 (`CTESubstitution.scala:64-67`):

```scala
val isCommand = plan.exists {
  case _: Command | _: ParsedStatement | _: InsertIntoDir => true
  case _ => false
}
```

Command/ParsedStatement/InsertIntoDir인 경우 traversal 동작이 다름 — 이
경우 분석 단계에서 즉시 inline. `InlineCTE.scala:58`의 주석이 가정하는
invariant.

### InlineCTE — case class 시그니처

`InlineCTE.scala:49`:

```scala
case class InlineCTE(alwaysInline: Boolean = false) extends Rule[LogicalPlan] {
  override def apply(plan: LogicalPlan): LogicalPlan = {
    if (!plan.isInstanceOf[Subquery] && plan.containsPattern(CTE)) {
      val cteMap = mutable.SortedMap.empty[Long, (CTERelationDef, Int, mutable.Map[Long, Int])]
      buildCTEMap(plan, cteMap)
      cleanCTEMap(cteMap)
      val notInlined = mutable.ArrayBuffer.empty[CTERelationDef]
      val inlined = inlineCTE(plan, cteMap, notInlined)
      if (notInlined.isEmpty) inlined else WithCTE(inlined, notInlined.toSeq)
    } else {
      plan
    }
  }
  ...
}
```

`alwaysInline: Boolean` 파라미터는 외부에서 강제 inline을 지시할 때 사용
(예: subquery 처리 경로). default `false`로 사용되며, `Optimizer.scala:167-168`
의 `Batch("Inline CTE", Once, InlineCTE())` 호출이 default 인스턴스를
사용한다. **`Once` 실행** — fixed-point 아님. inline은 한 번의 트리 순회로
완결된다.

### shouldInline 함수 (3가지 OR 조건)

`InlineCTE.scala:70-78`:

```scala
private def shouldInline(cteDef: CTERelationDef, refCount: Int): Boolean = alwaysInline || {
  // We do not need to check enclosed `CTERelationRef`s for `deterministic` or `OuterReference`,
  // because:
  // 1) It is fine to inline a CTE if it references another CTE that is non-deterministic;
  // 2) Any `CTERelationRef` that contains `OuterReference` would have been inlined first.
  refCount == 1 ||
    cteDef.deterministic ||
    cteDef.child.exists(_.expressions.exists(_.isInstanceOf[OuterReference]))
}
```

세 분기:
1. **`refCount == 1`** — main query 전체에서 단 한 번 참조 → inline해도
   복제 비용 없음. 가장 흔한 경로.
2. **`cteDef.deterministic`** — body가 결정론적이면 다중 참조여도 복제 가능
   (각 복제본이 같은 결과 보장). `rand()`, `current_timestamp()`, UDF
   (nondeterministic 표시) 등이 포함되면 false → inline 안 됨.
3. **`OuterReference` 포함** — outer query의 attribute를 참조하는 CTE는
   correlated subquery 형태로, 이미 처리 단계에서 inline되어야 함 — 안전
   장치성 분기.

### case C가 inline된 정확한 이유

case C의 `monthly_avg` CTE는 main query에서 정확히 한 번 참조 (`CTERelationRef 0`
이 1회 등장). `buildCTEMap` (`InlineCTE.scala:93-141`)이 `refCount = 1`을
기록 → `shouldInline`의 첫 분기에서 즉시 true → inline 적용. **이 한 줄이
case C Optimized plan에서 `WithCTE`/`Def`/`Ref` 세 노드가 모두 사라진
정확한 메커니즘**이다. `cteDef.deterministic` 검사는 이 케이스에서 실행
조차 되지 않는다 (OR의 short-circuit).

### 3-pass 알고리즘

| Pass | 함수 | 라인 | 역할 |
|---|---|---|---|
| 1 | `buildCTEMap` | `InlineCTE.scala:93-141` | plan 순회하며 `CTERelationDef`마다 `(id → (def, refCount, outRefMap))` 맵 생성. `CTERelationRef` 만날 때마다 그 cteId의 refCount 증가. nested CTE의 outgoing reference도 별도 맵에 기록. |
| 2 | `cleanCTEMap` | `InlineCTE.scala:150-162` | refCount==0인 CTE가 다른 CTE를 참조하면 그 참조 카운트 감산 (전이적 dead CTE 제거). 역순(`reverse`) 순회로 의존성 처리. |
| 3 | `inlineCTE` | `InlineCTE.scala:164-213` | 재귀적으로 `CTERelationRef`를 만나면 `shouldInline` 호출, true면 body로 치환. `outputSet` 일치 시 직접 `cteDef.child` 반환, 불일치 시 `Project + DeduplicateRelations` 래핑. |

`outputSet` 일치 시 직접 `cteDef.child` 반환 (`InlineCTE.scala:185-186`),
불일치 시 attribute 매핑을 위한 `Project` + `DeduplicateRelations`로
exprId 충돌 방지 (`InlineCTE.scala:187-198`). 후자는 동일 CTE를 두 번
이상 inline할 때 각 복제본의 attribute exprId가 충돌하지 않도록 새 ID
부여.

## L4. Performance Levers

### shouldInline의 3가지 OR 조건이 lever

사용자가 직접 토글하는 boolean 설정은 없지만, **CTE body 작성 방식**이
결정 메커니즘을 좌우한다:
- `rand()`, `current_timestamp()`, UDF (nondeterministic 표시) 포함 시
  `cteDef.deterministic = false` → 다중 참조 시 inline 안 됨 → shuffle
  reuse 경로로 빠짐.
- 단일 참조 CTE는 항상 inline (1번 분기) — body 결정성 무관.
- 따라서 다중 참조하면서 비결정 표현식을 포함하는 CTE는 자동으로 reuse
  경로로 빠진다 — 의도한 동작이 아니라면 deterministic UDF로 교체하거나
  참조를 1회로 줄여야 한다.

### `spark.sql.legacy.ctePrecedencePolicy`

| 값 | 동작 |
|---|---|
| `EXCEPTION` (default) | 이름 충돌 시 예외 발생 |
| `LEGACY` | Spark 2.x 호환 — outer CTE 우선 + 즉시 inline |
| `CORRECTED` | 이름 충돌 무시 + inner 우선 |

신규 코드는 default 유지 권장. `LEGACY`는 마이그레이션 임시 용도.

### `alwaysInline = true` 강제 경로

`InlineCTE` case class의 `alwaysInline` 파라미터를 `true`로 호출하는
경로는 catalyst 내부의 subquery 처리 등에서 사용. 일반 `defaultBatches`
경로는 default `false` 인스턴스를 사용하므로 사용자 lever는 아님.

### 다중 참조 CTE의 SparkOptimizer 경로

`SparkOptimizer.scala:111`:

```scala
Batch("Replace CTE with Repartition", Once, ReplaceCTERefWithRepartition)
```

inline되지 않은 CTE 참조에 대해 명시적 `Repartition` 노드를 삽입해 후속
exchange reuse가 가능하도록 표시. 즉 다중 참조 + non-deterministic CTE는
inline 대신 **shuffle 결과 재사용** 경로로 실현된다. `ReplaceCTERefWithRepartition`
규칙 자체의 메커니즘은 이 페이지 범위 밖 (후속 ingest 후보).

`spark.sql.optimizer.excludedRules`에 `InlineCTE`를 추가하면 모든 CTE가
이 reuse 경로로 빠지게 됨 — 디버깅 외에는 거의 사용하지 않음.

### Recursive CTE

Spark 3.5는 `WITH RECURSIVE`를 제한적 지원. `CTERelationRef`의 `recursive`
flag로 표시. `InlineCTE.scala`는 recursive 케이스 분기를 별도로 가지지
않고 일반 inline 경로를 따르며, recursive CTE의 의미는 별도 분석/실행
경로에서 처리. 자세한 동작은 후속 ingest 후보.

## L5. How to Observe

- `df.queryExecution.analyzed.toString` 또는 `df.explain(mode="extended")`
  → Analyzed Logical Plan 섹션에서 `WithCTE`, `CTERelationDef id, ...`,
  `CTERelationRef id, ..., [cols], recursive` 세 노드 확인.
- `df.queryExecution.optimizedPlan.toString` → 같은 세 노드가 모두 사라지고
  CTE body가 main query에 병합되었는지 확인. case C가 정확히 이 사례.
- 세 노드가 **남아 있다면** inline 미적용 — 참조 횟수, 복잡도, deterministic
  여부를 확인. `shouldInline`의 3가지 분기 중 어디서 false가 됐는지 추론
  가능 (refCount는 plan에서 직접 셀 수 있음, deterministic은 body의
  표현식 검토).
- Physical plan에서 CTE가 어떻게 실현됐는지: inline되지 않은 경우
  `ReusedExchange` 또는 `InMemoryRelation` 노드 등장 여부 관찰.
- 강제 비활성화 실험: `spark.conf.set("spark.sql.optimizer.excludedRules",
  "org.apache.spark.sql.catalyst.optimizer.InlineCTE")` 후 case C 같은
  단일 참조 CTE 쿼리 재실행 → `WithCTE`/`Def`/`Ref` 세 노드 잔존 +
  Physical plan에서 어떻게 실현되는지 비교. 좋은 probe 실험.

> 📝 probe 페이지 미작성. 후속 `/probe 03-logical-optimization/cte-inline`
> 대상 — "1회 참조 vs 다중 참조 CTE의 plan 차이 비교 + InlineCTE
> 비활성화 시 `ReplaceCTERefWithRepartition` 경로 관찰".
