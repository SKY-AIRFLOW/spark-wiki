# Bootstrap 프롬프트 — Spark 내부 동작 Wiki 초기화

Repo 루트에 `CLAUDE.md`를 배치한 후 Claude Code에 이 프롬프트를 붙여넣으세요.
이 프롬프트는 **단 한 번**만 실행합니다.

---

Repo 루트의 `CLAUDE.md`를 주의 깊게 읽으세요. 이것이 당신의 운영
매뉴얼입니다. 다음을 이해했음을 확인하세요:

- 파이프라인 척추 (Section 1) — SQL에서 Executor까지 10개의 화살표
- 단계 순서로 정렬된 디렉터리 구조 (Section 3)
- 모든 단계 페이지에 필수인 Depth Ladder L1-L5 (Section 5)
- 네 가지 운영 명령: `/ingest`, `/query`, `/trace`, `/lint`, `/probe` (Section 7)

그 후 Section 10에 기술된 최초 실행 초기화를 수행하세요.

## 수행할 작업

1. 디렉터리 골격을 생성하세요:
   ```
   raw/spark-docs/
   raw/spark-source/
   raw/papers/
   raw/books/
   raw/blog/
   raw/talks/
   raw/explain/
   raw/codegen/
   raw/ui/

   wiki/00-overview/
   wiki/01-parsing/
   wiki/02-analysis/
   wiki/03-logical-optimization/
   wiki/04-physical-planning/
   wiki/05-cost-and-aqe/
   wiki/06-codegen/
   wiki/07-rdd-and-stages/
   wiki/08-scheduling/
   wiki/09-execution/
   wiki/_probes/
   wiki/case/

   _meta/
   _meta/_lookup/
   ```

2. `wiki/00-overview/execution-pipeline.md`를 다음만 포함하여 생성하세요:
   - 파이프라인 다이어그램 (ASCII 또는 Mermaid)
   - 각 화살표가 무슨 일을 하는지 한 문장 설명
   - (아직 비어 있는) 단계 디렉터리를 가리키는 `related:` 플레이스홀더
   - `confidence: tentative`인 frontmatter
     (아직 소스가 없는 scaffolding이므로)

3. 네 개의 meta 파일을 생성하세요:
   - `_meta/index.md` — 단계별로 묶인 빈 목록
   - `_meta/log.md` — 다음 한 줄의 항목:
     `## [YYYY-MM-DD] init | wiki가 파이프라인 skeleton으로 초기화됨`
   - `_meta/_lookup/config-index.md` — 테이블 헤더만:
     `| Config | Default | Stage | Pages |`
   - `_meta/_lookup/api-to-stage.md` — 테이블 헤더만:
     `| API | Stage | Pages |`

4. 그 외 어떤 콘텐츠 파일도 생성하지 마세요. Catalyst 페이지도, AQE
   페이지도, codegen 페이지도 만들지 마세요 — 전부 만들지 마세요.
   모든 단계 콘텐츠 페이지는 이후 실제 `/ingest` 또는 `/trace`에서만
   생성되어야 합니다.

5. 다음을 보고하세요:
   - 생성한 디렉터리 트리
   - `execution-pipeline.md`의 내용
   - 첫 `/ingest`에서 무엇을 수행할지에 대한 간단한 진술
   - `CLAUDE.md`에서 모호하게 느껴지는 부분 — 시작하기 전에 질문

5단계를 넘어 진행하지 마세요. 이 Wiki는 인내심 있는 구조입니다.
훈련 데이터의 사전지식이 아니라 실제 입력에서만 성장합니다.
