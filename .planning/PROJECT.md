# Dev Lifecycle Orchestrator

## What This Is

Claude Code 스킬 형태의 개발 라이프사이클 오케스트레이터. 9단계 파이프라인(PLAN → DO → TEST → COMMIT → DEPLOY → DEPLOY TEST → DOCUMENT → RETROSPECT → PROMOTE)을 통해 GSD, PDCA, 커스텀 스킬(ADR, retrospective, work-log, canvas-design)을 하나의 워크플로로 통합한다. 모든 프로젝트 타입(웹, 모바일, API, 데스크톱 등)에서 범용적으로 사용할 수 있는 스킬.

## Core Value

**개발 과정에서 만들어지는 산출물이 AI의 외부 메모리 역할을 하여, 재작업과 맥락 유실을 근본적으로 방지한다.**

## Requirements

### Validated (v1.0)

- ✓ 범용 스킬 scaffold + state 관리 + artifact registry — Phase 1
- ✓ E2E Spec 5-layer chain 포맷 — Phase 2
- ✓ 클릭 가능한 HTML+JS 프로토타입 — Phase 3
- ✓ PLAN Stage (스펙+프로토타입 생성, 합의 게이트) — Phase 4
- ✓ DO Stage (스펙 기반 구현 추적, deviation 로깅, ADR 감지) — Phase 5
- ✓ TEST Stage (스펙 검증, 프로토타입 구조 비교, dual gate) — Phase 6
- ✓ COMMIT Stage (검증 게이트, why-centric 커밋) — Phase 7
- ✓ Memory & Decision Trail (settings changelog, decision log, Living State) — Phase 8
- ✓ Outer Loop (Deploy → Deploy Test → Document → Retrospect → Promote) — Phase 9
- ✓ Architecture & Resilience (progressive disclosure, session hooks, reconcile) — Phase 10

### Validated (v2.0 — gstack 패턴 차용, completed 2026-03-27)

- [x] **lifecycle-config** — `get/set` CLI 스타일 설정 관리 (mode, skip_stages, proactive, auto_skip) — Phase 11
- [x] **Stage-internal iteration** — DO 안에서 step별 mini-verify 루프 (QA→Fix→Verify 패턴) — Phase 12
- [x] **lifecycle-upgrade** — 스키마 마이그레이션 + 롤백 + 변경사항 요약 — Phase 13
- [x] **Completeness 점수** — 각 스테이지 N/10 점수, 의사결정 시 Completeness 비교 — Phase 12
- [x] **Proactive suggestions + opt-out** — 스테이지 전환 시 관련 스킬 추천, proactive: false로 끄기 — Phase 15
- [x] **Session tracking** — 세션별 context 파일 (.lifecycle/sessions/) — Phase 14
- [x] **Migration markers** — .lifecycle/.v2-migrated 등 안전한 업그레이드 표시 — Phase 13
- [x] **Before/after snapshot diff** — 스펙 baseline 대비 구현 delta report — Phase 14
- [x] **Analytics** — 스테이지별 병목 추적, 재작업 이벤트, time-per-stage — Phase 14
- [x] **gstack 스킬 연동** — review, qa, investigate, careful 등 적절한 시점에 추천 — Phase 15

### Out of Scope

- 특정 프로젝트(muse) 전용 기능 — 범용 스킬이 목적
- GSD/PDCA 본체 수정 — 기존 시스템 위에서 오케스트레이션만 담당
- Stage 9 PROMOTE 자동화 — 데모 영상 생성은 선택적 기능으로 유지

## Context

### 현재 상태
- `~/.claude/skills/dev-lifecycle/SKILL.md`에 스킬 정의가 존재하지만, muse 프로젝트에 하드코딩되어 있음
- GSD는 plan → execute → verify를 담당 (retrospective, ADR은 GSD 본체에 없음)
- PDCA(bkit)는 plan → design → do → check → act → report를 담당
- ADR, gsd-retrospective, work-log, canvas-design은 모두 사용자가 만든 커스텀 스킬

### 핵심 문제 3가지
1. **기능 완성도 100% 실패** — Claude가 "완성"이라고 해도 실제로 버튼이 없거나, 버튼을 눌러도 동작 안 하거나, API는 있는데 호출 안 하거나, end-to-end로 작동하지 않는 경우가 100%
2. **메모리 유실** — 최초 의도 → 개발 중 방향 변경 → 세팅 변경 → 최종 상태의 히스토리가 사라져서, 나중에 세팅을 바꾸려면 처음부터 다시 설명해야 함
3. **역할 중복** — GSD plan-phase vs dev-lifecycle Stage 1 vs PDCA plan이 겹치면서 어디서 뭘 해야 하는지 불명확

### 해결 방향
- User Interaction Prototype으로 기능 완성도 문제 해결 (PLAN에서 합의 → DO에서 구현 → TEST에서 검증)
- Stage 산출물 자체가 구조화된 메모리 역할 → 별도 "기억해"가 필요 없는 구조
- 먼저 dev-lifecycle을 제대로 만들고 → 실제 프로젝트에 적용 → 메모리 문제 해결 여부 역검토

## Constraints

- **스킬 형태**: Claude Code 스킬(SKILL.md)로 배포 — 별도 서버나 인프라 없음
- **기존 시스템 의존**: GSD, PDCA, ADR 등 기존 스킬은 수정하지 않고 위에서 오케스트레이션
- **범용성**: 특정 프레임워크/언어/플랫폼에 종속되지 않아야 함

## Key Decisions

| Decision | Rationale | Outcome |
|----------|-----------|---------|
| 먼저 만들고 역검토 | 메모리 해결책을 먼저 설계하면 끼워맞추기가 됨 — 실사용 후 검증이 올바른 순서 | ✓ Good |
| User Interaction Prototype은 클릭 가능한 HTML+JS | 정적 와이어프레임으로는 플로우 검증 불가, 실제 클릭해서 확인해야 누락 방지 | ✓ Good |
| end-to-end 기능 명세 도입 | "함수 존재" ≠ "기능 작동" — 화면→연결→처리→응답→에러 전체 체인 명세 필요 | ✓ Good |
| gstack 패턴 전면 차용 (v2.0) | symlink 설치, config CLI, stage-internal iteration, upgrade, analytics 등 — 성숙한 스킬 시스템의 검증된 패턴 | — Pending |

## Current Milestone: v2.0 gstack Pattern Adoption

**Goal:** gstack에서 검증된 패턴들을 dev-lifecycle에 전면 차용하여 설정 관리, 내부 반복 루프, 업그레이드, 분석, 스킬 연동을 강화한다.

**Target features:**
- lifecycle-config (파일 기반 설정 관리)
- Stage-internal iteration (QA→Fix→Verify 루프)
- lifecycle-upgrade (스키마 마이그레이션)
- Completeness 점수 + Proactive suggestions
- Session tracking + Migration markers
- Before/after snapshot diff
- Analytics (병목/재작업 추적)
- gstack 스킬 연동

---
*Last updated: 2026-03-24 after v2.0 milestone start*
