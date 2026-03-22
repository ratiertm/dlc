# Phase 8: Memory & Decision Trail - Context

**Gathered:** 2026-03-22
**Status:** Ready for planning

<domain>
## Phase Boundary

메모리 문제 해결을 위한 4가지 메커니즘을 구현한다: (1) 모든 설정 변경의 자동 기록, (2) ADR보다 가벼운 경량 결정 이력, (3) 세션 시작 시 즉시 맥락 복원하는 Living State Document, (4) ADR과 코드의 WHY+SEE 주석 연결. 이 Phase는 프로젝트의 핵심 목표인 "AI 메모리 한계 해결"에 직접 대응한다.

</domain>

<decisions>
## Implementation Decisions

### Living State Document
- 포함 정보: 현재 상태 + 최근 결정 + 활성 설정값 목록 + 프로젝트 시작부터 현재까지 주요 이벤트 요약
- 업데이트 시점: 매 Stage 전환 시 + 매 세션 종료 시 (둘 다)
- 목적: 다음 세션에서 이 파일 하나만 읽으면 전체 맥락 즉시 복원

### 설정 변경 기록
- 범위: 모든 설정 변경 자동 기록 (.env, config, 프레임워크 설정 등)
- 각 기록에 what(무엇이 바뀌었는지) + why(왜 바꿨는지) 포함
- ADR보다 가벼움 — 한 줄이면 충분

### 경량 결정 이력 (Lightweight Decision Log)
- 이미 결정됨 (PROJECT.md): ADR보다 가벼운 한 줄짜리 결정 이력 누적
- 방향 전환, 세팅 변경, 작은 트레이드오프 등 ADR까지는 필요 없는 결정

### WHY+SEE 코드 주석
- 이미 결정됨 (Phase 2, 5): ADR 생성 시 관련 코드에 WHY+SEE 주석 삽입
- `// SPEC:` 주석과 공존하는 패턴 (Phase 5에서 정의)

### Claude's Discretion
- Living State Document의 구체적 마크다운 구조
- settings-changelog의 저장 위치 (.lifecycle/ 내부)
- 경량 결정 이력의 파일 위치와 포맷
- SessionStart hook의 구현 방식

</decisions>

<canonical_refs>
## Canonical References

**Downstream agents MUST read these before planning or implementing.**

### 프로젝트 정의
- `.planning/PROJECT.md` — 핵심 목표: AI 메모리 한계 해결, 세팅 변경 맥락 유실 문제
- `.planning/REQUIREMENTS.md` — Phase 8 요구사항: MEMO-01, MEMO-02, MEMO-03, MEMO-04

### 기존 스킬 인프라
- `skill/SKILL.md` — 현재 범용 스킬 (Session Start 섹션 참조)
- `skill/templates/state.json` — 상태 추적 스키마 (Living State와 연동)
- `skill/references/stage-transitions.md` — Stage 전환 시 Living State 업데이트 트리거

### ADR 패턴
- `~/.claude/skills/adr/SKILL.md` — WHY+SEE 코드 주석 패턴
- `skill/references/do-stage.md` — ADR 자동 감지 + WHY+SEE 삽입 로직

### 이전 Phase 결정
- `.planning/phases/05-do-stage/05-CONTEXT.md` — deviation log + ADR 트리거 결정
- `.planning/phases/01-foundation/01-CONTEXT.md` — .lifecycle/ 네임스페이스, state.json 구조

</canonical_refs>

<code_context>
## Existing Code Insights

### Reusable Assets
- `skill/templates/state.json` — session.last_active, resume_hint 필드 (Living State 연동 가능)
- `~/.claude/skills/adr/SKILL.md` — WHY+SEE 주석 패턴 (Step 4)
- `skill/references/do-stage.md` — ADR 감지 + 코드 주석 삽입 로직 (Step 4)

### Established Patterns
- `.lifecycle/` 네임스페이스 — 모든 프로젝트별 상태 파일 위치
- Stage 어댑터 패턴 (plan-stage, do-stage, test-stage, commit-stage) — 동일 구조 참조 가능
- Claude memory 파일 (`~/.claude/projects/*/memory/`) — 기존 메모리 시스템과의 관계

### Integration Points
- `.lifecycle/state.json` — Living State와 동기화
- `.lifecycle/manifest.json` — settings changelog를 artifact로 등록
- SessionStart hook — Living State 자동 로드 (ARCH-03, Phase 10에서 구현)

</code_context>

<specifics>
## Specific Ideas

- 사용자 원래 문제: "최초 의도 → 개발 중 방향 변경 → 세팅 변경 → 최종 상태의 히스토리가 사라져서, 나중에 세팅 바꾸려면 처음부터 다시 설명해야 함"
- Living State가 이 문제의 해결책 — 한 파일로 전체 맥락 복원

</specifics>

<deferred>
## Deferred Ideas

- SessionStart hook 구현 — Phase 10 (ARCH-03)에서 처리
- Compaction 자동 복구 — v2 (AUTO-03)

</deferred>

---

*Phase: 08-memory-decision-trail*
*Context gathered: 2026-03-22*
