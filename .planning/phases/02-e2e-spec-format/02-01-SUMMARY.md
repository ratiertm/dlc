---
phase: 02-e2e-spec-format
plan: 01
subsystem: spec-format
tags: [e2e-spec, markdown, yaml-frontmatter, 5-step-chain, traceability]

# Dependency graph
requires:
  - phase: 01-foundation
    provides: skill directory structure, manifest.json template, stage-transitions.md
provides:
  - E2E spec template (spec-template.md) with 5-step chain and mandatory Storage field
  - Format reference doc (e2e-spec.md) covering ID convention, cross-artifact linking, status lifecycle
  - Realistic example spec (user-login.spec.md) demonstrating all template fields
affects: [03-prototype-format, 04-plan-adapter, 05-do-adapter, 06-test-adapter]

# Tech tracking
tech-stack:
  added: []
  patterns: [5-step chain (Screen/Connection/Processing/Response/Error), e2e-{feature}-{NNN} spec ID, mandatory Storage field in Processing, deviation log format]

key-files:
  created:
    - skill/templates/spec-template.md
    - skill/references/e2e-spec.md
    - skill/examples/user-login.spec.md
  modified: []

key-decisions:
  - "5-step chain (not 7): Storage as mandatory field within Processing, not separate step -- lower overhead per spec"
  - "Sequential IDs across interactions: e2e-chat-001 through e2e-chat-010 for multi-interaction features, no sub-IDs"
  - "6 error conditions in example (exceeds minimum 3): added invalid email format as separate client-side validation"

patterns-established:
  - "Spec template: YAML frontmatter + H1 title + round-trip summary + 5 chain steps + deviation log"
  - "Chain step format: ## heading with spec ID, Chain type, Status, What, Verification Criteria, Details"
  - "Storage format: {destination} -- {operation} (READ | WRITE | READ+WRITE)"
  - "Cross-artifact linking: spec heading, data-spec-id HTML attribute, // SPEC: code comment, verification.json key"

requirements-completed: [SPEC-01]

# Metrics
duration: 2min
completed: 2026-03-22
---

# Phase 2 Plan 1: E2E Spec Format Summary

**5-step chain spec template (Screen/Connection/Processing/Response/Error) with mandatory Storage field, e2e-{feature}-{NNN} ID convention, and user-login example demonstrating full round-trip flow**

## Performance

- **Duration:** 2 min
- **Started:** 2026-03-22T02:43:53Z
- **Completed:** 2026-03-22T02:46:00Z
- **Tasks:** 2
- **Files modified:** 3

## Accomplishments
- Spec template with all 5 chain steps, chain-specific Detail fields, and mandatory Storage in Processing
- Format reference doc (204 lines) covering ID convention, cross-artifact linking, status lifecycle, PLAN/DO/TEST usage, anti-patterns, manifest integration
- Realistic user-login example spec with 6 error conditions and all verification criteria filled
- All 3 files deployed to both git (skill/) and runtime (~/.claude/skills/dev-lifecycle/)

## Task Commits

Each task was committed atomically:

1. **Task 1: Create spec template and format reference** - `98f0cb4` (feat)
2. **Task 2: Create realistic example spec and deploy to runtime** - `bdbe41c` (feat)

## Files Created/Modified
- `skill/templates/spec-template.md` - Fillable 5-step chain template with YAML frontmatter, chain-specific Details sections, deviation log
- `skill/references/e2e-spec.md` - Format reference doc: ID convention, cross-artifact linking, status lifecycle, storage rules, anti-patterns, manifest integration
- `skill/examples/user-login.spec.md` - Complete login feature spec: 5 chain steps, mandatory Storage field, 6 error conditions

## Decisions Made
- Used 5 steps (not 7) with Storage as mandatory field within Processing -- lower overhead, higher adoption
- Sequential IDs across interactions within a feature (no sub-IDs needed)
- Added 6 error conditions in example (exceeding minimum 3/5) to demonstrate comprehensive error coverage including separate client-side format validation

## Deviations from Plan

None - plan executed exactly as written.

## Issues Encountered
None

## User Setup Required
None - no external service configuration required.

## Next Phase Readiness
- Spec template ready for Phase 3 (Prototype Format) to reference via cross-artifact linking (data-spec-id)
- Spec template ready for Phase 4 (PLAN Adapter) to use when generating feature specs
- Format reference doc provides complete context for DO and TEST adapter phases (5, 6)

## Self-Check: PASSED

All created files verified on disk. All task commits verified in git log.

---
*Phase: 02-e2e-spec-format*
*Completed: 2026-03-22*
