# Dev Lifecycle Orchestrator

## What This Is

Claude Code 스킬 형태의 개발 라이프사이클 오케스트레이터. 9단계 파이프라인(PLAN → DO → TEST → COMMIT → DEPLOY → DEPLOY TEST → DOCUMENT → RETROSPECT → PROMOTE)을 통해 GSD, PDCA, 커스텀 스킬(ADR, retrospective, work-log, canvas-design)을 하나의 워크플로로 통합한다. 모든 프로젝트 타입(웹, 모바일, API, 데스크톱 등)에서 범용적으로 사용할 수 있는 스킬.

## Core Value

**개발 과정에서 만들어지는 산출물이 AI의 외부 메모리 역할을 하여, 재작업과 맥락 유실을 근본적으로 방지한다.**

## Requirements

### Validated

(None yet — ship to validate)

### Active

- [ ] 현재 muse 프로젝트 하드코딩 제거 — 범용 스킬로 전환
- [ ] GSD/PDCA/커스텀 스킬 간 역할 경계 명확화 — 겹침과 빈틈 해소
- [ ] User Interaction Prototype — 기능별 클릭 가능한 HTML+JS 목업을 PLAN 단계에서 생성하여 사용자와 합의
- [ ] End-to-end 기능 명세 — 화면 → 연결 → 처리 → 응답 → 에러의 전체 플로우를 명세하고 검증 기준으로 사용
- [ ] 세팅/설정 변경 자동 기록 — 변경 시점에 맥락(왜)+값(무엇)을 자동으로 영구 기록하는 메커니즘
- [ ] 의사결정 히스토리 추적 — ADR보다 가벼운 수준의 설정 변경/방향 전환 이력 누적
- [ ] 현재 최종 상태 문서 — 세션 시작 시 읽으면 프로젝트 전체 맥락을 즉시 복원할 수 있는 살아있는 문서
- [ ] Stage 간 산출물 연결 — 각 Stage의 출력이 다음 Stage의 입력으로 자동 전달되는 구조
- [ ] 코드 주석에 ADR 연결 — 의사결정 변경 시 코드의 WHY+SEE 주석으로 추후 F/U 가능

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
| 먼저 만들고 역검토 | 메모리 해결책을 먼저 설계하면 끼워맞추기가 됨 — 실사용 후 검증이 올바른 순서 | — Pending |
| User Interaction Prototype은 클릭 가능한 HTML+JS | 정적 와이어프레임으로는 플로우 검증 불가, 실제 클릭해서 확인해야 누락 방지 | — Pending |
| end-to-end 기능 명세 도입 | "함수 존재" ≠ "기능 작동" — 화면→연결→처리→응답→에러 전체 체인 명세 필요 | — Pending |

---
*Last updated: 2026-03-22 after initialization*
