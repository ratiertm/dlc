# Phase 2: E2E Spec Format - Context

**Gathered:** 2026-03-22
**Status:** Ready for planning

<domain>
## Phase Boundary

E2E Feature Spec의 포맷과 ID 체계를 정의한다. 각 기능을 사용자 액션부터 최종 저장소(DB/LLM/파일)까지의 왕복 플로우로 명세하는 구조화된 Markdown 포맷을 만들고, 이 포맷이 PLAN/DO/TEST 전체를 관통하여 활용되는 방식을 정의한다. 실제 PLAN/DO/TEST 어댑터 구현은 Phase 4-6에서 수행.

</domain>

<decisions>
## Implementation Decisions

### Spec 파일 포맷
- Markdown (.spec.md) 포맷 사용 — 사용자 가독성 우선, GSD 패턴과 일관
- frontmatter로 메타데이터(ID, feature, status 등) 관리
- 위치: `.lifecycle/features/{feature-name}/spec.md` — 프로젝트 단위 문서

### 5단계 체인 핵심 원칙
- **전체 왕복 플로우**: 사용자 액션 → API/함수 호출 → 서버/로직 처리 → 최종 저장소(DB/LLM/파일) → 응답 → 사용자 화면 업데이트
- **최종 저장소 명시 필수**: Processing 단계에 "어디에 저장/조회하는지"를 필수 필드로 포함
- 단계 수(5 vs 7)는 Claude's Discretion — 핵심은 왕복 플로우가 빠짐없이 커버되는 것

### Spec ID 체계
- 형식: `e2e-{feature}-{NNN}` (예: e2e-login-001, e2e-chat-002)
- 기능명 포함으로 직관적 식별
- 코드에 `// SPEC: e2e-{feature}-{NNN}` 주석으로 연결 — WHY+SEE 패턴과 유사

### Spec 활용 범위 (PLAN → DO → TEST)
- **PLAN**: Spec 작성 + 사용자 합의
- **DO**: Spec의 각 단계를 체크리스트로 추적 (Screen ✅ Connection ✅ Processing ⬜ ...)
- **TEST**: Spec의 각 단계별 개별 pass/fail 판정 + 리포트 생성

### Claude's Discretion
- 5단계 vs 7단계(+State Change, Side Effects) 최종 결정
- frontmatter 필드 상세 스키마
- Spec 템플릿의 구체적 섹션 구조
- deviation log 포맷 (DO에서 spec 이탈 시)

</decisions>

<canonical_refs>
## Canonical References

**Downstream agents MUST read these before planning or implementing.**

### 기존 스킬 구조
- `~/.claude/skills/dev-lifecycle/SKILL.md` — 현재 범용 스킬 (Phase 1 완성)
- `~/.claude/skills/dev-lifecycle/references/stage-transitions.md` — Stage 간 전환 조건, artifact 요구사항
- `~/.claude/skills/dev-lifecycle/templates/manifest.json` — artifact 등록 구조

### 프로젝트 정의
- `.planning/PROJECT.md` — 핵심 문제: "완성이라 해도 100% 기능 안 됨", E2E spec이 핵심 해결책
- `.planning/REQUIREMENTS.md` — Phase 2 요구사항: SPEC-01
- `.planning/research/FEATURES.md` — E2E spec 포맷 상세 분석 (YAML vs Markdown, 5-layer chain)
- `.planning/research/ARCHITECTURE.md` — spec-prototype 통합 아키텍처, .lifecycle/features/ 구조

### 이전 Phase
- `.planning/phases/01-foundation/01-CONTEXT.md` — templates/ 폴더 구조, .lifecycle/ 네임스페이스 결정

</canonical_refs>

<code_context>
## Existing Code Insights

### Reusable Assets
- `~/.claude/skills/dev-lifecycle/templates/manifest.json` — artifact 등록 패턴, stage별 required_inputs 참고 가능
- `~/.claude/skills/dev-lifecycle/references/stage-transitions.md` — Stage gate 조건, spec이 gate rule로 연결될 구조

### Established Patterns
- GSD의 PLAN.md 포맷 — frontmatter + markdown body + XML tasks
- ADR의 WHY+SEE 주석 — 코드에 문서를 연결하는 패턴 (SPEC 주석도 동일 패턴)

### Integration Points
- `.lifecycle/features/{name}/spec.md` — Phase 3(Prototype)의 prototype.html과 같은 디렉토리에 위치
- manifest.json의 `1_plan.outputs` — spec.md가 PLAN Stage의 산출물로 등록
- Stage gate rules — spec 존재 여부가 DO 진입 조건

</code_context>

<specifics>
## Specific Ideas

- 사용자 강조: "사용자에서 시작해서 맨 마지막 DB든 LLM이든 그 피드백이 다시 돌아오는 전체 flow" — 단방향이 아닌 왕복
- Spec은 프로젝트 단위 문서 (글로벌 스킬이 아님) — `.lifecycle/` 내부에 위치

</specifics>

<deferred>
## Deferred Ideas

None — discussion stayed within phase scope

</deferred>

---

*Phase: 02-e2e-spec-format*
*Context gathered: 2026-03-22*
