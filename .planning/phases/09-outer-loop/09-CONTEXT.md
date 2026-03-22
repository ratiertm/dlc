# Phase 9: Outer Loop - Context

**Gathered:** 2026-03-22
**Status:** Ready for planning

<domain>
## Phase Boundary

Outer Loop Stage 5-9 어댑터를 구현한다: DEPLOY(범용 배포 템플릿), DEPLOY TEST(스모크 테스트), DOCUMENT(아키텍처/시퀀스 다이어그램, CLAUDE.md/README.md), RETROSPECT(회고 + ADR gap + work-log + Living State 반영), PROMOTE(선택적 데모 생성). Inner Loop(Phase 4-7)에서 확립된 Step 0-N 어댑터 패턴을 따른다.

</domain>

<decisions>
## Implementation Decisions

### 전체 방향
- Inner Loop 어댑터 패턴(Step 0-N + anti-patterns + file locations) 동일하게 적용
- 5개 Stage를 각각 별도 참조 파일로 생성 (deploy-stage.md, deploy-test-stage.md, document-stage.md, retrospect-stage.md, promote-stage.md)
- 모든 프로젝트 타입 동등 지원 (Phase 1 결정)

### Stage 5 DEPLOY
- 프로젝트 타입별 범용 배포 템플릿 (웹/모바일/API/데스크톱/CLI)
- 자동 감지 + 사용자 확인 (Phase 1 결정)

### Stage 6 DEPLOY TEST
- 스모크 테스트: 서버 상태, 핵심 플로우, 리소스 체크

### Stage 7 DOCUMENT
- 아키텍처 다이어그램, 시퀀스 다이어그램 (Mermaid 또는 canvas-design)
- CLAUDE.md / README.md 업데이트

### Stage 8 RETROSPECT
- gsd-retrospective 스킬 호출
- ADR 누락 체크
- work-log 기록
- Living State에 교훈 반영 (Phase 8에서 구현된 메커니즘 활용)

### Stage 9 PROMOTE
- 선택적 — 데모 영상 생성 파이프라인 (Maestro + ADB + FFmpeg)
- 스킵 가능, 패널티 없음

### Claude's Discretion
- 각 Stage 참조 파일의 구체적 Step 구조
- 배포 템플릿의 프로젝트 타입별 세부 내용
- SKILL.md Stage 5-9 섹션 업데이트 방식
- role-matrix Stage 5-9 업데이트

</decisions>

<canonical_refs>
## Canonical References

**Downstream agents MUST read these before planning or implementing.**

### Inner Loop 어댑터 패턴
- `skill/references/plan-stage.md` — Stage 어댑터 원본 패턴 (311줄)
- `skill/references/do-stage.md`, `test-stage.md`, `commit-stage.md` — 동일 패턴 참조

### 스킬 인프라
- `skill/SKILL.md` — Stage 5-9 섹션 (현재 간략, 업데이트 필요)
- `skill/references/role-matrix.md` — Stage 5-9 역할 매핑 (현재 기존 스킬 primary)
- `skill/references/stage-transitions.md` — Stage 4→5→...→9 전환 조건
- `skill/references/project-detection.md` — 프로젝트 타입 감지 (DEPLOY에서 사용)

### 커스텀 스킬 연동
- `~/.claude/skills/gsd-retrospective/SKILL.md` — 회고 스킬 (RETROSPECT에서 호출)
- `~/.claude/skills/adr/SKILL.md` — ADR 누락 체크 (RETROSPECT에서 사용)
- `~/.claude/skills/work-log/SKILL.md` — 작업 기록 (RETROSPECT에서 호출)

### 프로젝트 정의
- `.planning/REQUIREMENTS.md` — Phase 9 요구사항: PIPE-05~09

</canonical_refs>

<code_context>
## Existing Code Insights

### Reusable Assets
- 4개 Inner Loop 어댑터 (plan/do/test/commit-stage.md) — 동일 구조
- `skill/references/project-detection.md` — 9+ 프로젝트 타입 감지 로직
- `skill/templates/manifest.json` — Stage 5-9 gate rules 이미 정의

### Established Patterns
- Step 0-N + anti-patterns + file locations 구조
- SKILL.md Read directive 패턴
- role-matrix 3-table 업데이트 패턴

### Integration Points
- `.lifecycle/state.json` — Stage 5-9 상태 추적
- `.lifecycle/manifest.json` — Outer Loop artifact 등록
- Living State — RETROSPECT에서 교훈 반영

</code_context>

<specifics>
## Specific Ideas

No specific requirements — follow Inner Loop adapter pattern for all 5 stages

</specifics>

<deferred>
## Deferred Ideas

None — discussion stayed within phase scope

</deferred>

---

*Phase: 09-outer-loop*
*Context gathered: 2026-03-22*
