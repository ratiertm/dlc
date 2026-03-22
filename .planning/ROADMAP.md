# Roadmap: Dev Lifecycle Orchestrator

## Overview

This roadmap delivers a Claude Code skill that orchestrates the full development lifecycle with two core innovations woven into every inner loop stage: E2E Feature Specs and User Interaction Prototypes. PLAN inherently creates specs and prototypes. DO inherently implements against them. TEST inherently verifies against them. The roadmap starts with foundation and format definitions, then builds each inner loop stage with spec+prototype integration baked in, followed by memory systems, the outer loop, and architectural hardening. 10 phases at fine granularity.

## Phases

**Phase Numbering:**
- Integer phases (1, 2, 3): Planned milestone work
- Decimal phases (2.1, 2.2): Urgent insertions (marked with INSERTED)

Decimal phases appear between their surrounding integers in numeric order.

- [ ] **Phase 1: Foundation** - Generic skill scaffold, state management, artifact registry, role matrix
- [x] **Phase 2: E2E Spec Format** - Define the 5-layer spec chain format and spec ID system (completed 2026-03-22)
- [ ] **Phase 3: Prototype Format** - Define the single-file HTML+JS prototype template with data attributes and manifest
- [ ] **Phase 4: PLAN Stage** - Spec generation + prototype generation + user agreement gate
- [ ] **Phase 5: DO Stage** - Spec-as-checklist implementation + deviation logging + ADR detection
- [ ] **Phase 6: TEST Stage** - Spec layer verification + prototype structural diff + manual test steps
- [ ] **Phase 7: COMMIT Stage** - Verification gate + artifact validation + why-centric commit
- [ ] **Phase 8: Memory & Decision Trail** - Settings changelog, decision log, Living State Document, code-ADR linking
- [ ] **Phase 9: Outer Loop** - DEPLOY, DEPLOY TEST, DOCUMENT, RETROSPECT, PROMOTE stages
- [ ] **Phase 10: Skill Architecture & Resilience** - Progressive disclosure, session hooks, state reconciliation, execution modes

## Phase Details

### Phase 1: Foundation
**Goal**: A generic, project-agnostic dev-lifecycle skill exists with working state management and artifact tracking
**Depends on**: Nothing (first phase)
**Requirements**: FOUND-01, FOUND-02, FOUND-03, FOUND-05
**Success Criteria** (what must be TRUE):
  1. User can invoke the dev-lifecycle skill from any project directory without muse-specific references or assumptions
  2. state.json persists current stage, active feature, and progress across sessions
  3. Each stage's output is registered in a manifest and available as required input to the next stage
  4. A role matrix document clearly states which skill (GSD, PDCA, ADR, retro, work-log) handles what within each stage
**Plans**: 2 plans

Plans:
- [ ] 01-01-PLAN.md — Create references/ and templates/ foundation artifacts (role-matrix, stage-transitions, project-detection, skill-invocation, state.json, manifest.json)
- [ ] 01-02-PLAN.md — Rewrite SKILL.md: remove muse hardcoding, implement progressive disclosure

### Phase 2: E2E Spec Format
**Goal**: A well-defined, machine-parseable E2E spec format exists that captures the full feature chain from screen to error handling
**Depends on**: Phase 1
**Requirements**: SPEC-01
**Success Criteria** (what must be TRUE):
  1. A feature interaction can be described using the 5-layer chain (Screen, Connection, Processing, Response, Error) with clear per-layer fields
  2. Each spec step has a unique spec ID (e2e-NNN) that can be referenced across all artifacts (spec, prototype, code, verification)
  3. The format is documented with at least one realistic example spec
**Plans**: 1 plan

Plans:
- [ ] 02-01-PLAN.md — Create spec template, format reference doc, and realistic user-login example spec

### Phase 3: Prototype Format
**Goal**: A prototype template exists that produces clickable single-file HTML prototypes with embedded manifest and semantic data attributes
**Depends on**: Phase 2
**Requirements**: PROTO-01, PROTO-02, PROTO-03, PROTO-04
**Success Criteria** (what must be TRUE):
  1. A single HTML file opens with file:// in any browser and renders a clickable multi-screen prototype
  2. Hash-based SPA routing enables screen-to-screen navigation without a server
  3. Every interactive element carries data-* attributes (data-screen, data-action, data-field, data-error) linking back to spec IDs
  4. An embedded JSON manifest lists all screens, actions, and fields so tools can parse the prototype programmatically
**Plans**: TBD

Plans:
- [ ] 03-01: TBD

### Phase 4: PLAN Stage
**Goal**: The PLAN stage produces both an E2E spec and a clickable prototype for each feature, blocking until user approves both
**Depends on**: Phase 2, Phase 3
**Requirements**: SPEC-02, PROTO-05, PIPE-01
**Success Criteria** (what must be TRUE):
  1. Running PLAN for a feature generates both spec.md and prototype.html in the feature directory
  2. User can open the prototype in a browser, click through all screens, and provide feedback
  3. PLAN does not complete until user explicitly approves both spec and prototype (hard agreement gate)
  4. Preflight check validates prerequisites before spec/prototype generation begins
**Plans**: TBD

Plans:
- [ ] 04-01: TBD
- [ ] 04-02: TBD

### Phase 5: DO Stage
**Goal**: The DO stage uses the approved E2E spec as an implementation checklist, tracking per-layer completion and logging deviations
**Depends on**: Phase 4
**Requirements**: SPEC-03, SPEC-05, PIPE-02
**Success Criteria** (what must be TRUE):
  1. During implementation, each spec layer (Screen, Connection, Processing, Response, Error) is tracked with a completion status
  2. If implementation deviates from the approved spec, the deviation is recorded in a log with reason and impact
  3. ADR-worthy decisions are detected and recorded with WHY+SEE comments in the code
  4. The DO stage refuses to start without a PLAN-approved spec
**Plans**: TBD

Plans:
- [ ] 05-01: TBD
- [ ] 05-02: TBD

### Phase 6: TEST Stage
**Goal**: The TEST stage verifies implementation against both the E2E spec (per-layer pass/fail) and the prototype (structural comparison), producing actionable results for both Claude and the user
**Depends on**: Phase 5
**Requirements**: SPEC-04, PROTO-06, PIPE-03
**Success Criteria** (what must be TRUE):
  1. Each spec layer is verified with a clear PASS/FAIL status and a verification report is generated
  2. Prototype manifest is structurally compared against actual implementation to detect missing screens, actions, or fields
  3. TEST produces specific manual test steps the user can execute for behavioral verification
  4. Both Claude's structural verification and user's behavioral verification are required before proceeding
**Plans**: TBD

Plans:
- [ ] 06-01: TBD
- [ ] 06-02: TBD

### Phase 7: COMMIT Stage
**Goal**: The COMMIT stage blocks until all spec steps pass verification, then creates a why-centric commit with artifact references
**Depends on**: Phase 6
**Requirements**: PIPE-04, ARCH-04
**Success Criteria** (what must be TRUE):
  1. COMMIT is blocked if any spec step has FAIL status -- user sees a specific gap list pointing to what needs fixing
  2. The commit message centers on "why" and references relevant ADRs
  3. Stage transition gate validates all required artifacts (spec, prototype, verification report) exist before allowing COMMIT
**Plans**: TBD

Plans:
- [ ] 07-01: TBD

### Phase 8: Memory & Decision Trail
**Goal**: Every settings change, decision, and state transition is automatically recorded and accessible for future context restoration
**Depends on**: Phase 1
**Requirements**: MEMO-01, MEMO-02, MEMO-03, MEMO-04
**Success Criteria** (what must be TRUE):
  1. When a setting or config changes, the change is automatically recorded with context (why) and value (what)
  2. Lightweight decisions accumulate in a one-line-per-entry log without requiring full ADR ceremony
  3. A Living State Document exists that, when read at session start, restores full project context instantly
  4. ADR references appear as WHY+SEE comments in code, making future follow-up traceable
**Plans**: TBD

Plans:
- [ ] 08-01: TBD
- [ ] 08-02: TBD

### Phase 9: Outer Loop
**Goal**: Stages 5-9 (DEPLOY, DEPLOY TEST, DOCUMENT, RETROSPECT, PROMOTE) are operational with project-type-aware templates
**Depends on**: Phase 7
**Requirements**: PIPE-05, PIPE-06, PIPE-07, PIPE-08, PIPE-09
**Success Criteria** (what must be TRUE):
  1. User can deploy using a project-type-appropriate template (web, API, CLI, desktop)
  2. Smoke tests verify server state, core flows, and resources after deployment
  3. DOCUMENT stage generates or updates architecture diagrams, CLAUDE.md, and README.md
  4. RETROSPECT produces a retrospective with ADR gap check, work-log entry, and Living State update
  5. PROMOTE optionally generates demo content (skippable without penalty)
**Plans**: TBD

Plans:
- [ ] 09-01: TBD
- [ ] 09-02: TBD
- [ ] 09-03: TBD

### Phase 10: Skill Architecture & Resilience
**Goal**: The skill is production-ready with progressive disclosure, automatic session restoration, state resilience, and execution mode support
**Depends on**: Phase 8, Phase 9
**Requirements**: ARCH-01, ARCH-02, ARCH-03, ARCH-05, FOUND-04
**Success Criteria** (what must be TRUE):
  1. SKILL.md is under 500 lines with detailed logic in references/ files loaded on demand
  2. Existing skills (GSD, PDCA, ADR, retro, work-log) are called through adapter interfaces without modification
  3. SessionStart hook automatically loads the Living State Document so context is restored without user action
  4. If state.json is lost or corrupted, the system reconstructs state from filesystem artifacts (reconcile)
  5. User can select execution mode (hotfix/feature/release/milestone) to skip unnecessary stages
**Plans**: TBD

Plans:
- [ ] 10-01: TBD
- [ ] 10-02: TBD

## Progress

**Execution Order:**
Phases execute in numeric order: 1 -> 2 -> 3 -> 4 -> 5 -> 6 -> 7 -> 8 -> 9 -> 10
Note: Phase 8 depends only on Phase 1 and can run in parallel with Phases 2-7 if desired.

| Phase | Plans Complete | Status | Completed |
|-------|----------------|--------|-----------|
| 1. Foundation | 0/2 | Not started | - |
| 2. E2E Spec Format | 1/1 | Complete   | 2026-03-22 |
| 3. Prototype Format | 0/1 | Not started | - |
| 4. PLAN Stage | 0/2 | Not started | - |
| 5. DO Stage | 0/2 | Not started | - |
| 6. TEST Stage | 0/2 | Not started | - |
| 7. COMMIT Stage | 0/1 | Not started | - |
| 8. Memory & Decision Trail | 0/2 | Not started | - |
| 9. Outer Loop | 0/3 | Not started | - |
| 10. Skill Architecture & Resilience | 0/2 | Not started | - |
