# Phase 6: TEST Stage - Context

**Gathered:** 2026-03-22
**Status:** Ready for planning

<domain>
## Phase Boundary

TEST Stage 어댑터를 구현한다. E2E Spec의 각 단계별 pass/fail 검증, Prototype manifest vs 실제 구현의 구조적 대조, 사용자를 위한 행동 검증 체크리스트를 생성한다. Claude의 구조 검증과 사용자의 행동 검증이 모두 통과해야 COMMIT으로 넘어갈 수 있다.

</domain>

<decisions>
## Implementation Decisions

### Spec 단계별 검증
- 이미 결정됨 (Phase 2): 각 단계(Screen/Connection/Processing/Response/Error)별 개별 pass/fail 판정
- 검증 리포트를 `.lifecycle/features/{name}/verification.json` 또는 `.md`로 생성

### 검증 실패 시 처리
- FAIL 목록을 사용자에게 보고 — 사용자가 방향 결정 (자동 수정/DO로 돌려보내기/무시)
- Claude가 자동으로 수정하지 않음 — 사용자 판단에 맡김

### 사용자 행동 검증
- 단계별 체크리스트 형식으로 안내 ("1. 로그인 페이지 열기 2. 이메일 입력 3. 로그인 클릭 4. 대시보드 확인")
- E2E Spec의 각 단계를 사용자가 실제로 수행할 수 있는 구체적 행동으로 변환
- Claude 구조 검증 + 사용자 행동 검증 둘 다 통과해야 COMMIT 진입 가능

### Prototype 구조 대조
- 이미 결정됨 (Phase 3): data-spec-id 속성 + JSON manifest 이중 연결
- TEST에서 prototype manifest의 specMapping vs 실제 구현의 존재 여부 비교

### Claude's Discretion
- 검증 리포트의 구체적 포맷 (JSON vs Markdown)
- Prototype 구조 대조의 구체적 비교 방법
- 행동 검증 체크리스트의 세부 형식
- test-stage.md 참조 파일의 구조

</decisions>

<canonical_refs>
## Canonical References

**Downstream agents MUST read these before planning or implementing.**

### Stage 어댑터 패턴
- `skill/references/plan-stage.md` — PLAN Stage 어댑터 (패턴 참조)
- `skill/references/do-stage.md` — DO Stage 어댑터 (직전 Stage, 산출물이 TEST 입력)

### Spec + Prototype
- `skill/references/e2e-spec.md` — Spec 포맷, status lifecycle (implemented → verified)
- `skill/references/prototype.md` — Prototype 포맷, manifest 스키마, data-spec-id 규칙

### 스킬 인프라
- `skill/SKILL.md` — Stage 3 TEST 섹션
- `skill/references/role-matrix.md` — Stage 3 역할 매핑
- `skill/references/stage-transitions.md` — Stage 2→3 전환 조건

### 프로젝트 정의
- `.planning/REQUIREMENTS.md` — Phase 6 요구사항: SPEC-04, PROTO-06, PIPE-03

</canonical_refs>

<code_context>
## Existing Code Insights

### Reusable Assets
- `skill/references/plan-stage.md` (311줄), `do-stage.md` (307줄) — Stage 어댑터 동일 패턴
- `skill/references/e2e-spec.md` — spec status: `implemented` → `verified` 전환 정의
- `skill/references/prototype.md` — manifest의 `specMapping` 구조 (TEST가 이것을 파싱)

### Established Patterns
- Step 0-N 구조 + anti-patterns + file locations (plan-stage, do-stage 동일)
- Spec 체크리스트 추적: DO에서 `pending` → `implemented`, TEST에서 `implemented` → `verified/failed`

### Integration Points
- `.lifecycle/features/{name}/spec.md` — TEST가 읽어서 각 단계 검증
- `.lifecycle/features/{name}/prototype.html` — manifest JSON 파싱하여 구조 대조
- `.lifecycle/state.json` — Stage를 TEST로 업데이트
- stage-transitions.md `2_to_3` gate — DO 완료 확인

</code_context>

<specifics>
## Specific Ideas

- 사용자 핵심 문제: "완성이라 했는데 100% 안 된다" — TEST Stage가 이것의 최종 방어선
- Claude 구조 검증이 PASS여도 사용자 행동 검증이 FAIL이면 → COMMIT 불가

</specifics>

<deferred>
## Deferred Ideas

None — discussion stayed within phase scope

</deferred>

---

*Phase: 06-test-stage*
*Context gathered: 2026-03-22*
