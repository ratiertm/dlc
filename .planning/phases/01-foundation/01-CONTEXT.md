# Phase 1: Foundation - Context

**Gathered:** 2026-03-22
**Status:** Ready for planning

<domain>
## Phase Boundary

범용 스킬 골격을 구축한다. muse 하드코딩을 제거하고, state.json으로 상태를 추적하며, artifact manifest로 Stage 간 산출물을 연결하고, GSD/PDCA/커스텀 스킬 간 역할 매트릭스를 정의한다. 이후 모든 Phase의 기반이 되는 인프라 작업.

</domain>

<decisions>
## Implementation Decisions

### 스킬 배포 구조
- 글로벌(`~/.claude/skills/dev-lifecycle/`) + 프로젝트 로컬(`.claude/skills/`) 오버라이드 구조
- SKILL.md는 500줄 이하, 상세 로직은 `references/` 기능별 파일로 분리 (preflight.md, e2e-spec.md, prototype.md 등)
- `templates/` 폴더에 전체 Stage 산출물 템플릿 포함 (spec, prototype, verification report, deploy checklist 등)

### 상태 저장 방식
- `.lifecycle/` 별도 네임스페이스 사용 — GSD의 `.planning/`과 분리
- `state.json` — 현재 Stage, 진행 상태, 활성 기능 추적
- `manifest.json` — artifact 추적을 state.json과 별도 파일로 관리
- GSD STATE.md와의 관계 — Claude's Discretion (읽기 전용 or 독립 운영 중 최적 방식 판단)

### 역할 경계 전략
- dev-lifecycle이 Stage 전환 시 자동으로 적절한 스킬 감지 및 호출
- 사용자는 dev-lifecycle, GSD, PDCA를 병렬로 사용 가능 — 단일 진입점 강제하지 않음
- GSD plan과 PDCA plan 겹칠 때 dev-lifecycle이 상황에 따라 조율하여 호출

### 범용화
- 모든 프로젝트 타입(웹/모바일/API/데스크톱/CLI) 동등 지원
- 프로젝트 타입 감지: 자동 감지(package.json, pubspec.yaml 등) 후 사용자 확인
- muse 전용 하드코딩(rsync 경로, flutter build apk, PIN 인증, God Agent 등) 전체 제거

### Claude's Discretion
- GSD STATE.md와의 구체적 동기화 방식
- references/ 파일 목록과 각 파일의 책임 범위
- state.json 스키마의 구체적 필드 설계

</decisions>

<canonical_refs>
## Canonical References

**Downstream agents MUST read these before planning or implementing.**

### 현재 스킬
- `~/.claude/skills/dev-lifecycle/SKILL.md` — 현재 muse 하드코딩된 스킬 (제거 대상 참조)

### 프로젝트 정의
- `.planning/PROJECT.md` — 프로젝트 목표, 핵심 문제 3가지, 핵심 아이디어
- `.planning/REQUIREMENTS.md` — Phase 1 요구사항: FOUND-01, FOUND-02, FOUND-03, FOUND-05

### 리서치
- `.planning/research/STACK.md` — 스킬 구조, progressive disclosure, hooks 시스템
- `.planning/research/ARCHITECTURE.md` — .lifecycle/ 네임스페이스, adapter pattern, manifest 구조

</canonical_refs>

<code_context>
## Existing Code Insights

### Reusable Assets
- 현재 SKILL.md (281줄) — 9-Stage 파이프라인 정의, Stage별 설명이 기반으로 사용 가능
- GSD의 gsd-tools.cjs — commit, state 관리 등 CLI 유틸리티 패턴 참고 가능

### Established Patterns
- GSD: `.planning/` + STATE.md + ROADMAP.md + config.json 패턴
- PDCA(bkit): `.pdca-status.json` + `docs/01-plan/` 구조
- ADR: `docs/decisions/NNN-slug.md` + WHY+SEE 코드 주석 패턴

### Integration Points
- GSD 스킬 호출: `/gsd:plan-phase`, `/gsd:verify-work`, `/gsd:complete-milestone`
- PDCA 스킬 호출: `/pdca plan`, `/pdca do`, `/pdca analyze`
- ADR 스킬 호출: `/adr` (트레이드오프 감지 시)
- 기타: `/gsd-retrospective`, `/work-log`, `/canvas-design`

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

*Phase: 01-foundation*
*Context gathered: 2026-03-22*
