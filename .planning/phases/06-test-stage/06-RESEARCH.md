# Phase 6: TEST Stage - Research

**Researched:** 2026-03-22
**Domain:** Stage 3 TEST adapter -- E2E Spec verification, Prototype structural comparison, user behavioral checklist
**Confidence:** HIGH

## Summary

Phase 6 implements the TEST Stage adapter (`test-stage.md`), which is Stage 3 of the dev-lifecycle pipeline. This stage serves as the final defense line before COMMIT, verifying that DO Stage implementation actually matches the approved E2E Spec. The adapter must perform three distinct verification types: (1) Claude's per-step structural verification of spec against code, (2) prototype manifest structural comparison against actual implementation, and (3) user behavioral verification via step-by-step checklist. Both Claude's structural verification and user behavioral verification must pass before COMMIT gate opens.

The adapter follows the identical structural pattern established by `plan-stage.md` (311 lines) and `do-stage.md` (307 lines): Step 0-N sequential workflow, anti-patterns table, relationship to other skills, and file locations table. Key inputs are the spec file (`.lifecycle/features/{name}/spec.md` with status `implementing` and step statuses `implemented`), the prototype file (`.lifecycle/features/{name}/prototype.html` with embedded JSON manifest), and the code files referenced by `// SPEC:` comments.

**Primary recommendation:** Build test-stage.md as a reference adapter mirroring do-stage.md structure. Produce two outputs: `verification.md` (human-readable report) and `verification.json` (machine-parseable per-step results keyed by spec ID). Both Claude structural + user behavioral verification required before COMMIT transition.

<user_constraints>

## User Constraints (from CONTEXT.md)

### Locked Decisions
- Spec layer pass/fail verification with report generation (decided Phase 2)
- Prototype manifest structural comparison using data-spec-id + JSON manifest dual linking (decided Phase 3)
- FAIL -> report to user, user decides direction (no auto-fix by Claude)
- User behavioral verification via step-by-step checklist
- Claude structural + user behavioral both required before COMMIT

### Claude's Discretion
- Verification report format (JSON vs Markdown) -- recommend both
- Prototype structural comparison method specifics
- Behavioral verification checklist detail format
- test-stage.md reference file structure

### Deferred Ideas (OUT OF SCOPE)
None -- discussion stayed within phase scope

</user_constraints>

<phase_requirements>

## Phase Requirements

| ID | Description | Research Support |
|----|-------------|-----------------|
| SPEC-04 | TEST에서 E2E Spec의 각 단계별 pass/fail 검증 및 리포트 생성 | Step 2 (per-step verification loop) reads each spec step, checks implementation via SPEC comments and verification criteria, produces per-step pass/fail in verification report |
| PROTO-06 | TEST에서 프로토타입의 manifest와 실제 구현을 구조적으로 비교하여 누락 검출 | Step 3 (prototype structural comparison) parses manifest JSON specMapping/screens/interactions/fields/errorStates and checks each against actual implementation files |
| PIPE-03 | Stage 3 TEST -- E2E Spec 단계별 검증 + Prototype 구조 대조 + GSD verify + PDCA analyze | Complete test-stage.md adapter integrating all three verification types plus GSD/PDCA supporting skill invocation |

</phase_requirements>

## Standard Stack

This phase produces documentation files (Markdown reference + templates), not executable code. No npm packages needed.

### Core Deliverables
| Deliverable | Type | Purpose | Pattern Source |
|-------------|------|---------|---------------|
| `test-stage.md` | Reference adapter | TEST Stage workflow instructions | `plan-stage.md` (311 lines), `do-stage.md` (307 lines) |
| SKILL.md Stage 3 update | Skill section | Point Stage 3 to test-stage.md | Existing Stage 1, Stage 2 sections |
| `verification.json` format | Schema definition | Machine-parseable per-step results | Cross-artifact linking in `e2e-spec.md` |
| `verification.md` format | Report template | Human-readable verification report | New (no existing template) |

### Supporting Assets
| Asset | Purpose | When Needed |
|-------|---------|-------------|
| `role-matrix.md` update | Stage 3 role clarification | If current entry needs refinement |
| `stage-transitions.md` | Already defines 2_to_3 and 3_to_4 gates | Reference only, no changes needed |

## Architecture Patterns

### Recommended File Structure
```
skill/
  references/
    test-stage.md          # NEW: TEST Stage adapter (primary deliverable)
    e2e-spec.md            # UPDATE: verification.json format reference
    stage-transitions.md   # NO CHANGE: gates already defined
    role-matrix.md         # POSSIBLE UPDATE: Stage 3 row refinement
  SKILL.md                 # UPDATE: Stage 3 section points to test-stage.md
```

### Pattern 1: Stage Adapter Structure (Established)
**What:** Step 0-N sequential workflow with prerequisites, actions, anti-patterns, and file locations
**When to use:** Every stage adapter follows this pattern
**Evidence:** Both `plan-stage.md` and `do-stage.md` use identical structure

```
## When This Runs
## Prerequisites
## Step 0: Stage Initialization
## Step 1: [First action]
## Step 2: [Second action]
...
## Step N: Completion
## Anti-Patterns
## Relationship to Other Skills
## File Locations
```

### Pattern 2: Spec Status Lifecycle (Established)
**What:** Status transitions tracked in YAML frontmatter and per-step Status fields
**When to use:** TEST updates statuses from `implementing` -> `verified`/`failed` (spec-level) and `implemented` -> `verified`/`failed` (step-level)
**Source:** `e2e-spec.md` Status Lifecycle section

```
Spec-level:  implementing --(TEST passes)--> verified
                          --(TEST fails)---> failed --(DO fixes)--> implementing

Step-level:  implemented  --(TEST verifies)--> verified
                          --(TEST fails)-----> failed
```

### Pattern 3: Dual Output -- Report + Structured Data
**What:** Produce both human-readable Markdown and machine-parseable JSON verification outputs
**When to use:** TEST stage output that must be read by both humans (report) and machines (gate conditions)
**Rationale:** `verification.json` enables programmatic gate checking; `verification.md` enables user review. The cross-artifact linking pattern (spec ID as key) is already established in `e2e-spec.md`.

### Pattern 4: Behavioral Verification Checklist (New)
**What:** Convert each E2E spec step into concrete user-performable actions
**When to use:** User behavioral verification gate
**Rationale:** Claude's structural verification catches code-level issues but cannot verify runtime behavior. User must click through the actual implementation to confirm it works as expected. This mirrors the PLAN Stage agreement gate pattern (user must open prototype.html in browser).

### Anti-Patterns to Avoid
- **Auto-fixing failures:** Claude must NOT automatically fix failed steps. CONTEXT.md explicitly says FAIL -> report to user, user decides direction.
- **Skipping user behavioral verification:** Even if Claude structural verification passes, user must confirm via behavioral checklist before COMMIT.
- **Working from memory:** MUST read spec.md and prototype.html from disk, same as DO Stage pattern.
- **Combining structural and behavioral into one pass:** These are separate verification types. Structural can pass while behavioral fails. Track independently.

## Don't Hand-Roll

| Problem | Don't Build | Use Instead | Why |
|---------|-------------|-------------|-----|
| Gate condition checking | Custom gate logic | Existing `stage-transitions.md` gate rules | Gates already defined: `2_to_3` and `3_to_4` |
| Spec ID parsing | Custom parser | Same pattern as DO Stage Step 1 (scan for `## e2e-{feature}-{NNN}` headings) | Consistency with DO Stage |
| Manifest JSON parsing | Custom schema | `prototype-manifest` script tag + `JSON.parse` | Prototype already embeds manifest in standardized format |
| Status lifecycle | Custom state machine | `e2e-spec.md` Status Lifecycle definitions | Statuses already defined, just apply them |
| Verification output key format | Custom key scheme | Spec ID as key (`e2e-{feature}-{NNN}`) | Cross-artifact linking already uses spec ID everywhere |

**Key insight:** TEST Stage consumes outputs from PLAN (spec + prototype) and DO (implemented code + SPEC comments). All the data structures, ID conventions, and linking patterns are already defined. TEST just needs to read, compare, and report.

## Common Pitfalls

### Pitfall 1: Forgetting Deviation-Aware Verification
**What goes wrong:** TEST verifies against the original spec, ignoring approved deviations recorded during DO Stage.
**Why it happens:** The spec file contains both original step content AND a Deviations section. If TEST only reads step content, it will flag approved deviations as failures.
**How to avoid:** Step 2 (per-step verification) must read the `## Deviations` section first and adjust expectations for steps with approved deviations (DEV-NNN entries with `Approved: yes`).
**Warning signs:** Steps that the user explicitly approved as deviations are flagged as FAIL.

### Pitfall 2: Prototype Comparison Scope Creep
**What goes wrong:** TEST tries to compare visual appearance or CSS styling between prototype and implementation.
**Why it happens:** The prototype is a clickable HTML file that looks like a UI. Natural instinct is to compare how things look.
**How to avoid:** Prototype comparison is STRUCTURAL only -- does the implementation have the same screens, interactions, fields, and error states listed in the manifest? Prototypes are for flow validation, not design (stated in `prototype.md`).
**Warning signs:** Comparisons mention colors, fonts, layout, or pixel measurements.

### Pitfall 3: Blocking on Non-Automated Verification
**What goes wrong:** TEST Stage tries to run automated tests against the implementation (e.g., starting a server, running Playwright).
**Why it happens:** "TEST" implies automated testing. But this is a Claude Code skill -- it operates by reading files, not running servers.
**How to avoid:** Claude's verification is code-level structural analysis (reading source files, checking SPEC comments exist, verifying functions/components match spec). Runtime behavioral verification is delegated to the user via checklist.
**Warning signs:** Commands that try to `npm start`, `npm test`, or launch browsers.

### Pitfall 4: Ambiguous Pass/Fail Criteria
**What goes wrong:** Verification report says "partially implemented" or gives percentages instead of clear pass/fail per step.
**Why it happens:** Some steps may be partially implemented -- e.g., the endpoint exists but error handling is missing.
**How to avoid:** Each spec step gets exactly one status: `verified` or `failed`. If ANY verification criteria for a step is unchecked, the step is `failed`. Partial implementation = fail.
**Warning signs:** Report contains "partial", "mostly done", percentage scores.

### Pitfall 5: Missing SPEC Comment Verification
**What goes wrong:** TEST verifies functionality but does not check that `// SPEC: e2e-{feature}-{NNN}` comments exist in the code.
**Why it happens:** SPEC comments seem cosmetic, but they are the traceability thread.
**How to avoid:** Step 2 verification must include checking that each spec step has a corresponding `// SPEC:` comment in the codebase. Missing SPEC comment = traceability gap = warning (not fail, but reported).
**Warning signs:** Verification report does not mention SPEC comment presence.

## Code Examples

### Verification JSON Output Format
```json
{
  "feature": "user-login",
  "specFile": ".lifecycle/features/user-login/spec.md",
  "prototypeFile": ".lifecycle/features/user-login/prototype.html",
  "verifiedAt": "2026-03-22T12:00:00Z",
  "specVerification": {
    "overall": "passed",
    "steps": {
      "e2e-login-001": {
        "title": "Login Form Display",
        "chain": "Screen",
        "status": "verified",
        "criteria": [
          { "text": "Login form renders at /login route", "passed": true },
          { "text": "Email input exists with type=\"email\"", "passed": true }
        ],
        "specComment": { "found": true, "file": "src/pages/Login.tsx", "line": 15 },
        "deviations": []
      },
      "e2e-login-003": {
        "title": "Credential Validation",
        "chain": "Processing",
        "status": "failed",
        "criteria": [
          { "text": "Email lookup queries the users table", "passed": true },
          { "text": "Password comparison uses bcrypt", "passed": false, "reason": "plain text comparison found" }
        ],
        "specComment": { "found": true, "file": "src/api/auth.ts", "line": 42 },
        "deviations": []
      }
    }
  },
  "prototypeVerification": {
    "overall": "passed",
    "screens": {
      "expected": ["login", "loading", "dashboard"],
      "found": ["login", "loading", "dashboard"],
      "missing": []
    },
    "specMapping": {
      "total": 5,
      "matched": 5,
      "missing": []
    },
    "interactions": {
      "total": 2,
      "matched": 2,
      "missing": []
    },
    "fields": {
      "total": 2,
      "matched": 2,
      "missing": []
    },
    "errorStates": {
      "total": 2,
      "matched": 2,
      "missing": []
    }
  },
  "userVerification": {
    "status": "pending",
    "checklist": [
      { "step": "e2e-login-001", "action": "1. Open /login in browser", "passed": null },
      { "step": "e2e-login-001", "action": "2. Verify email input, password input, submit button visible", "passed": null },
      { "step": "e2e-login-002", "action": "3. Enter test@example.com / password123, click Login", "passed": null },
      { "step": "e2e-login-003", "action": "4. Verify loading/processing state appears briefly", "passed": null },
      { "step": "e2e-login-004", "action": "5. Verify dashboard shows with Welcome message", "passed": null },
      { "step": "e2e-login-005", "action": "6. Go back to /login, try empty fields, verify error", "passed": null },
      { "step": "e2e-login-005", "action": "7. Try wrong credentials, verify error message", "passed": null }
    ]
  }
}
```

### Verification Markdown Report Format
```markdown
# Verification Report: {feature-name}

**Verified:** {ISO timestamp}
**Spec:** .lifecycle/features/{name}/spec.md
**Overall:** PASSED / FAILED

## Claude Structural Verification

### Spec Step Results

| Step | Title | Chain | Status | Issues |
|------|-------|-------|--------|--------|
| e2e-login-001 | Login Form Display | Screen | PASS | -- |
| e2e-login-003 | Credential Validation | Processing | FAIL | bcrypt not used |

### Prototype Structural Comparison

| Category | Expected | Found | Missing |
|----------|----------|-------|---------|
| Screens | 3 | 3 | -- |
| Spec Mappings | 5 | 5 | -- |
| Interactions | 2 | 2 | -- |
| Fields | 2 | 2 | -- |
| Error States | 2 | 2 | -- |

### SPEC Comment Traceability

| Step | Comment Found | File | Line |
|------|--------------|------|------|
| e2e-login-001 | Yes | src/pages/Login.tsx | 15 |

## User Behavioral Verification

Please perform these steps and confirm:

1. [ ] Open /login in browser
2. [ ] Verify email input, password input, submit button visible
3. [ ] Enter test@example.com / password123, click Login
...

## Summary

- Spec steps: N passed, M failed
- Prototype: N/M categories matched
- SPEC comments: N/M found
- User verification: PENDING
```

### Test-Stage Step 0: Stage Initialization Pattern
```markdown
## Step 0: Stage Initialization

**Actions:**

1. Read `.lifecycle/state.json` and verify:
   - `current.feature` has a value
   - `current.stage` = `3` (or transition from 2 to 3)

2. Check gate condition (2_to_3):
   - Read `.lifecycle/manifest.json`
   - Verify: `artifacts.2_do.outputs.length > 0`
   - If gate fails: set `current.status = "blocked"`, display missing artifacts, STOP

3. Update `state.json`:
   - `current.stage` = `3`
   - `current.stage_name` = `"TEST"`
   - `current.status` = `"in_progress"`

4. Read spec from `.lifecycle/features/{feature-name}/spec.md`:
   - Verify YAML frontmatter `status: implementing`
   - Verify at least one step has `**Status:** implemented`

5. Read prototype from `.lifecycle/features/{feature-name}/prototype.html`:
   - Parse manifest JSON from `<script type="application/json" id="prototype-manifest">`
```

### Behavioral Checklist Generation Pattern
```markdown
For each spec step, generate a user-performable action:

Screen step    -> "Open {route/page} in browser. Verify {elements} are visible."
Connection step -> "Perform {user action} (e.g., click button, submit form). Verify {request indicator}."
Processing step -> "Verify {processing feedback} appears (loading state, spinner)."
Response step   -> "Verify {expected result} is displayed (data, navigation, UI update)."
Error step      -> "Test each error condition: {condition}. Verify {expected error behavior}."
```

## State of the Art

| Old Approach (SKILL.md current) | New Approach (After Phase 6) | Impact |
|--------------------------------|------------------------------|--------|
| Stage 3 TEST: generic "plan-vs-implementation completeness" | Per-step spec verification + prototype structural comparison + user behavioral checklist | Concrete, traceable verification instead of vague completeness check |
| GSD `/gsd:verify-work` as primary for Stage 3 | dev-lifecycle as primary, GSD as supporting | Spec-driven verification replaces generic verification |
| No verification output format | `verification.json` + `verification.md` | Machine-parseable + human-readable outputs |
| No user verification step | Behavioral checklist with explicit pass/fail per step | Runtime behavior validated by user, not just code-level checks |

## Validation Architecture

### Test Framework
| Property | Value |
|----------|-------|
| Framework | Manual verification (Claude Code skill -- no automated test runner) |
| Config file | None -- skill produces reference docs, not executable code |
| Quick run command | N/A -- verification is manual review of produced adapter file |
| Full suite command | N/A |

### Phase Requirements -> Test Map
| Req ID | Behavior | Test Type | Automated Command | File Exists? |
|--------|----------|-----------|-------------------|-------------|
| SPEC-04 | Per-step pass/fail verification + report | manual-only | Review `test-stage.md` Step 2 covers all spec step statuses | Wave 0 |
| PROTO-06 | Prototype manifest structural comparison | manual-only | Review `test-stage.md` Step 3 covers all manifest sections | Wave 0 |
| PIPE-03 | Complete TEST stage pipeline | manual-only | Review `test-stage.md` Steps 0-5 form complete pipeline | Wave 0 |

**Justification for manual-only:** This phase produces Markdown reference documents (skill adapter files), not executable code. Verification = review that the adapter instructions correctly implement the required behaviors. The adapter itself will be validated when actually used on a real feature in a future session.

### Sampling Rate
- **Per task commit:** Manual review of adapter completeness
- **Per wave merge:** Cross-reference against CONTEXT.md decisions
- **Phase gate:** All three requirements covered in test-stage.md

### Wave 0 Gaps
None -- no test infrastructure needed for documentation-only deliverables.

## Open Questions

1. **Verification report file format: single or dual?**
   - What we know: CONTEXT.md says Claude's discretion for report format (JSON vs Markdown)
   - Recommendation: Produce BOTH `verification.json` (machine-parseable, used by gates) and `verification.md` (human-readable, for user review). The JSON is canonical, the Markdown is derived from it.

2. **Backward transition on failure: explicit or automatic?**
   - What we know: `stage-transitions.md` says backward transitions do NOT require gate checks and reset target stage to `in_progress`
   - What's unclear: Should TEST automatically transition back to DO on failure, or just report and let user decide?
   - Recommendation: Report only. CONTEXT.md explicitly says "FAIL -> report to user, user decides direction." Do NOT auto-transition back to DO.

3. **GSD verify-work integration depth**
   - What we know: `role-matrix.md` lists GSD as primary for Stage 3, PDCA as supporting
   - What's unclear: With dev-lifecycle now doing spec-specific verification, GSD's role has shifted
   - Recommendation: dev-lifecycle is primary for spec/prototype verification. GSD `/gsd:verify-work` is available as optional supporting tool. PDCA `/pdca analyze` for gap analysis. Adapter should note this role shift.

## Sources

### Primary (HIGH confidence)
- `skill/references/plan-stage.md` -- Stage adapter structural pattern (Step 0-N, anti-patterns, file locations)
- `skill/references/do-stage.md` -- DO Stage outputs (spec with `implementing` status, SPEC comments, deviations)
- `skill/references/e2e-spec.md` -- Status lifecycle (implementing -> verified/failed), cross-artifact linking, verification.json key format
- `skill/references/prototype.md` -- Manifest schema (specMapping, interactions, fields, errorStates), dual spec linking rules
- `skill/references/stage-transitions.md` -- Gate conditions (2_to_3, 3_to_4), backward transition rules
- `skill/references/role-matrix.md` -- Stage 3 skill assignments, output types (verification, test-report)
- `skill/SKILL.md` -- Current Stage 3 TEST section (lines 87-100), needs update to point to test-stage.md

### Secondary (MEDIUM confidence)
- `skill/examples/user-login.spec.md` -- Concrete spec example used to derive verification.json format
- `skill/examples/user-login.prototype.html` -- Concrete prototype example used to derive structural comparison pattern

### Tertiary (LOW confidence)
- None -- all findings derived from existing project artifacts

## Metadata

**Confidence breakdown:**
- Standard stack: HIGH -- all deliverables follow established adapter pattern from plan-stage.md and do-stage.md
- Architecture: HIGH -- status lifecycle, cross-artifact linking, and manifest schema are already defined in spec/prototype references
- Pitfalls: HIGH -- derived from established patterns (deviation handling, prototype purpose, Claude Code limitations)

**Research date:** 2026-03-22
**Valid until:** 2026-04-22 (stable -- documentation-only phase with well-defined inputs)
