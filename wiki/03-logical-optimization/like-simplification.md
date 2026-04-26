---
stage: 03-logical-optimization
title: Like Simplification
spark_versions: ["3.5"]
sources:
  - raw/explain/2026-04-19-query-A-self-join-growth.txt
  - raw/explain/2026-04-19-query-B-row-number-top5.txt
  - raw/explain/2026-04-19-query-C-window-moving-avg.txt
  - raw/spark-source/v3.5.6/sql/catalyst/src/main/scala/org/apache/spark/sql/catalyst/optimizer/expressions.scala
last_verified: 2026-04-26
confidence: verified
related:
  - 03-logical-optimization/predicate-pushdown.md
  - case/2026-04-19-query-A-self-join-growth.md
  - case/2026-04-19-query-B-row-number-top5.md
  - case/2026-04-19-query-C-window-moving-avg.md
configs:
  - spark.sql.optimizer.excludedRules
apis:
  - SQL LIKE
tags: [optimizer, catalyst, expression-rewrite, like]
---

## L1. What & Why

`LikeSimplification`은 `LIKE 'pattern'`을 정규식 매칭 대신 더 가벼운
표현식 (`StartsWith` / `EndsWith` / `Contains` / `EqualTo`, 또는
`prefix%suffix`의 결합)으로 재작성한다. wildcard `%`의 위치에 따라 정확히
5가지 정규식 패턴으로 분류되며, 매치 시 대응 표현식으로 치환. 이 변환이
없으면 모든 LIKE 비교가 정규식 엔진을 거친다 — 단순 prefix/suffix/contains
매칭에는 과한 비용. case A/B/C 모두 `LIKE '서울%'` → `StartsWith(agent_gu, 서울)`
변환을 관찰한다.

## L2. Inputs & Outputs

입력: `Like(input, Literal('pattern'), escapeChar)`. 출력: 5종 표현식 중
하나, 또는 매치 안 되거나 escape char 포함 시 변경 없음.

## L3. Key Mechanisms

`raw/spark-source/v3.5.6/.../optimizer/expressions.scala:720`:

```scala
object LikeSimplification extends Rule[LogicalPlan] with PredicateHelper
```

### 5개 정규식 패턴

`expressions.scala:723-727`:

```scala
private val startsWith = "([^_%]+)%".r
private val endsWith = "%([^_%]+)".r
private val startsAndEndsWith = "([^_%]+)%([^_%]+)".r
private val contains = "%([^_%]+)%".r
private val equalTo = "([^_%]*)".r
```

`[^_%]+`는 wildcard `%`나 `_`이 포함되지 않은 부분 문자열을 매치 — 5개
정규식 자체가 single-char wildcard `_` 포함 패턴을 배제한다. `_` 들어간
LIKE는 매치 실패 → 정규식 엔진 그대로 사용.

### simplifyLike — 5분기 변환

`expressions.scala:729-757`. escape char 포함 시 None 반환 (skip), 그
외에는 5개 패턴 중 첫 매치 분기로 변환.

| 패턴 | 입력 예시 | 출력 |
|---|---|---|
| `prefix%` | `'서울%'` | `StartsWith(input, prefix)` |
| `%suffix` | `'%동'` | `EndsWith(input, suffix)` |
| `prefix%suffix` | `'서울%동'` | `Length(input) >= prefix+suffix.len AND StartsWith AND EndsWith` |
| `%infix%` | `'%구로%'` | `Contains(input, infix)` |
| no wildcard | `'서울특별시'` | `EqualTo(input, str)` |

### prefix%suffix의 Length 보장

`expressions.scala:747-749`의 `Length(input) >= prefix.length + postfix.length`
는 `'a' LIKE 'a%a'` 같은 false-positive 방지 invariant. `'a'`는
`StartsWith('a')` + `EndsWith('a')`를 동시에 만족하지만 길이 1로
의미상 매치 불가 — 길이 ≥ 2 가드가 차단.

case A/B/C의 `'LIKE 서울%'`은 단순 `startsWith` 분기라 이 Length invariant은
적용되지 않는다.

## L4. Performance Levers

`spark.sql.optimizer.excludedRules`로 `LikeSimplification` 제외 시 모든
LIKE 비교가 정규식 엔진을 통과 — 운영 환경에서는 거의 변경하지 않음.

> 📝 escape char 분기 (3가지 — invalid escape, escaped wildcard, escaped
> escape), `MultiLikeBase` (`LikeAll`/`LikeAny`), single-char wildcard `_`
> 처리는 후속 ingest 시 보강.

## L5. How to Observe

`df.queryExecution.optimizedPlan`에서 `Like(...)` 노드가 5종 단순 표현식으로
치환됐는지 관찰. case A/B/C 모두 Optimized plan에 `StartsWith(agent_gu, 서울)`
등장.

> 📝 BatchScan의 `filters=[..., agent_gu LIKE '서울%', ...]`은 변환 후에도
> 원본 LIKE 형태 유지 — Iceberg V2 connector가 별도 push-down 경로에서
> 원본을 받기 때문 (단계 04 `V2ScanRelationPushDown`, 후속 ingest).

> 📝 probe 페이지 미작성. 후속 `/probe 03-logical-optimization/like-simplification`
> 후보.
