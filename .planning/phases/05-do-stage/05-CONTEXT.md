# Phase 5: DO Stage - Context

**Gathered:** 2026-03-22
**Status:** Ready for planning

<domain>
## Phase Boundary

DO Stage 어댑터를 구현한다. 구현 중 승인된 E2E Spec의 각 단계를 체크리스트로 추적하고, spec에서 이탈 시 deviation log를 기록하며, ADR이 필요한 의사결정을 감지하여 제안한다. DO Stage는 승인된 spec 없이 시작할 수 없다.

</domain>

<decisions>
## Implementation Decisions

### Spec 체크리스트 추적
- 이미 결정됨 (Phase 2): DO에서 각 단계별 체크리스트 (Screen ✅ Connection ✅ Processing ⬜ ...)
- 코드에 `// SPEC: e2e-{feature}-{NNN}` 주석 삽입 (Phase 2)

### Deviation Log
- spec.md 내부에 기록 — 기능 문서와 함께, 한 곳에서 전체 이력 확인 (리서치 권장과 일치)
- Deviation 발생 시 기록 + 사용자 확인 — 이탈 사유를 알리고 사용자 승인 후 계속 진행
- 사용자가 승인하지 않으면 원래 spec대로 구현

### ADR 자동 감지
- 두 가지 트리거 모두 적용:
  1. Deviation 발생 시 — spec에서 이탈 = 의사결정 → ADR 제안
  2. 트레이드오프 감지 시 — "A 대신 B로" 같은 대화에서 트레이드오프 포착 → ADR 제안
- ADR 생성 시 코드에 WHY+SEE 주석 삽입 (기존 ADR 스킬 패턴)

### Claude's Discretion
- Deviation log의 구체적 마크다운 형식 (spec.md 내 위치, 필드)
- 체크리스트 추적의 시각적 표현 방식
- ADR 감지의 구체적 키워드/패턴
- DO Stage 참조 파일(do-stage.md)의 구조

</decisions>

<canonical_refs>
## Canonical References

**Downstream agents MUST read these before planning or implementing.**

### Stage 어댑터 패턴 (Phase 4 산출물)
- `skill/references/plan-stage.md` — PLAN Stage 어댑터 (DO Stage가 동일 패턴 따름)

### Spec + Prototype 포맷
- `skill/references/e2e-spec.md` — Spec 포맷 참조, deviation log 언급
- `skill/templates/spec-template.md` — Spec 템플릿 (deviation log 섹션 포함 여부 확인)

### 스킬 인프라
- `skill/SKILL.md` — Stage 2 DO 섹션
- `skill/references/role-matrix.md` — Stage 2 역할 매핑
- `skill/references/stage-transitions.md` — Stage 1→2 전환 조건 (approved spec 필수)

### 프로젝트 정의
- `.planning/REQUIREMENTS.md` — Phase 5 요구사항: SPEC-03, SPEC-05, PIPE-02

### ADR 스킬
- `~/.claude/skills/adr/SKILL.md` — WHY+SEE 코드 주석 패턴, ADR 생성 워크플로

</canonical_refs>

<code_context>
## Existing Code Insights

### Reusable Assets
- `skill/references/plan-stage.md` — Stage 어댑터 참조 파일 패턴 (311줄, 동일 구조로 do-stage.md 작성)
- `~/.claude/skills/adr/SKILL.md` — ADR 생성 6단계 워크플로, WHY+SEE 주석 형식

### Established Patterns
- plan-stage.md의 Step 구조 (Step 0 초기화, Step 1-4 파이프라인, anti-patterns)
- spec-template.md의 Deviation Log 섹션 (Phase 2에서 정의)
- `// SPEC:` 주석과 `// WHY:` + `// SEE:` 주석의 공존 패턴

### Integration Points
- `.lifecycle/features/{name}/spec.md` — deviation log 추가 위치
- `.lifecycle/state.json` — Stage를 DO로 업데이트
- `.lifecycle/manifest.json` — DO Stage outputs 등록
- stage-transitions.md의 `1_to_2` gate rule — approved spec 확인

</code_context>

<specifics>
## Specific Ideas

- Deviation 발생 시 사용자에게 즉시 알림 — "spec과 다르게 구현하려 합니다. 이유: [X]. 계속할까요?"
- 사용자가 거부하면 원래 spec대로 돌아감

</specifics>

<deferred>
## Deferred Ideas

None — discussion stayed within phase scope

</deferred>

---

*Phase: 05-do-stage*
*Context gathered: 2026-03-22*
