# `raw/spark-source/` — 규약

## raw/ 전체 원칙 (재확인)

`raw/`는 **immutable한 진실의 원천**이다. 원칙적으로 외부 소스를
`raw/` 하위에 **복사**해 저장한다. 외부 URL은 사라질 수 있으므로
링크만 유지하는 것은 허용되지 않는다.

## `raw/spark-source/`의 예외

Spark 소스 코드는 전체 체크아웃이 수 GB에 달해 레포에 통째로
복사하는 것이 실용적이지 않다. 따라서 이 디렉터리에 한해 다음
예외를 허용한다.

**커밋 해시와 GitHub 경로를 기록하고, 필요할 때 해당 해시에서
특정 파일/라인 범위를 fetch해 사용한다.**

## 보관 방식

두 가지 옵션을 허용한다.

1. **스냅샷 저장** — `/ingest` 사이클에서 실제로 인용한 파일/범위를
   그 시점의 해시 기준으로 이 디렉터리에 직접 저장한다. 경로는
   Apache Spark 레포의 원본 경로를 반영한다:
   ```
   raw/spark-source/<commit-hash>/<original-path>
   예: raw/spark-source/v3.5.1/sql/catalyst/.../Optimizer.scala
   ```
   해시 디렉터리 밑에 부분 트리만 존재해도 무방하다 (실제로
   인용된 파일만).

2. **해시 + 경로 레퍼런스** — 파일을 복사하지 않고, 아래의
   `sources-manifest.md`에 한 줄 항목으로 기록한다. `/ingest`
   사이클 시작 시 해당 해시의 파일을 fetch해 검토하고, 요약이 끝나면
   fetch된 임시본을 버린다. 이 방식은 큰 파일에 한정해 사용한다.

단계 페이지의 `sources:` frontmatter는 어느 경우든 `raw/spark-source/...`
형식의 경로로 인용한다. 옵션 2의 경우, `sources-manifest.md`의
항목 ID를 함께 기재한다.

## `sources-manifest.md`

옵션 2로 fetch-only 처리한 소스들의 대장. 최초 /ingest 전까지는
비어 있다.

## 왜 이 예외를 두는가

- Spark 소스의 "증거 고정성(version pin)"만 확보되면 immutable
  요건은 만족된다. 복사본 자체가 목적이 아니라 **특정 커밋 시점의
  텍스트에 대한 주장**이 핵심이다.
- 커밋 해시 기반 GitHub URL은 영구적이다 (Apache Spark 레포가
  사라지지 않는 한). 블로그 포스트와 달리 rot 위험이 낮다.
- 전체 체크아웃 복사는 Wiki 레포의 signal-to-noise를 파괴한다.

## `/ingest` 사이클에서 이 디렉터리를 다루는 방법

1. 사용자가 `raw/spark-source/.../File.scala`를 가리키면, 그 경로에
   실제 파일이 존재하는지 먼저 확인한다.
2. 없으면 `sources-manifest.md`를 확인해 해시/경로 항목이 있는지
   본다.
3. 둘 다 없으면 사용자에게 어느 커밋 해시의 어느 경로를 ingest할지
   확인받는다. 그 후 옵션 1 (스냅샷) 또는 옵션 2 (manifest)를
   골라 저장한다.
4. 옵션 2의 경우 fetch 권한을 요청해야 할 수 있다 — 실행 전에
   사용자에게 확인한다.
