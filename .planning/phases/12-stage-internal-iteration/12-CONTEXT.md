# Phase 12: Stage-Internal Iteration - Context

**Gathered:** 2026-03-24
**Status:** Ready for planning

<domain>
## Phase Boundary

The DO stage catches implementation issues immediately through step-level mini-verification, and all stages communicate quality through Completeness scoring. This modifies the existing DO stage adapter and adds scoring across all stages.

</domain>

<decisions>
## Implementation Decisions

### Mini-verify depth
- Execution-level verification: after implementing each spec step, actually run/build/test to confirm it works
- Not just code existence — verify the function runs, the API responds, the UI renders
- verification_strictness config setting controls behavior: full (execute), basic (code exists), skip (no mini-verify)

### Failure loop
- On mini-verify failure: auto-retry up to 3 times with fixes
- After 3 failures: report to user with specific failure details, user decides direction
- Each retry attempt modifies only the failing step's code (not entire feature)

### Recording
- Mini-verify results recorded directly in spec.md — each step's Status field updated to verified/failed
- Include attempt count in status (e.g., "verified (attempt 2)")
- No separate log file — spec is the single source of truth

### Completeness scoring
- Claude judges overall quality N/10 per stage — not formula-based, contextual judgment
- Considers: artifact completeness, spec step coverage, deviation count, verification pass rate
- Score displayed at stage completion and in Living State Document

### Decision point display
- When presenting options during any stage, show Completeness comparison per option
- Format: "Option A: 6/10 (missing error handling) vs Option B: 9/10 (full coverage)"
- Helps user make informed trade-off decisions

### Claude's Discretion
- Exact mini-verify commands per project type (npm test, python -m pytest, flutter test, etc.)
- How to detect "execution succeeded" vs "execution failed" across different tech stacks
- Completeness score weighting across different quality dimensions

</decisions>

<canonical_refs>
## Canonical References

**Downstream agents MUST read these before planning or implementing.**

### DO Stage (primary modification target)
- `skill/references/do-stage.md` — Current DO stage workflow, Step 0-5 structure, spec checklist tracking
- `skill/references/test-stage.md` — TEST stage verification patterns (reference for mini-verify design)

### Configuration (controls iteration behavior)
- `skill/references/lifecycle-config.md` — verification_strictness setting definition and resolution
- `skill/templates/config.yaml` — Default config values

### Spec format
- `skill/references/e2e-spec.md` — Spec step structure, Status field format
- `skill/templates/spec-template.md` — Spec template with Status fields

### Architecture
- `skill/SKILL.md` — Current SKILL.md structure (353 lines, 500 budget)
- `skill/references/stage-transitions.md` — Gate logic, config-aware mode resolution

</canonical_refs>

<code_context>
## Existing Code Insights

### Reusable Assets
- do-stage.md Step 2 (Per-step implementation): already tracks per-step completion — extend with mini-verify after each step
- test-stage.md Step 2 (Per-step verification): verification patterns can be reused for mini-verify
- lifecycle-config.md: verification_strictness setting already defined (full/basic/skip)

### Established Patterns
- Spec Status field: `**Status:** not_started | implementing | implemented | verified | failed`
- Stage gate checks via manifest.json outputs array
- Progressive disclosure: SKILL.md section + Read directive to reference

### Integration Points
- do-stage.md Step 2 is the insertion point for mini-verify loop
- SKILL.md Stage 2 section needs Completeness mention
- Living State template needs Completeness score field

</code_context>

<specifics>
## Specific Ideas

- gstack QA→Fix→Verify pattern: iterate until clean or max retries
- gstack Completeness scoring: "Completeness: A=5/10 (happy path), B=9/10 (all SQL patterns)"
- Mini-verify should feel like a natural part of DO, not a separate stage insertion

</specifics>

<deferred>
## Deferred Ideas

None — discussion stayed within phase scope

</deferred>

---

*Phase: 12-stage-internal-iteration*
*Context gathered: 2026-03-24*
