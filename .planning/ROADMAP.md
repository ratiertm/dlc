# Roadmap: Dev Lifecycle Orchestrator

## Overview

### v1.0 (Completed)

10 phases delivered the core dev-lifecycle skill: foundation, E2E spec format, prototype format, inner loop stages (PLAN/DO/TEST/COMMIT), memory & decision trail, outer loop (DEPLOY through PROMOTE), and skill architecture hardening. All 34 v1.0 requirements complete.

### v2.0 (Active) -- gstack Pattern Adoption

5 phases adopt proven patterns from gstack: file-based configuration, stage-internal iteration loops, safe upgrade/migration, observability/analytics, and ecosystem skill integration. Phases numbered 11-15 continuing from v1.0. 19 requirements at fine granularity.

## Phases

**Phase Numbering:**
- Phases 1-10: v1.0 (all complete)
- Phases 11-15: v2.0 (active milestone)
- Decimal phases (e.g., 12.1): Urgent insertions (marked with INSERTED)

- [x] ~~Phase 1-10: v1.0 Foundation through Architecture~~ (all complete)
- [x] **Phase 11: Configuration** - File-based lifecycle-config with layered settings and change tracking (completed 2026-03-23)
- [ ] **Phase 12: Stage-Internal Iteration** - Mini-verify loops in DO, Completeness scoring, decision comparison
- [ ] **Phase 13: Lifecycle Upgrade** - Schema migration, migration markers, rollback, changelog
- [ ] **Phase 14: Observability & Analytics** - Session tracking, stage transition logs, rework events, snapshot diff, time metrics
- [ ] **Phase 15: Ecosystem Integration** - gstack skill suggestions at stage transitions, opt-out, extensible mapping

## Phase Details

### Phase 11: Configuration
**Goal**: Users can manage lifecycle behavior through a unified config system with layered precedence and automatic change tracking
**Depends on**: Phase 10 (v1.0 complete)
**Requirements**: CONF-01, CONF-02, CONF-03
**Success Criteria** (what must be TRUE):
  1. User can get/set any lifecycle setting (mode, skip_stages, proactive, auto_skip, verification_strictness) via lifecycle-config commands
  2. Settings resolve correctly through three layers: environment variable overrides .lifecycle/config.yaml overrides built-in defaults
  3. Every config change is automatically logged in settings-changelog with the reason for the change
**Plans**: 2 plans
Plans:
- [ ] 11-01-PLAN.md — Create config template and lifecycle-config reference (resolution algorithm, get/set/list, changelog integration)
- [ ] 11-02-PLAN.md — Integrate config into SKILL.md and update stage-transitions for config-aware mode resolution

### Phase 12: Stage-Internal Iteration
**Goal**: The DO stage catches implementation issues immediately through step-level mini-verification, and all stages communicate quality through Completeness scoring
**Depends on**: Phase 11 (config controls iteration behavior)
**Requirements**: ITER-01, ITER-02, ITER-03, ITER-04
**Success Criteria** (what must be TRUE):
  1. After implementing each spec step in DO, a mini-verify check runs immediately (not deferred to the TEST stage)
  2. When a mini-verify fails, DO automatically loops on that step (QA -> Fix -> Verify) until it passes before moving to the next step
  3. Each stage completion reports a Completeness score (N/10) based on artifact quality and coverage metrics
  4. When a decision point has multiple options, each option shows its Completeness comparison (e.g., Option A: 6/10 vs Option B: 9/10) to inform the choice
**Plans**: TBD

### Phase 13: Lifecycle Upgrade
**Goal**: Users can safely upgrade dev-lifecycle to new versions with automatic migration, rollback safety, and clear communication of changes
**Depends on**: Phase 11 (config schema is the first thing that needs migration support)
**Requirements**: UPGR-01, UPGR-02, UPGR-03, UPGR-04
**Success Criteria** (what must be TRUE):
  1. After git pull, .lifecycle/ files are automatically migrated to the new schema version without manual intervention
  2. Migration markers (e.g., .lifecycle/.v2-migrated) prevent completed migrations from re-running on subsequent upgrades
  3. If a migration fails, the user can roll back to the pre-upgrade state with all data intact
  4. After upgrade, the user sees a summary of what changed (new features, schema changes, deprecated settings)
**Plans**: TBD

### Phase 14: Observability & Analytics
**Goal**: Users have full visibility into their development process through session context files, stage transition analytics, rework tracking, and spec-vs-implementation diffs
**Depends on**: Phase 12 (iteration data feeds into analytics)
**Requirements**: OBSV-01, OBSV-02, OBSV-03, OBSV-04, OBSV-05
**Success Criteria** (what must be TRUE):
  1. Each session creates a context file in .lifecycle/sessions/ capturing decisions made and artifacts touched during that session
  2. Stage transitions are logged to .lifecycle/analytics/stage-transitions.jsonl with timestamps, enabling transition pattern analysis
  3. Backward stage transitions (rework) are tracked in .lifecycle/analytics/rework-events.jsonl with reason and impact
  4. User can generate a before/after snapshot diff comparing the original spec baseline against the current implementation delta
  5. Time-per-stage metrics are recorded and available so the user can identify bottleneck stages
**Plans**: TBD

### Phase 15: Ecosystem Integration
**Goal**: Stage transitions proactively suggest relevant companion skills, with user control over suggestion behavior and an extensible skill mapping
**Depends on**: Phase 11 (proactive setting), Phase 14 (analytics inform suggestions)
**Requirements**: ECOS-01, ECOS-02, ECOS-03
**Success Criteria** (what must be TRUE):
  1. At each stage transition, the system suggests relevant gstack/companion skills (e.g., review at COMMIT, qa at TEST, investigate at DO)
  2. User can disable all proactive suggestions by setting proactive: false in lifecycle-config
  3. The skill suggestion mapping is documented in a reference file and extensible (new skills can be added without modifying core logic)
**Plans**: TBD

## Progress

**v1.0 Execution (Complete):**
Phases 1-10 all complete. 34/34 requirements delivered.

**v2.0 Execution Order:**
Phase 11 -> 12 -> 13 -> 14 -> 15
Note: Phase 13 (Upgrade) depends on Phase 11 but is otherwise independent of Phase 12. Could run in parallel with 12 if desired.

| Phase | Plans Complete | Status | Completed |
|-------|----------------|--------|-----------|
| 1-10. v1.0 | 20/20 | Complete | 2026-03-22 |
| 11. Configuration | 2/2 | Complete    | 2026-03-23 |
| 12. Stage-Internal Iteration | 0/? | Not started | - |
| 13. Lifecycle Upgrade | 0/? | Not started | - |
| 14. Observability & Analytics | 0/? | Not started | - |
| 15. Ecosystem Integration | 0/? | Not started | - |
