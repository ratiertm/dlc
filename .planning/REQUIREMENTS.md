# Requirements: Dev Lifecycle Orchestrator

**Defined:** 2026-03-22
**Core Value:** 개발 과정 산출물이 AI의 외부 메모리 역할을 하여, 재작업과 맥락 유실을 근본적으로 방지한다.

## v1 Requirements

### E2E Feature Spec (핵심)

- [x] **SPEC-01**: 각 기능을 화면→연결→처리→응답→에러 5단계 체인으로 명세하는 포맷 정의
- [x] **SPEC-02**: E2E Spec을 PLAN에서 작성하고 사용자와 합의하는 워크플로
- [x] **SPEC-03**: DO에서 E2E Spec의 각 단계를 구현 체크리스트로 추적
- [x] **SPEC-04**: TEST에서 E2E Spec의 각 단계별 pass/fail 검증 및 리포트 생성
- [x] **SPEC-05**: DO 중 Spec에서 이탈 시 deviation log에 기록하는 메커니즘

### User Interaction Prototype (핵심)

- [x] **PROTO-01**: PLAN에서 단일 HTML 파일(vanilla HTML+CSS+JS)로 클릭 가능한 프로토타입 생성
- [x] **PROTO-02**: hash 기반 SPA 라우팅으로 멀티스크린 네비게이션 구현 (서버 불필요)
- [x] **PROTO-03**: data-* 속성(data-screen, data-action, data-field, data-error)으로 UI 요소에 의미 부여
- [x] **PROTO-04**: 내장 JSON manifest로 프로토타입 구조를 기계 파싱 가능하게 유지
- [x] **PROTO-05**: 사용자가 프로토타입을 브라우저에서 클릭하여 확인하고 피드백하는 합의 게이트
- [x] **PROTO-06**: TEST에서 프로토타입의 manifest와 실제 구현을 구조적으로 비교하여 누락 검출

### Foundation

- [x] **FOUND-01**: muse 하드코딩 제거 — 모든 프로젝트 타입에서 사용 가능한 범용 스킬 전환
- [x] **FOUND-02**: state.json으로 현재 Stage, 진행 상태, 활성 기능 추적 (세션 간 유지)
- [x] **FOUND-03**: Stage Artifact Manifest — 각 Stage 산출물을 기록하고 다음 Stage 필수 입력으로 연결
- [ ] **FOUND-04**: 상황별 실행 모드(핫픽스/피처/릴리즈/마일스톤) — 불필요한 Stage 생략 가능
- [x] **FOUND-05**: GSD/PDCA/커스텀 스킬 역할 매트릭스 — 각 Stage에서 어떤 스킬이 어떤 역할

### Memory & Decision Trail

- [x] **MEMO-01**: 세팅/설정 변경 시 맥락(왜)+값(무엇) 자동 영구 기록
- [x] **MEMO-02**: ADR보다 가벼운 한 줄짜리 결정 이력(Lightweight Decision Log) 누적
- [x] **MEMO-03**: Living State Document — 세션 시작 시 읽으면 전체 맥락 즉시 복원
- [ ] **MEMO-04**: ADR 생성 시 코드에 WHY+SEE 주석 삽입하여 F/U 가능

### Pipeline — Inner Loop (Stage 1-4)

- [x] **PIPE-01**: Stage 1 PLAN — preflight check + E2E Spec 작성 + Prototype 생성 + 사용자 합의 게이트
- [x] **PIPE-02**: Stage 2 DO — Spec 기준 구현 + deviation log + ADR 자동 감지 + WHY+SEE 주석
- [x] **PIPE-03**: Stage 3 TEST — E2E Spec 단계별 검증 + Prototype 구조 대조 + GSD verify + PDCA analyze
- [x] **PIPE-04**: Stage 4 COMMIT — why 중심 커밋 + ADR 참조 + artifact 존재 검증

### Pipeline — Outer Loop (Stage 5-9)

- [ ] **PIPE-05**: Stage 5 DEPLOY — 프로젝트 타입별 범용 배포 템플릿
- [ ] **PIPE-06**: Stage 6 DEPLOY TEST — 스모크 테스트 (서버 상태, 핵심 플로우, 리소스)
- [ ] **PIPE-07**: Stage 7 DOCUMENT — 아키텍처/시퀀스 다이어그램, CLAUDE.md/README.md 업데이트
- [ ] **PIPE-08**: Stage 8 RETROSPECT — 회고 + ADR 누락 체크 + work-log + Living State 반영
- [ ] **PIPE-09**: Stage 9 PROMOTE — 선택적 데모 영상 생성

### Skill Architecture

- [ ] **ARCH-01**: SKILL.md 500줄 이하, 상세 로직은 references/ 분리 (progressive disclosure)
- [ ] **ARCH-02**: 기존 스킬 수정 없이 호출 인터페이스만 정의 (adapter pattern)
- [ ] **ARCH-03**: SessionStart hook으로 Living State Document 자동 로드
- [x] **ARCH-04**: Stage 전환 게이트 — 필수 artifact 없이 다음 Stage 진입 차단
- [ ] **ARCH-05**: state.json 유실 시 파일시스템에서 상태 복원 (reconcile)

## v2 Requirements

### Enhanced Automation

- **AUTO-01**: 작업 규모 자동 감지하여 실행 모드 제안 (scope detection)
- **AUTO-02**: 다수 기능 병렬 Stage 처리
- **AUTO-03**: Compaction 자동 복구 강화

### Extended Formats

- **FMT-01**: 비웹 프로젝트용 프로토타입 변형 (API simulator, CLI simulator)
- **FMT-02**: E2E Spec 7단계 확장 (+ state change + side effects)
- **FMT-03**: multi-screen flow를 위한 spec 연결 포맷

### Integration

- **INTG-01**: bkit PDCA 상태와 양방향 동기화
- **INTG-02**: Obsidian vault 자동 연동

## Out of Scope

| Feature | Reason |
|---------|--------|
| GSD/PDCA 본체 수정 | 오케스트레이션만 담당 |
| 특정 프로젝트 전용 기능 | 범용 스킬 목적 |
| 픽셀 퍼펙트 목업 | Prototype은 플로우 검증용, 디자인 시안 아님 |
| 자동 테스트 코드 생성 | E2E Spec은 명세이지 테스트 코드가 아님 |
| 별도 서버/DB | Claude Code 스킬로만 동작 |
| 자동 Stage 진행 | 사용자 확인 없이 넘어가지 않음 |
| 0% 실패율 달성 | 현실적으로 20-30% 잔여, 성능/크로스기능/비결정성은 미해결 |

## Traceability

| Requirement | Phase | Status |
|-------------|-------|--------|
| SPEC-01 | Phase 2 | Complete |
| SPEC-02 | Phase 4 | Complete |
| SPEC-03 | Phase 5 | Complete |
| SPEC-04 | Phase 6 | Complete |
| SPEC-05 | Phase 5 | Complete |
| PROTO-01 | Phase 3 | Complete |
| PROTO-02 | Phase 3 | Complete |
| PROTO-03 | Phase 3 | Complete |
| PROTO-04 | Phase 3 | Complete |
| PROTO-05 | Phase 4 | Complete |
| PROTO-06 | Phase 6 | Complete |
| FOUND-01 | Phase 1 | Complete |
| FOUND-02 | Phase 1 | Complete |
| FOUND-03 | Phase 1 | Complete |
| FOUND-04 | Phase 10 | Pending |
| FOUND-05 | Phase 1 | Complete |
| MEMO-01 | Phase 8 | Complete |
| MEMO-02 | Phase 8 | Complete |
| MEMO-03 | Phase 8 | Complete |
| MEMO-04 | Phase 8 | Pending |
| PIPE-01 | Phase 4 | Complete |
| PIPE-02 | Phase 5 | Complete |
| PIPE-03 | Phase 6 | Complete |
| PIPE-04 | Phase 7 | Complete |
| PIPE-05 | Phase 9 | Pending |
| PIPE-06 | Phase 9 | Pending |
| PIPE-07 | Phase 9 | Pending |
| PIPE-08 | Phase 9 | Pending |
| PIPE-09 | Phase 9 | Pending |
| ARCH-01 | Phase 10 | Pending |
| ARCH-02 | Phase 10 | Pending |
| ARCH-03 | Phase 10 | Pending |
| ARCH-04 | Phase 7 | Complete |
| ARCH-05 | Phase 10 | Pending |

**Coverage:**
- v1 requirements: 34 total
- Mapped to phases: 34
- Unmapped: 0

---
*Requirements defined: 2026-03-22*
*Last updated: 2026-03-22 after roadmap v2 (SPEC/PROTO integrated into inner loop)*
