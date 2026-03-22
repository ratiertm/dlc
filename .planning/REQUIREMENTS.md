# Requirements: Dev Lifecycle Orchestrator

**Defined:** 2026-03-22
**Core Value:** 개발 과정 산출물이 AI의 외부 메모리 역할을 하여, 재작업과 맥락 유실을 근본적으로 방지한다.

## v1 Requirements

### Foundation

- [ ] **FOUND-01**: 현재 muse 프로젝트 하드코딩을 제거하여 모든 프로젝트 타입에서 사용 가능한 범용 스킬로 전환
- [ ] **FOUND-02**: state.json으로 현재 Stage, 진행 상태, 활성 기능을 추적하여 세션 간 상태 유지
- [ ] **FOUND-03**: Stage Artifact Manifest로 각 Stage의 산출물을 기록하고, 다음 Stage에서 필수 입력으로 참조
- [ ] **FOUND-04**: 상황별 실행 모드(핫픽스/피처/릴리즈/마일스톤)를 제공하여 불필요한 Stage 생략 가능
- [ ] **FOUND-05**: GSD/PDCA/커스텀 스킬 간 역할 매트릭스를 정의하여 각 Stage에서 어떤 스킬이 어떤 역할을 하는지 명확화

### User Interaction & Verification

- [ ] **UIXV-01**: End-to-End Feature Spec 포맷 정의 — 각 기능을 화면→연결→처리→응답→에러 5단계로 명세
- [ ] **UIXV-02**: PLAN 단계에서 클릭 가능한 HTML+JS User Interaction Prototype을 생성하여 사용자와 UI/기능 플로우 합의
- [ ] **UIXV-03**: DO 단계에서 E2E Spec의 각 단계(화면/연결/처리/응답/에러)가 구현되었는지 체크리스트로 추적
- [ ] **UIXV-04**: TEST 단계에서 User Interaction Prototype과 실제 구현을 대조하여 누락된 화면/기능/연결 검출
- [ ] **UIXV-05**: Stage 전환 시 필수 artifact 존재를 검증하여, 미완성 상태에서 다음 Stage로 넘어가지 못하게 차단

### Memory & Decision Trail

- [ ] **MEMO-01**: 세팅/설정 변경 시 맥락(왜 바꿨는지)과 값(무엇을 바꿨는지)을 자동으로 영구 기록
- [ ] **MEMO-02**: ADR보다 가벼운 한 줄짜리 결정 이력(Lightweight Decision Log)을 누적하여 방향 변경 추적
- [ ] **MEMO-03**: 세션 시작 시 읽으면 프로젝트 전체 맥락(현재 상태, 최근 결정, 활성 설정)을 즉시 복원할 수 있는 Living State Document 유지
- [ ] **MEMO-04**: ADR 생성 시 관련 코드에 WHY+SEE 주석을 삽입하여, 나중에 해당 코드를 수정할 때 결정 맥락을 즉시 확인 가능

### Pipeline — Inner Loop (Stage 1-4)

- [ ] **PIPE-01**: Stage 1 PLAN 어댑터 — GSD/PDCA plan 호출, preflight check(이전 회고 교훈, 관련 ADR, 미해결 gap, 기술부채), User Interaction Prototype 생성
- [ ] **PIPE-02**: Stage 2 DO 어댑터 — PDCA do 호출, 트레이드오프 감지 시 ADR 자동 생성 제안, 코드에 WHY+SEE 주석 삽입
- [ ] **PIPE-03**: Stage 3 TEST 어댑터 — GSD verify + PDCA analyze 호출, E2E Spec 대비 실제 동작 검증, Prototype 대조
- [ ] **PIPE-04**: Stage 4 COMMIT 어댑터 — 커밋 메시지에 "why" 중심, 관련 ADR 참조, 버전 태그

### Pipeline — Outer Loop (Stage 5-9)

- [ ] **PIPE-05**: Stage 5 DEPLOY 어댑터 — 프로젝트 타입별 배포 템플릿(웹/모바일/API/데스크톱), 범용 체크리스트
- [ ] **PIPE-06**: Stage 6 DEPLOY TEST 어댑터 — 배포 후 스모크 테스트(서버 상태, 핵심 플로우, 리소스 체크)
- [ ] **PIPE-07**: Stage 7 DOCUMENT 어댑터 — 아키텍처 다이어그램, 시퀀스 다이어그램, CLAUDE.md/README.md 업데이트
- [ ] **PIPE-08**: Stage 8 RETROSPECT 어댑터 — gsd-retrospective 호출, ADR 누락 체크, work-log 기록, 교훈을 Living State에 반영
- [ ] **PIPE-09**: Stage 9 PROMOTE 어댑터 — 선택적 데모 영상 생성 파이프라인

### Skill Architecture

- [ ] **ARCH-01**: SKILL.md를 500줄 이하로 유지하고, 상세 로직은 references/ 파일로 분리 (progressive disclosure)
- [ ] **ARCH-02**: 각 Stage 어댑터는 기존 스킬을 수정하지 않고, 호출 인터페이스만 정의 (adapter pattern)
- [ ] **ARCH-03**: SessionStart hook으로 Living State Document를 자동 로드하여 세션 시작 시 맥락 즉시 복원
- [ ] **ARCH-04**: PreToolUse hook으로 Stage 전환 게이트를 강제하여, 필수 artifact 없이 다음 Stage 진입 차단
- [ ] **ARCH-05**: state.json 유실 시 파일시스템의 artifact로부터 상태를 복원하는 reconcile 메커니즘

## v2 Requirements

### Enhanced Automation

- **AUTO-01**: 작업 규모 자동 감지하여 적절한 실행 모드 제안 (scope detection)
- **AUTO-02**: 다수 기능이 서로 다른 Stage에 있을 때 병렬 처리 지원
- **AUTO-03**: Compaction 발생 시 자동 복구 메커니즘 강화

### Extended Integration

- **INTG-01**: bkit PDCA의 상태(.pdca-status.json)와 dev-lifecycle state.json 양방향 동기화
- **INTG-02**: Obsidian vault와 자동 연동하여 프로젝트 대시보드 생성

## Out of Scope

| Feature | Reason |
|---------|--------|
| GSD/PDCA 본체 수정 | 기존 시스템 위에서 오케스트레이션만 담당 |
| 특정 프로젝트 전용 기능 | 범용 스킬이 목적 |
| 픽셀 퍼펙트 목업 | User Interaction Prototype은 플로우 검증용, 디자인 시안이 아님 |
| 자동 테스트 코드 생성 | E2E Spec은 명세이지 테스트 코드가 아님 |
| 별도 서버/DB | Claude Code 스킬 형태로만 동작, 인프라 없음 |
| 자동 Stage 진행 | 사용자 확인 없이 다음 Stage로 넘어가지 않음 |

## Traceability

| Requirement | Phase | Status |
|-------------|-------|--------|
| FOUND-01 | TBD | Pending |
| FOUND-02 | TBD | Pending |
| FOUND-03 | TBD | Pending |
| FOUND-04 | TBD | Pending |
| FOUND-05 | TBD | Pending |
| UIXV-01 | TBD | Pending |
| UIXV-02 | TBD | Pending |
| UIXV-03 | TBD | Pending |
| UIXV-04 | TBD | Pending |
| UIXV-05 | TBD | Pending |
| MEMO-01 | TBD | Pending |
| MEMO-02 | TBD | Pending |
| MEMO-03 | TBD | Pending |
| MEMO-04 | TBD | Pending |
| PIPE-01 | TBD | Pending |
| PIPE-02 | TBD | Pending |
| PIPE-03 | TBD | Pending |
| PIPE-04 | TBD | Pending |
| PIPE-05 | TBD | Pending |
| PIPE-06 | TBD | Pending |
| PIPE-07 | TBD | Pending |
| PIPE-08 | TBD | Pending |
| PIPE-09 | TBD | Pending |
| ARCH-01 | TBD | Pending |
| ARCH-02 | TBD | Pending |
| ARCH-03 | TBD | Pending |
| ARCH-04 | TBD | Pending |
| ARCH-05 | TBD | Pending |

**Coverage:**
- v1 requirements: 28 total
- Mapped to phases: 0
- Unmapped: 28 ⚠️

---
*Requirements defined: 2026-03-22*
*Last updated: 2026-03-22 after initial definition*
