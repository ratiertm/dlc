# TEST Stage Adapter

This reference defines the complete TEST Stage workflow -- Stage 3 of the dev-lifecycle pipeline. When Claude reads this file, it follows the instructions below to verify a feature implementation against its approved E2E spec and prototype.

## When This Runs

- User says "test", "verify", "check", or similar
- `state.json` `current.stage = 3` (or transitioning from 2 to 3)
- SKILL.md Stage 3 section directs here via `Read: $CLAUDE_SKILL_DIR/references/test-stage.md`

## Prerequisites

- `.lifecycle/` directory initialized (from PLAN stage)
- `state.json` and `manifest.json` exist
- DO stage must have outputs -- gate 2_to_3 must pass (`artifacts.2_do.outputs.length > 0`)

## Step 0: Stage Initialization

Before any verification work, verify the pipeline state and load all inputs.

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
   - `progress.current_stage_started_at` = current ISO 8601 timestamp

4. Read spec from `.lifecycle/features/{feature-name}/spec.md`:
   - **MUST read from disk** -- do NOT work from memory or context window
   - Verify YAML frontmatter `status: implementing`
   - Verify at least one step has `**Status:** implemented`
   - If status is not `implementing`: STOP, inform user DO stage must complete first

5. Read prototype from `.lifecycle/features/{feature-name}/prototype.html`:
   - Parse manifest JSON from `<script type="application/json" id="prototype-manifest">`
   - Store parsed manifest for Step 3 comparison

6. Read Deviations section from spec.md:
   - Scan for `## Deviations` section
   - Build deviation lookup: `{ "e2e-{feature}-{NNN}": { devId: "DEV-NNN", original, actual, approved } }`
   - Only deviations with `**Approved:** yes` adjust verification expectations
   - If no Deviations section exists, deviation lookup is empty (no deviations recorded)

## Step 1: Spec Step Enumeration

**Purpose:** Parse all spec steps and present the verification plan.

**CRITICAL:** Read spec.md from disk (`.lifecycle/features/{feature-name}/spec.md`). Do NOT enumerate steps from memory -- the spec file is the source of truth.

**Actions:**

1. Parse all spec steps by scanning for `## e2e-{feature}-{NNN}` headings
2. Extract from each step: spec ID, title, Chain type, Status field, Verification Criteria
3. Count approved deviations from the deviation lookup (Step 0.6)
4. Display verification plan:

```
TEST Stage -- {feature-name}
===========================
Spec: .lifecycle/features/{name}/spec.md
Prototype: .lifecycle/features/{name}/prototype.html
Steps to verify: N
Approved deviations: M

Verification plan:
  [ ] e2e-{feature}-001: {title} ({chain})
  [ ] e2e-{feature}-002: {title} ({chain}) [DEV-001]
  ...
```

Steps with approved deviations are marked with their DEV-NNN reference.

## Step 2: Per-Step Spec Verification

**Purpose:** Verify each implemented spec step against its verification criteria, accounting for approved deviations. This addresses requirement SPEC-04.

For each step with `**Status:** implemented`:

**a. Read the step's Verification Criteria from spec.md:**
- Each step has a Verification Criteria section with checkbox items
- These are the concrete checks that must pass

**b. Check deviations:**
- Look up the step's spec ID in the deviation lookup (from Step 0.6)
- If the step has an approved deviation (DEV-NNN with `**Approved:** yes`):
  - Adjust expected behavior to match the deviation's `**Actual:**` field
  - The original spec criteria that conflict with the deviation are superseded
  - Record the deviation reference in the verification result
- If no deviation exists, verify against the original spec criteria

**c. Search codebase for SPEC comment:**
- Search for `// SPEC: e2e-{feature}-{NNN}` in the project source files
- Record: found (true/false), file path, line number
- Missing SPEC comment is a traceability warning (reported but does not cause FAIL)

**d. Verify implementation at the SPEC comment location:**
- Read the surrounding code at the SPEC comment location
- Check each Verification Criteria item against the actual implementation
- For each criterion: mark passed (true) or failed (false) with reason if failed

**e. Set step status:**
- `verified` -- ALL verification criteria pass (accounting for approved deviations)
- `failed` -- ANY verification criterion fails
- There is no "partial" status. Partial implementation = `failed`.

**f. Update step status in spec.md on disk:**
- Change `**Status:** implemented` to `**Status:** verified` or `**Status:** failed`
- Write the change to disk immediately

**g. Re-display progress after each step:**

```
Verification progress:
  [PASS] e2e-{feature}-001: {title}
  [FAIL] e2e-{feature}-002: {title} -- {failure reason}
  [ .. ] e2e-{feature}-003: {title}
  ...
Progress: 2/N verified
```

**Critical rules:**
- MUST read spec.md from disk (not memory) for each step's criteria
- MUST account for approved deviations before checking criteria
- Partial implementation = FAIL (no "partial" or "mostly done" status)
- Missing SPEC comment = warning, not failure (traceability gap reported separately)

## Step 3: Prototype Structural Comparison

**Purpose:** Compare the prototype manifest against actual implementation to detect structural gaps. This addresses requirement PROTO-06. This is STRUCTURAL comparison only -- not visual.

**Actions:**

**a. Parse prototype manifest JSON:**
- Use the manifest parsed in Step 0.5
- Extract: `specMapping`, `screens`, `interactions`, `fields`, `errorStates`

**b. For each screen in manifest `screens[]`:**
- Check that a corresponding route, page, or view exists in the implementation
- Match by screen name to route/component naming
- Record: expected, found, missing

**c. For each entry in manifest `specMapping[]`:**
- Check that the mapped spec step was verified or at least implemented (cross-reference with Step 2 results)
- Record: total mappings, matched, missing

**d. For each interaction in manifest `interactions[]`:**
- Check that the action handler exists in the implementation code
- Look for event handlers, click handlers, form submit handlers matching the interaction's action/element
- Record: total, matched, missing

**e. For each field in manifest `fields[]`:**
- Check that the input, select, textarea, or form field exists in the implementation
- Match by field name/type to actual form elements
- Record: total, matched, missing

**f. For each error state in manifest `errorStates[]`:**
- Check that the error condition is handled in the implementation code
- Look for error display logic, error messages, validation feedback matching the trigger/message
- Record: total, matched, missing

**g. Compile comparison results:**

```
Prototype Structural Comparison
================================
| Category     | Expected | Found | Missing |
|-------------|----------|-------|---------|
| Screens      | 3        | 3     | --      |
| Spec Mappings| 5        | 5     | --      |
| Interactions | 2        | 2     | --      |
| Fields       | 2        | 1     | email   |
| Error States | 2        | 2     | --      |
```

**Critical rules:**
- Do NOT compare visual appearance, CSS, colors, or layout
- Do NOT compare pixel positions, font sizes, or styling
- Prototype is for flow validation, not design (per prototype.md)
- Structural = "do the same screens, interactions, fields, and error states exist?"

## Step 4: User Behavioral Verification Checklist

**Purpose:** Generate concrete, user-performable actions derived from spec steps so the user can verify runtime behavior. This addresses requirement PIPE-03.

**Generation rules by chain type:**

| Chain Type | Checklist Action Template |
|-----------|--------------------------|
| Screen | "Open {route/page} in browser. Verify {elements} are visible." |
| Connection | "Perform {user action} (click button, submit form). Verify {request indicator} (network call, loading state)." |
| Processing | "Verify {processing feedback} appears (loading spinner, progress bar, brief delay)." |
| Response | "Verify {expected result} is displayed (data, success message, navigation to new page)." |
| Error | "Test {error condition}. Verify {expected error behavior} (error message, validation highlight, recovery option)." |

**Actions:**

1. For each spec step, generate one or more checklist items using the template above
2. Number sequentially across all steps (1, 2, 3, ... N)
3. Include the spec step ID reference for traceability: `[e2e-{feature}-{NNN}]`
4. For Error steps with multiple conditions (from the Conditions table), generate one checklist item per condition
5. Account for approved deviations -- if a step has a deviation, use the deviation's `**Actual:**` behavior for the checklist item

**Output format:**

```
User Behavioral Verification Checklist
=======================================
Feature: {feature-name}
Steps: N items

Please perform each action and mark pass/fail:

1. [ ] [e2e-{feature}-001] Open /login in browser. Verify email input, password input, and submit button are visible.
2. [ ] [e2e-{feature}-002] Enter test credentials and click Login. Verify network request is initiated (loading state appears).
3. [ ] [e2e-{feature}-003] Verify loading spinner or processing indicator appears briefly.
4. [ ] [e2e-{feature}-004] Verify dashboard page loads with Welcome message and user data.
5. [ ] [e2e-{feature}-005] Leave email field empty and click Login. Verify "Email required" error appears.
6. [ ] [e2e-{feature}-005] Enter invalid credentials. Verify "Invalid email or password" error appears.
```

**Present this checklist to the user.** The user must perform each action on the running implementation and confirm pass/fail.

## Step 5: Completion

**Purpose:** Generate verification outputs, update pipeline state, and determine overall result.

**Actions:**

**a. Generate `verification.json`** at `.lifecycle/features/{feature-name}/verification.json`:

```json
{
  "feature": "{feature-name}",
  "specFile": ".lifecycle/features/{feature-name}/spec.md",
  "prototypeFile": ".lifecycle/features/{feature-name}/prototype.html",
  "verifiedAt": "{ISO-8601}",
  "specVerification": {
    "overall": "passed|failed",
    "steps": {
      "e2e-{feature}-{NNN}": {
        "title": "{step title}",
        "chain": "{chain type}",
        "status": "verified|failed",
        "criteria": [
          { "text": "{criterion text}", "passed": true|false, "reason": "{if failed}" }
        ],
        "specComment": { "found": true|false, "file": "{path}", "line": "{number}" },
        "deviations": ["DEV-NNN"]
      }
    }
  },
  "prototypeVerification": {
    "overall": "passed|failed",
    "screens": { "expected": [], "found": [], "missing": [] },
    "specMapping": { "total": 0, "matched": 0, "missing": [] },
    "interactions": { "total": 0, "matched": 0, "missing": [] },
    "fields": { "total": 0, "matched": 0, "missing": [] },
    "errorStates": { "total": 0, "matched": 0, "missing": [] }
  },
  "userVerification": {
    "status": "pending|passed|failed",
    "checklist": [
      { "step": "e2e-{feature}-{NNN}", "action": "{checklist item text}", "passed": null|true|false }
    ]
  }
}
```

**b. Generate `verification.md`** at `.lifecycle/features/{feature-name}/verification.md`:

```markdown
# Verification Report: {feature-name}

**Verified:** {ISO timestamp}
**Spec:** .lifecycle/features/{name}/spec.md
**Prototype:** .lifecycle/features/{name}/prototype.html
**Overall:** PASSED / FAILED

## Claude Structural Verification

### Spec Step Results

| Step | Title | Chain | Status | Deviation | Issues |
|------|-------|-------|--------|-----------|--------|
| e2e-{feature}-001 | {title} | {chain} | PASS/FAIL | -- | {if any} |

### Prototype Structural Comparison

| Category | Expected | Found | Missing |
|----------|----------|-------|---------|
| Screens | N | N | {items} |
| Spec Mappings | N | N | {items} |
| Interactions | N | N | {items} |
| Fields | N | N | {items} |
| Error States | N | N | {items} |

### SPEC Comment Traceability

| Step | Comment Found | File | Line |
|------|--------------|------|------|
| e2e-{feature}-{NNN} | Yes/No | {file} | {line} |

## User Behavioral Verification

Please perform these steps and confirm:

1. [ ] [e2e-{feature}-{NNN}] {action description}
...

## Summary

- Spec steps: N passed, M failed
- Prototype: N/M categories fully matched
- SPEC comments: N/M found
- User verification: PENDING / PASSED / FAILED
```

**c. Update spec.md YAML frontmatter:**
- If ALL steps verified: `status: implementing` -> `status: verified`
- If ANY step failed: `status: implementing` -> `status: failed`

**d. Register outputs in manifest.json** under `artifacts.3_test.outputs`:

```json
[
  {
    "type": "verification",
    "path": ".lifecycle/features/{feature-name}/verification.json",
    "created_at": "{ISO-8601}",
    "status": "complete"
  },
  {
    "type": "test-report",
    "path": ".lifecycle/features/{feature-name}/verification.md",
    "created_at": "{ISO-8601}",
    "status": "complete"
  }
]
```

**e. Determine overall result and next action:**

**Case 1: Any spec step FAILED or prototype has missing items:**
- Overall result = FAILED
- Display failure summary with specific failed steps and missing items
- **Do NOT auto-fix.** Do NOT automatically transition back to DO stage.
- Report failures to user and let user decide direction
- Per CONTEXT.md locked decision: "FAIL -> report to user, user decides direction"

```
TEST Stage -- {feature-name} -- FAILED
======================================
Spec verification: N/M steps passed
Prototype comparison: {missing items}

Failed steps:
  [FAIL] e2e-{feature}-{NNN}: {title} -- {reason}

Action required: Review failures above. You may:
  1. Return to DO stage to fix specific failures
  2. Accept current state and proceed (not recommended)
  3. Revise the spec if requirements changed
```

**Case 2: All structural checks PASS:**
- Announce: "Claude structural verification PASSED. User behavioral verification PENDING."
- Present the behavioral checklist (from Step 4) to the user
- COMMIT gate is blocked until user confirms behavioral verification
- Wait for user to perform each checklist item and report results

**Case 3: User confirms all behavioral checks PASS (full PASS):**
- Overall result = PASSED
- Update `verification.json` userVerification status to "passed"
- Update state.json:
  - `current.status` = `"completed"`
  - `session.last_active` = current ISO 8601 timestamp
  - `session.resume_hint` = `"TEST complete. All verification passed. Ready for COMMIT stage."`
- Announce: "TEST complete. All verification passed. Ready for COMMIT stage."

**Case 4: User reports any behavioral check FAIL:**
- Overall result = FAILED
- Update `verification.json` userVerification status to "failed", mark failed items
- Follow Case 1 protocol (report to user, do NOT auto-fix)

## Anti-Patterns

These are the most common mistakes. Do NOT do any of these:

| # | Anti-Pattern | Why It Is Wrong |
|---|-------------|-----------------|
| 1 | **Auto-fixing failures** | Claude must NOT fix failed steps. CONTEXT.md explicitly states: FAIL -> report to user, user decides direction. Auto-fix removes user agency and may introduce new issues. |
| 2 | **Skipping user behavioral verification** | Even if Claude structural verification passes, user must confirm via behavioral checklist before COMMIT gate opens. Runtime behavior cannot be verified by reading code alone. |
| 3 | **Working from memory instead of reading spec from disk** | Spec IDs and criteria in memory may not match the actual file. Always read `.lifecycle/features/{feature-name}/spec.md` from disk. Same rule as DO Stage. |
| 4 | **Combining structural and behavioral into one pass** | These are independent verification types. Structural can pass while behavioral fails. Track and report them separately in verification outputs. |
| 5 | **Comparing prototype visual appearance** | Prototype comparison is STRUCTURAL only -- do the same screens, interactions, fields, and error states exist? Do NOT compare CSS, colors, fonts, layout, or pixel positions. |
| 6 | **Ambiguous pass/fail** | No "partial", "mostly done", or percentage scores. Each spec step is exactly `verified` or `failed`. If ANY criterion for a step fails, the step is `failed`. |

## Relationship to Other Skills

- **dev-lifecycle:** Primary skill for Stage 3. This adapter orchestrates the TEST Stage workflow, including spec verification, prototype comparison, and behavioral checklist generation.
- **GSD:** Available as supporting tool. User can invoke `/gsd:verify-work` for broader project-level verification beyond spec-specific checks. GSD does not replace spec-driven verification.
- **PDCA:** Available as supporting tool. User can invoke `/pdca analyze` for gap analysis. PDCA's analytical framework complements but does not replace per-step spec verification.
- **ADR:** Not directly involved in Stage 3. ADRs created during DO Stage are informational context but do not affect verification logic.

**Note on role shift:** dev-lifecycle is now primary for Stage 3 verification (spec-driven). This replaces the earlier model where GSD was primary for Stage 3. GSD's `/gsd:verify-work` remains available as a supporting tool for broader verification needs.

## File Locations

| File | Path | Purpose |
|------|------|---------|
| This adapter | `$CLAUDE_SKILL_DIR/references/test-stage.md` | TEST Stage workflow instructions |
| Spec format reference | `$CLAUDE_SKILL_DIR/references/e2e-spec.md` | ID convention, status lifecycle, deviation log format |
| Prototype format reference | `$CLAUDE_SKILL_DIR/references/prototype.md` | Manifest schema, data attributes, dual spec linking |
| Stage transitions | `$CLAUDE_SKILL_DIR/references/stage-transitions.md` | Gate conditions (2_to_3, 3_to_4) |
| State template | `$CLAUDE_SKILL_DIR/templates/state.json` | State schema with stage/status fields |
| Manifest template | `$CLAUDE_SKILL_DIR/templates/manifest.json` | Artifact registry with gate rules |
| Verification output (JSON) | `.lifecycle/features/{name}/verification.json` | Machine-parseable per-step results |
| Verification output (MD) | `.lifecycle/features/{name}/verification.md` | Human-readable verification report |

**Runtime location:** `$CLAUDE_SKILL_DIR` = `~/.claude/skills/dev-lifecycle`
**Version-controlled location:** `skill/` directory in the project repository
