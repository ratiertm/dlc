# Phase 4: PLAN Stage - Context

**Gathered:** 2026-03-22
**Status:** Ready for planning

<domain>
## Phase Boundary

PLAN Stage 어댑터를 구현한다. 사용자가 기능을 요청하면 dev-lifecycle이 preflight check를 수행하고, E2E Spec과 클릭 가능한 Prototype을 생성한 뒤, 사용자가 브라우저에서 확인하고 승인하는 합의 게이트까지의 전체 플로우를 구현한다. GSD/PDCA는 별도로 작동하며, dev-lifecycle은 spec+prototype 생성을 독립적으로 담당한다.

</domain>

<decisions>
## Implementation Decisions

### Preflight Check
- 이전 회고 교훈이 최우선 — 같은 실수 반복 방지가 핵심 목적
- ADR, 미해결 gap, 기술부채도 체크하지만 회고 교훈이 가장 중요
- Preflight 실패 시 경고만 표시 — 차단하지 않고 사용자가 인지하도록

### 사용자 합의 게이트
- 사용자가 prototype.html을 브라우저에서 직접 열어 클릭하며 확인 (Claude 설명이 아닌 직접 체험)
- 피드백 루프: 수정 → prototype 재생성 → 재확인 — 사용자가 승인할 때까지 반복
- PLAN은 사용자가 명시적으로 승인하지 않으면 완료되지 않음 (hard gate)

### GSD/PDCA 역할 관계
- GSD가 프로젝트 계획/실행의 주도적 역할 (이미 진행 중인 것)
- PDCA가 품질 순환 보조
- dev-lifecycle은 **spec + prototype 생성**을 독립적으로 담당 — GSD/PDCA 위에 추가되는 레이어
- PLAN Stage에서 GSD plan-phase/PDCA plan을 호출하지 않음 — dev-lifecycle이 직접 spec.md + prototype.html 생성

### Claude's Discretion
- Preflight check의 구체적 출력 포맷
- spec 생성 시 사용자와의 대화 방식 (질문 → 생성 vs 초안 → 수정)
- prototype 생성에 소요되는 시간/복잡도 관리
- .lifecycle/ 디렉토리 구조 초기화 시점

</decisions>

<canonical_refs>
## Canonical References

**Downstream agents MUST read these before planning or implementing.**

### Spec + Prototype 포맷 (Phase 2-3 산출물)
- `skill/templates/spec-template.md` — E2E Spec 템플릿 (5단계 체인)
- `skill/templates/prototype-template.html` — Prototype 보일러플레이트 (4섹션)
- `skill/references/e2e-spec.md` — Spec 포맷 참조, ID 체계, cross-artifact linking
- `skill/references/prototype.md` — Prototype 포맷 참조, data-* 속성, manifest 스키마
- `skill/examples/user-login.spec.md` — Spec 예제
- `skill/examples/user-login.prototype.html` — Prototype 예제

### Stage 인프라 (Phase 1 산출물)
- `skill/references/stage-transitions.md` — Stage 전환 조건, gate rules
- `skill/references/role-matrix.md` — 9 Stage × 6 스킬 역할 매트릭스
- `skill/templates/state.json` — 상태 추적 스키마
- `skill/templates/manifest.json` — artifact 등록 구조

### 프로젝트 정의
- `.planning/PROJECT.md` — 핵심 문제: 100% 기능 실패, User Interaction Prototype이 해결책
- `.planning/REQUIREMENTS.md` — Phase 4 요구사항: SPEC-02, PROTO-05, PIPE-01

</canonical_refs>

<code_context>
## Existing Code Insights

### Reusable Assets
- `skill/templates/spec-template.md` — PLAN이 이 템플릿을 채워서 spec.md 생성
- `skill/templates/prototype-template.html` — PLAN이 이 보일러플레이트를 기반으로 prototype.html 생성
- `~/.claude/skills/dev-lifecycle/SKILL.md` — 현재 범용 스킬 (252줄), PLAN Stage 섹션에 이 어댑터가 연결됨

### Established Patterns
- Phase 2-3: template/reference/example 3파일 패턴
- ADR 스킬: WHY+SEE 코드 주석 패턴 (SPEC 주석도 동일)
- GSD: preflight 개념 (ROADMAP.md 기반 검증)

### Integration Points
- SKILL.md의 Stage 1 PLAN 섹션 — 이 어댑터가 호출되는 진입점
- `.lifecycle/features/{name}/` — spec.md + prototype.html 저장 위치
- `.lifecycle/state.json` — 현재 Stage를 PLAN으로 기록
- `.lifecycle/manifest.json` — spec.md, prototype.html을 1_plan.outputs로 등록

</code_context>

<specifics>
## Specific Ideas

- 사용자 강조: "너는 완성이라 하지만 실제로 100% 안 된다" — 합의 게이트가 이 문제의 핵심 해결책
- prototype을 브라우저에서 직접 열어 확인 — IDE 내부가 아닌 실제 브라우저 체험

</specifics>

<deferred>
## Deferred Ideas

None — discussion stayed within phase scope

</deferred>

---

*Phase: 04-plan-stage*
*Context gathered: 2026-03-22*
