# Phase 7: COMMIT Stage - Context

**Gathered:** 2026-03-22
**Status:** Ready for planning

<domain>
## Phase Boundary

COMMIT Stage 어댑터를 구현한다. TEST Stage의 검증 결과를 확인하여 FAIL이 있으면 COMMIT을 차단하고 gap list를 표시한다. 모든 검증이 PASS면 why 중심 커밋 메시지를 생성하고, 관련 ADR을 참조하며, 필수 artifact 존재를 최종 확인한다.

</domain>

<decisions>
## Implementation Decisions

### 검증 게이트
- TEST Stage의 verification 결과 확인 — FAIL 항목이 있으면 COMMIT 차단
- 차단 시 구체적 gap list 표시 (어떤 spec step이 FAIL인지, 왜인지)
- 사용자가 방향 결정 (DO로 돌아가기 / 무시하고 진행 등)

### 커밋 메시지
- "why" 중심 — 변경의 이유를 중심으로 작성
- 관련 ADR이 있으면 커밋 메시지에 `(ADR-NNN)` 참조 포함
- 관련 spec ID도 참조 가능

### Artifact 검증
- manifest.json의 gate rules에 따라 필수 artifact 존재 확인
- spec.md, prototype.html, verification report가 모두 있어야 COMMIT 진입

### Claude's Discretion
- commit-stage.md 참조 파일의 구체적 구조
- 커밋 메시지 포맷의 세부사항
- gap list 표시 형식
- SKILL.md/role-matrix Stage 4 업데이트 방식

</decisions>

<canonical_refs>
## Canonical References

**Downstream agents MUST read these before planning or implementing.**

### Stage 어댑터 패턴
- `skill/references/test-stage.md` — TEST Stage (직전 Stage, dual gate 산출물이 COMMIT 입력)
- `skill/references/plan-stage.md` — PLAN Stage (패턴 참조)

### 스킬 인프라
- `skill/SKILL.md` — Stage 4 COMMIT 섹션
- `skill/references/role-matrix.md` — Stage 4 역할 매핑
- `skill/references/stage-transitions.md` — Stage 3→4 전환 조건
- `skill/templates/manifest.json` — gate rules (3_to_4)

### 프로젝트 정의
- `.planning/REQUIREMENTS.md` — Phase 7 요구사항: PIPE-04, ARCH-04

</canonical_refs>

<code_context>
## Existing Code Insights

### Reusable Assets
- `skill/references/plan-stage.md`, `do-stage.md`, `test-stage.md` — 동일 Step 0-N 패턴
- `skill/templates/manifest.json` — gate_rules `3_to_4` 조건 이미 정의됨

### Established Patterns
- Stage 어댑터: Step 0 초기화 → Step 1-N 파이프라인 → anti-patterns → file locations
- Stage 1-3 모두 dev-lifecycle primary로 업데이트됨

### Integration Points
- `.lifecycle/features/{name}/verification.json` — TEST 결과 읽기
- `.lifecycle/state.json` — Stage를 COMMIT으로 업데이트
- `.lifecycle/manifest.json` — 4_commit outputs 등록

</code_context>

<specifics>
## Specific Ideas

No specific requirements — open to standard approaches

</specifics>

<deferred>
## Deferred Ideas

None — discussion stayed within phase scope

</deferred>

---

*Phase: 07-commit-stage*
*Context gathered: 2026-03-22*
