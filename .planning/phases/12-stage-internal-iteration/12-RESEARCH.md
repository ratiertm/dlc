# Phase 12: Stage-Internal Iteration - Research

**Researched:** 2026-03-24
**Domain:** DO stage mini-verification loop + cross-stage Completeness scoring
**Confidence:** HIGH

## Summary

Phase 12 adds two capabilities to the dev-lifecycle skill: (1) a mini-verify loop inside DO stage Step 2, where each spec step is execution-verified immediately after implementation, and (2) a Completeness scoring system (N/10) that spans all stages. Both features modify existing reference files (primarily `do-stage.md`) and require minimal SKILL.md changes through progressive disclosure.

The primary modification target is `do-stage.md` Step 2 (Per-Step Implementation Loop). Currently, step 2d simply updates status to `implemented`. The mini-verify inserts between 2d and 2e: after marking `implemented`, run an execution-level check (build/test/run), and if it fails, retry up to 3 times before reporting to user. The `verification_strictness` config setting maps directly to mini-verify depth: `strict` = full execution, `standard` = execution with override, `relaxed` = code existence only.

The SKILL.md budget is 353/500 lines with 147 remaining. Both features can be added via brief SKILL.md mentions (~10-15 lines total) with detailed logic in references, following the established progressive disclosure pattern.

**Primary recommendation:** Extend do-stage.md Step 2 with a mini-verify sub-step (2d.1), add a new `completeness-scoring.md` reference for the scoring system, and add ~10-15 lines to SKILL.md.

<user_constraints>
## User Constraints (from CONTEXT.md)

### Locked Decisions
- Mini-verify depth: execution-level verification (run/build/test), not just code existence
- Failure loop: auto-retry up to 3 times, then report to user with specific failure details
- Each retry modifies only the failing step's code (not entire feature)
- Recording: mini-verify results in spec.md Status field directly (e.g., "verified (attempt 2)")
- No separate log file -- spec is single source of truth
- Completeness: Claude judges N/10 per stage, not formula-based
- Considers: artifact completeness, spec step coverage, deviation count, verification pass rate
- Decision display: option-by-option Completeness comparison format

### Claude's Discretion
- Exact mini-verify commands per project type (npm test, python -m pytest, flutter test, etc.)
- How to detect "execution succeeded" vs "execution failed" across different tech stacks
- Completeness score weighting across different quality dimensions

### Deferred Ideas (OUT OF SCOPE)
None -- discussion stayed within phase scope
</user_constraints>

<phase_requirements>
## Phase Requirements

| ID | Description | Research Support |
|----|-------------|-----------------|
| ITER-01 | During DO stage, each spec step is mini-verified immediately after implementation | do-stage.md Step 2 extension with new sub-step 2d.1; verification_strictness mapping to mini-verify depth |
| ITER-02 | When mini-verify fails, DO loops on that step until pass (inner QA-Fix-Verify) | Retry loop pattern with max 3 attempts; status recording with attempt count in spec.md |
| ITER-03 | Each stage reports a Completeness score (N/10) based on artifact quality | New completeness-scoring.md reference; brief mention in SKILL.md; score in LIVING-STATE.md |
| ITER-04 | Decision points present options with Completeness comparison | Display format in completeness-scoring.md; integration into decision-point flows across stages |
</phase_requirements>

## Standard Stack

This phase modifies existing Markdown reference files only -- no new libraries or packages.

### Core Files to Modify

| File | Current Lines | Change Type | Purpose |
|------|---------------|-------------|---------|
| `skill/references/do-stage.md` | 318 | Extend Step 2 | Add mini-verify sub-step (2d.1) with retry loop |
| `skill/SKILL.md` | 353 | Add ~10-15 lines | Brief mentions of mini-verify and Completeness |
| `skill/templates/living-state.md` | 67 | Add field | Completeness score display |
| `skill/templates/spec-template.md` | 125 | Update Status values | Add `verified (attempt N)` status format |

### New Files to Create

| File | Purpose | Estimated Lines |
|------|---------|----------------|
| `skill/references/completeness-scoring.md` | Full Completeness scoring logic, display format, decision comparison | ~80-100 |

### Alternatives Considered

| Instead of | Could Use | Tradeoff |
|------------|-----------|----------|
| Separate completeness-scoring.md | Inline in do-stage.md | Completeness is cross-stage; separate file lets all stage adapters reference it |
| Extending do-stage.md only | New mini-verify.md reference | Mini-verify is DO-specific; keeping it in do-stage.md avoids unnecessary indirection |

## Architecture Patterns

### Insertion Point: do-stage.md Step 2

Current Step 2 flow:
```
2a. Read step details
2b. Implement step in code
2c. Add SPEC comment
2d. Update status to "implemented"
2e. Re-display checklist
2f. If deviation needed -> Step 3
```

New Step 2 flow with mini-verify:
```
2a. Read step details
2b. Implement step in code
2c. Add SPEC comment
2d. Update status to "implemented"
2d.1. MINI-VERIFY (new)
  - Resolve verification_strictness from config
  - If strict/standard: execute verification command
  - If relaxed: code existence check only (skip execution)
  - On pass: update status to "verified (attempt 1)"
  - On fail: retry loop (see ITER-02 pattern)
2e. Re-display checklist (now shows verified/failed status)
2f. If deviation needed -> Step 3
```

### Pattern 1: Mini-Verify Execution

**What:** After implementing a spec step, run a project-type-appropriate verification command to confirm it works.
**When to use:** Every spec step during DO stage (controlled by verification_strictness).

```markdown
### Step 2d.1: Mini-Verify

After updating status to `implemented`, verify the step actually works:

**a. Resolve verification depth:**
- Read `verification_strictness` using config resolution (env > config.yaml > default)
- `strict` or `standard`: Execute verification (proceed to step b)
- `relaxed`: Skip execution, status stays `implemented`, continue to 2e

**b. Determine verification command:**
- Read `state.json` `project_type` field
- Select appropriate command:
  | Project Type | Verification Command | Success Signal |
  |-------------|---------------------|----------------|
  | node/react/next | `npm test -- --bail` or `npm run build` | exit code 0 |
  | python | `python -m pytest {test_file} -x` or `python -c "import {module}"` | exit code 0 |
  | flutter | `flutter test {test_file}` or `flutter analyze` | exit code 0 |
  | rust | `cargo check` or `cargo test {test_name}` | exit code 0 |
  | go | `go build ./...` or `go test ./{pkg}` | exit code 0 |
  | generic | Attempt build/compile command from project config | exit code 0 |

- Choose the most targeted command for the specific step:
  - Screen/Response steps: build/compile check
  - Connection/Processing steps: unit test if available, else build
  - Error steps: specific error case test if available

**c. Execute and evaluate:**
- Run the verification command
- Capture: exit code, stdout (last 20 lines), stderr (last 20 lines)
- Success = exit code 0 AND no error patterns in output
- Failure = non-zero exit code OR error patterns detected

**d. On success:**
- Update spec.md status: `**Status:** verified (attempt 1)`
- Continue to step 2e

**e. On failure: enter retry loop (see Step 2d.2)**
```

### Pattern 2: Retry Loop (QA-Fix-Verify)

**What:** When mini-verify fails, automatically fix and retry up to 3 times.
**When to use:** On any mini-verify failure during DO stage.

```markdown
### Step 2d.2: Retry Loop

On mini-verify failure, enter the QA->Fix->Verify loop:

**attempt = 1** (first failure after initial implementation)

**Loop (max 3 attempts):**

1. **Diagnose:** Analyze error output from verification command
   - Identify the specific failure (compile error, test assertion, runtime error)
   - Scope the fix to the failing step's code ONLY (do not modify other steps)

2. **Fix:** Apply targeted fix to the failing code
   - Modify ONLY files related to the current spec step
   - Keep SPEC comment in place

3. **Re-verify:** Run the same verification command again
   - If pass: update status `**Status:** verified (attempt {N})`
   - If fail: increment attempt, continue loop

4. **After 3 failures:** STOP retrying
   - Update status: `**Status:** failed (3 attempts)`
   - Report to user:
     ```
     Mini-verify FAILED after 3 attempts
     =====================================
     Step: e2e-{feature}-{NNN} ({title})
     Last error: {error summary}
     Attempts: 3/3

     The step's code is implemented but verification fails.
     Options:
       1. I'll investigate and fix manually, then resume
       2. Skip verification for this step (mark as implemented)
       3. Revise the spec step if requirements changed
     ```
   - Wait for user direction before continuing
```

### Pattern 3: Completeness Scoring

**What:** Claude judges overall quality N/10 at stage completion.
**When to use:** Every stage completion, and at decision points.

```markdown
## Completeness Scoring

### Stage Completion Score

At the completion of each stage, before the completion announcement, assess Completeness:

**Dimensions (contextual weighting, not formula):**
- Artifact completeness: Are all expected outputs present and well-formed?
- Spec step coverage: What percentage of steps are verified vs implemented vs failed?
- Deviation count: How many deviations from original spec?
- Verification pass rate: Mini-verify successes vs failures (DO stage)
- Edge case coverage: Are error conditions handled? (for TEST stage)

**Score:** N/10 -- Claude's contextual judgment considering all dimensions

**Display at stage completion:**
```
Stage {N} {NAME} -- COMPLETE
Completeness: 8/10 (all steps verified, 1 deviation, error handling thorough)
```

### Decision Point Comparison

When presenting options at any decision point, include Completeness projection:

```
Option A: 6/10 (happy path only, missing error handling for 3 conditions)
Option B: 9/10 (full coverage including edge cases, adds ~30min)
Option C: 7/10 (covers errors but skips loading states)
```

### Living State Integration

Add Completeness to LIVING-STATE.md template:

```markdown
## Current State

- Stage: {N} {NAME} -- {status}
- Completeness: {N}/10 ({brief justification})
```
```

### Anti-Patterns to Avoid

- **Over-engineering mini-verify commands:** Don't try to run the entire test suite for each step. Pick the most targeted, fastest command that validates the specific step.
- **Retrying with the same approach:** Each retry attempt should analyze the specific error and try a different fix, not repeat the same code.
- **Formula-based Completeness:** The user explicitly decided against formulas. Claude judges holistically, considering all dimensions. Don't create weighted averages or point systems.
- **Mini-verify blocking on non-executable steps:** Some steps (e.g., pure documentation or config) may not have a meaningful execution check. Use judgment -- a build check suffices.

## Don't Hand-Roll

| Problem | Don't Build | Use Instead | Why |
|---------|-------------|-------------|-----|
| Project type detection for mini-verify commands | Custom detection logic | Existing `state.json` `project_type` field from `project-detection.md` | Already detected and confirmed by user in session start |
| Config resolution for verification_strictness | Custom env/file reading | Existing 3-layer resolution from `lifecycle-config.md` | Already implemented in Phase 11 |
| Status lifecycle management | New status tracking | Existing spec.md Status field format from `e2e-spec.md` | Just extend values, don't replace the system |
| Stage completion tracking | Custom completion detection | Existing manifest.json output registration | Completeness score is a display addition, not a gate change |

**Key insight:** Phase 12 extends existing patterns rather than creating new systems. The mini-verify slots into the existing Step 2 loop. Completeness scoring is a display/judgment layer on top of existing artifact tracking.

## Common Pitfalls

### Pitfall 1: verification_strictness Mapping Mismatch
**What goes wrong:** CONTEXT.md says "full/basic/skip" but lifecycle-config.md defines "strict/standard/relaxed"
**Why it happens:** The CONTEXT.md used informal names for the verification levels
**How to avoid:** Map explicitly: strict->full execution, standard->execution with override, relaxed->code existence only (skip execution). Document this mapping in do-stage.md.
**Warning signs:** If both naming conventions appear in the same file

### Pitfall 2: Mini-Verify Timeout on Long Builds
**What goes wrong:** Some projects take minutes to build/test, blocking the DO loop
**Why it happens:** Running full build instead of targeted check
**How to avoid:** Always prefer the most targeted command (single test file, single module compile, not full suite). Set a reasonable timeout (~30s for mini-verify).
**Warning signs:** Mini-verify taking longer than the implementation step itself

### Pitfall 3: Retry Loop Modifying Unrelated Code
**What goes wrong:** Fix attempt for step N accidentally breaks step N-1
**Why it happens:** Scope creep during debugging
**How to avoid:** Explicitly constrain fixes to files touched by the current spec step. If the error requires changes outside the step's scope, escalate to user instead of retrying.
**Warning signs:** Git diff showing changes in files unrelated to the current step

### Pitfall 4: SKILL.md Line Budget Overflow
**What goes wrong:** Adding mini-verify and Completeness details pushes SKILL.md past 500 lines
**Why it happens:** Putting detailed logic in SKILL.md instead of references
**How to avoid:** SKILL.md gets ~10-15 lines total (brief mentions with Read directives). All detailed logic goes in do-stage.md and completeness-scoring.md.
**Warning signs:** Any SKILL.md addition exceeding 20 lines

### Pitfall 5: Completeness Score Becoming a Gate
**What goes wrong:** Low Completeness score blocks stage transitions
**Why it happens:** Conflating Completeness (informational) with gates (blocking)
**How to avoid:** Completeness is display-only. Existing gate conditions (manifest.json artifact checks) remain the only blockers. Score is for user information and decision support.
**Warning signs:** If-conditions checking Completeness score before allowing transition

### Pitfall 6: Status Field Format Breaking Existing Parsers
**What goes wrong:** Adding "(attempt N)" to Status field breaks Step 1 checklist parsing
**Why it happens:** Existing parsers look for exact string matches like `**Status:** implemented`
**How to avoid:** The status format `verified (attempt 2)` starts with "verified" which is already a known value. Parsers should match on prefix: `verified` matches both `verified` and `verified (attempt N)`. TEST stage already handles `verified` status. Update parsers to use startsWith rather than exact match.
**Warning signs:** Checklist display showing wrong counts after mini-verify

## Code Examples

### Mini-Verify Integration in do-stage.md Step 2

The key insertion point is between current steps 2d and 2e:

```markdown
**d. Update the step's status in spec.md:**
- Change `**Status:** pending` to `**Status:** implemented`
- Write the change to disk immediately

**d.1. Mini-verify the step (if verification_strictness allows):**

1. Resolve `verification_strictness` using lifecycle-config resolution (env > config.yaml > default)
2. If `relaxed`: skip mini-verify, continue to step 2e
3. Determine verification command based on `project_type` from state.json:
   - Match chain type to command: build check for Screen/Response, test for Connection/Processing/Error
   - Prefer targeted commands (single file/module) over full suite
   - Timeout: 30 seconds max
4. Execute command, capture exit code + last 20 lines of stdout/stderr
5. If exit code = 0: update status `**Status:** verified (attempt 1)`, continue to 2e
6. If exit code != 0: enter retry loop:
   a. Diagnose error from output (compile error? test failure? runtime crash?)
   b. Fix ONLY the failing step's code (scope constraint)
   c. Re-run verification command
   d. If pass: `**Status:** verified (attempt {N})`, continue to 2e
   e. If fail and attempts < 3: increment attempt, repeat from 6a
   f. If fail and attempts = 3: `**Status:** failed (3 attempts)`, report to user, STOP

**e. Re-display the checklist with updated progress:**
```

### Completeness Score Display at DO Completion

Extension to do-stage.md Step 5 (Completion):

```markdown
**d.5. Assess and display Completeness score:**

Evaluate the DO stage output quality:
- How many steps verified vs implemented vs failed?
- How many deviations recorded?
- Were any retries needed? How many?
- Is error handling complete?

Display:
```
DO Stage -- {feature-name} -- COMPLETE
=======================================
Spec: .lifecycle/features/{feature-name}/spec.md
Status: implementing (M/M steps done)
Completeness: 8/10 (all steps verified, 1 deviation, 2 retries needed)

Checklist:
  [x] e2e-{feature}-001: {title} -- verified (attempt 1)
  [x] e2e-{feature}-002: {title} -- verified (attempt 2)
  ...
```

Record Completeness score in state.json for Living State:
- `current.completeness` = `{ "score": 8, "reason": "all steps verified, 1 deviation" }`
```

### LIVING-STATE.md Template Addition

```markdown
## Current State

- Stage: {N} {NAME} -- {status}
- Completeness: {N}/10 ({brief justification})
- Inner loop progress: {stages completed summary}
```

### verification_strictness Mapping Table

To be added to do-stage.md:

```markdown
### verification_strictness and Mini-Verify

| Config Value | Mini-Verify Behavior | Status After |
|-------------|---------------------|-------------|
| `strict` | Execute verification command. Must pass. No override. | `verified (attempt N)` or `failed` |
| `standard` | Execute verification command. Must pass. User can override with justification. | `verified (attempt N)` or `failed` or `implemented (override)` |
| `relaxed` | Skip execution. Code existence check only. | `implemented` (no mini-verify) |
```

## State of the Art

| Old Approach (current) | New Approach (Phase 12) | Impact |
|------------------------|------------------------|--------|
| Status: `implemented` then deferred to TEST | Status: `verified (attempt N)` via immediate mini-verify | Issues caught during DO, not deferred |
| No quality signal until TEST stage | Completeness N/10 at each stage completion | Continuous quality awareness |
| Decision options presented without quality context | Options compared with Completeness projection | Better-informed user decisions |
| TEST stage is first verification point | Mini-verify in DO + full verification in TEST | Layered quality assurance |

## Open Questions

1. **Standard override in mini-verify**
   - What we know: `standard` verification_strictness allows override with justification per lifecycle-config.md
   - What's unclear: Exact UX for override during mini-verify retry (after 3 fails? or at any point?)
   - Recommendation: Allow override only after at least 1 retry attempt. User says "skip verification for this step" and provides justification. Status becomes `implemented (override: {justification})`.

2. **Completeness score persistence format**
   - What we know: Score displays at stage completion and in LIVING-STATE.md
   - What's unclear: Whether to store in state.json, manifest.json, or both
   - Recommendation: Store in state.json under `current.completeness` object. LIVING-STATE.md reads from there during regeneration. Simple, consistent with existing state management.

3. **Mini-verify for non-code steps**
   - What we know: Some spec steps are config-only or documentation
   - What's unclear: What "execution verification" means for these
   - Recommendation: Build/compile check is the minimum. If the project builds successfully after changes, the step passes. Steps that truly have no executable component (rare) can be marked `verified (no-exec)`.

## Validation Architecture

### Test Framework
| Property | Value |
|----------|-------|
| Framework | Manual verification via skill execution |
| Config file | None -- skill files are Markdown instructions |
| Quick run command | Test by running a feature through DO stage |
| Full suite command | Complete inner loop (PLAN-DO-TEST-COMMIT) on test feature |

### Phase Requirements - Test Map
| Req ID | Behavior | Test Type | Automated Command | File Exists? |
|--------|----------|-----------|-------------------|-------------|
| ITER-01 | Mini-verify runs after each spec step | manual-only | Run DO stage, observe mini-verify execution | N/A |
| ITER-02 | Failed mini-verify retries up to 3x | manual-only | Introduce deliberate error, observe retry loop | N/A |
| ITER-03 | Completeness score at stage completion | manual-only | Complete a stage, verify score display | N/A |
| ITER-04 | Decision options show Completeness comparison | manual-only | Trigger a decision point, verify comparison format | N/A |

**Justification for manual-only:** This skill consists of Markdown instruction files that Claude follows. There is no executable code to unit test. Verification requires running Claude through the actual workflow and observing behavior.

### Sampling Rate
- **Per task commit:** Review modified .md files for consistency with spec format and existing patterns
- **Per wave merge:** Run a test feature through the modified DO stage flow
- **Phase gate:** Complete inner loop on a real or test feature confirming all 4 ITER requirements

### Wave 0 Gaps
None -- no test infrastructure needed for Markdown instruction files.

## Sources

### Primary (HIGH confidence)
- `skill/references/do-stage.md` -- Current DO stage Step 2 structure (the insertion point)
- `skill/references/test-stage.md` -- TEST stage verification patterns (reuse for mini-verify)
- `skill/references/lifecycle-config.md` -- verification_strictness definition and resolution
- `skill/references/e2e-spec.md` -- Spec Status field format and lifecycle
- `skill/references/stage-transitions.md` -- Gate logic, Living State regeneration procedure
- `skill/SKILL.md` -- Current structure (353 lines), progressive disclosure pattern
- `skill/templates/spec-template.md` -- Current Status field format
- `skill/templates/config.yaml` -- Default verification_strictness value
- `skill/templates/living-state.md` -- Living State template structure

### Secondary (MEDIUM confidence)
- `.planning/phases/12-stage-internal-iteration/12-CONTEXT.md` -- User decisions and constraints
- `.planning/REQUIREMENTS.md` -- ITER-01 through ITER-04 definitions
- `.planning/STATE.md` -- Line budget concern (353/500), project context

## Metadata

**Confidence breakdown:**
- Standard stack: HIGH -- all files read directly, no external dependencies
- Architecture: HIGH -- insertion points identified precisely in existing code
- Pitfalls: HIGH -- derived from actual code patterns and constraints in the files

**Research date:** 2026-03-24
**Valid until:** 2026-04-24 (stable -- Markdown instruction files, no external dependencies)
