# Phase 3: Prototype Format - Context

**Gathered:** 2026-03-22
**Status:** Ready for planning

<domain>
## Phase Boundary

클릭 가능한 단일 HTML 파일 프로토타입 템플릿을 만든다. hash 기반 SPA 라우팅, data-* 속성, 내장 JSON manifest를 포함하여 사용자가 브라우저에서 기능 플로우를 직접 확인할 수 있게 한다. Phase 2의 E2E Spec과 연결되어 spec의 각 단계가 prototype의 UI 요소로 매핑된다.

</domain>

<decisions>
## Implementation Decisions

### 시각적 수준
- 미니맘 UI — 기본 CSS 스타일링으로 깔끔하게, 와이어프레임보다는 나지만 실제 앱 디자인은 아님
- 반응형 디자인 — 모바일/데스크톱 둘 다 확인 가능

### 인터랙션 시뮬레이션
- 버튼 클릭 시 화면 전환 + mock 데이터 표시 — 단순 alert가 아닌 실제 플로우 체험
- 에러 시뮬레이션 — Claude's Discretion (에러 UI를 포함할지 여부)

### Spec 연결 방식
- 이중 연결: data-spec-id 속성 + JSON manifest 양쪽에서 참조
- 각 UI 요소에 `data-spec-id="e2e-{feature}-{NNN}"` 속성
- 내장 JSON manifest에 `{specId, element, type}` 매핑

### Claude's Discretion
- 에러 상태 시뮬레이션 포함 여부
- CSS 프레임워크 사용 여부 (vanilla vs 경량 라이브러리)
- prototype 보일러플레이트의 구체적 HTML 구조
- mock 데이터 형식과 주입 방식

</decisions>

<canonical_refs>
## Canonical References

**Downstream agents MUST read these before planning or implementing.**

### E2E Spec (Phase 2 산출물)
- `skill/templates/spec-template.md` — 5단계 체인 템플릿, spec ID 형식
- `skill/references/e2e-spec.md` — 포맷 참조, cross-artifact linking (data-spec-id 규칙)
- `skill/examples/user-login.spec.md` — 실제 예제 (prototype이 이 spec을 기반으로 생성될 수 있음)

### 프로젝트 정의
- `.planning/PROJECT.md` — User Interaction Prototype이 핵심 해결책
- `.planning/REQUIREMENTS.md` — Phase 3 요구사항: PROTO-01, PROTO-02, PROTO-03, PROTO-04
- `.planning/research/STACK.md` — 단일 HTML, hash 라우팅, data-* 속성, embedded manifest 기술 결정

### 이전 Phase
- `.planning/phases/02-e2e-spec-format/02-CONTEXT.md` — spec 포맷, ID 체계, .lifecycle/features/ 위치

</canonical_refs>

<code_context>
## Existing Code Insights

### Reusable Assets
- `skill/templates/spec-template.md` — spec ID 포맷 (`e2e-{feature}-{NNN}`) 참조
- `skill/references/e2e-spec.md` — Cross-Artifact Linking 테이블 (data-spec-id 속성 규칙 정의됨)
- `skill/examples/user-login.spec.md` — prototype 생성 시 참고할 실제 spec 예제

### Established Patterns
- Phase 1: references/ 기능별 분리, templates/ 산출물 템플릿
- Phase 2: .spec.md Markdown 포맷 + YAML frontmatter
- skill/ 디렉토리와 ~/.claude/skills/ 런타임 디렉토리 이중 배포

### Integration Points
- `.lifecycle/features/{name}/prototype.html` — spec.md와 같은 디렉토리에 위치
- manifest.json `1_plan.outputs` — prototype.html이 PLAN Stage 산출물로 등록
- TEST 단계에서 prototype manifest vs 실제 구현 구조 비교 (Phase 6에서 구현)

</code_context>

<specifics>
## Specific Ideas

- 사용자가 브라우저에서 직접 열어 클릭하며 플로우를 확인하는 것이 핵심 — 개발자 도구 없이도 동작해야 함
- file:// 프로토콜로 열어도 작동해야 함 (서버 불필요)

</specifics>

<deferred>
## Deferred Ideas

None — discussion stayed within phase scope

</deferred>

---

*Phase: 03-prototype-format*
*Context gathered: 2026-03-22*
