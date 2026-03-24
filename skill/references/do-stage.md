# DO Stage Adapter

This reference defines the complete DO Stage workflow -- Stage 2 of the dev-lifecycle pipeline. When Claude reads this file, it follows the instructions below to implement a feature against an approved E2E spec.

## When This Runs

- User says "implement", "build", "do", or similar
- `state.json` `current.stage = 2` (or transitioning from 1 to 2)
- SKILL.md Stage 2 section directs here via `Read: $CLAUDE_SKILL_DIR/references/do-stage.md`

## Prerequisites

- `.lifecycle/` directory initialized (from PLAN stage)
- `state.json` and `manifest.json` exist
- Approved spec required -- gate 1_to_2 must pass (`artifacts.1_plan.outputs.length > 0 && artifacts.1_plan.completed_at != null`)

## Step 0: Stage Initialization

Before any implementation work, verify the pipeline state and load the approved spec.

**Actions:**

1. Read `.lifecycle/state.json` and verify:
   - `current.feature` has a value
   - `current.stage` = `2` (or transition from 1 to 2)

2. Check gate condition (1_to_2):
   - Read `.lifecycle/manifest.json`
   - Verify: `artifacts.1_plan.outputs.length > 0`
   - Verify: `artifacts.1_plan.completed_at != null`
   - If gate fails: set `current.status = "blocked"`, display missing artifacts, STOP

3. Update `state.json`:
   - `current.stage` = `2`
   - `current.stage_name` = `"DO"`
   - `current.status` = `"in_progress"`
   - `progress.current_stage_started_at` = current ISO 8601 timestamp

4. Read approved spec from `.lifecycle/features/{feature-name}/spec.md`:
   - **MUST read from disk** -- do NOT work from memory or context window
   - Verify YAML frontmatter `status: agreed`
   - If status is not `agreed`: STOP, inform user spec must be approved first

5. Update spec YAML frontmatter:
   - `status: agreed` -> `status: implementing`

## Step 1: Spec Load and Checklist Build

**Purpose:** Parse the approved spec and present it as an implementation checklist.

**CRITICAL:** Read spec.md from disk (`.lifecycle/features/{feature-name}/spec.md`). Do NOT generate the checklist from memory -- the spec file is the source of truth for IDs and step details.

**Actions:**

1. Parse all spec steps by scanning for `## e2e-{feature}-{NNN}` headings
2. Extract from each step: spec ID, title, Chain type, Status field
3. Build and display the checklist:

```
DO Stage -- {feature-name}
======================
Spec: .lifecycle/features/{feature-name}/spec.md
Status: implementing (N/M steps done)

Checklist:
  [x] e2e-{feature}-001: {title} ({chain})
  [ ] e2e-{feature}-002: {title} ({chain})
  ...
Deviations: 0 recorded
ADRs created: 0
Next step: e2e-{feature}-{NNN} ({chain})
```

4. Identify the first step with `**Status:** pending` as the starting point

**Edge case -- resuming DO:** If some steps already show `**Status:** implemented`, mark those as `[x]` in the checklist and resume from the first `pending` step.

## Step 2: Per-Step Implementation Loop

**Purpose:** Implement each spec step in order, tracking progress through the checklist.

For each spec step with `**Status:** pending`:

**a. Read the step's details:**
- Chain type (Screen, Connection, Processing, Response, Error)
- What section (describes what to implement)
- Details section (chain-specific fields: Element, Method, Storage, etc.)
- Verification Criteria (what "done" looks like)

**b. Implement the step in code:**
- Follow the spec's instructions for what to build
- Match the chain type to the appropriate implementation pattern

**c. Add SPEC comment at the implementation site:**
```
// SPEC: e2e-{feature}-{NNN} -- {step title}
```
- Place directly above the function/component/handler that implements the step
- One comment per spec step, at the most relevant code location
- For steps spanning multiple files, place at the entry point

**d. Update the step's status in spec.md:**
- Change `**Status:** pending` to `**Status:** implemented`
- Write the change to disk immediately

**d.1. Mini-verify the step:**

Resolve `verification_strictness` using lifecycle-config resolution (env `LIFECYCLE_VERIFICATION_STRICTNESS` > `.lifecycle/config.yaml` > default `standard`):

| Config Value | Mini-Verify Behavior | Status After |
|-------------|---------------------|-------------|
| `strict` | Execute verification command. Must pass. No override. | `verified (attempt N)` or `failed (3 attempts)` |
| `standard` | Execute verification command. Must pass. User can override after at least 1 retry. | `verified (attempt N)` or `failed (3 attempts)` or `implemented (override: {justification})` |
| `relaxed` | Skip execution. Code existence check only. | Status stays `implemented`, continue to 2e |

If `relaxed`: skip mini-verify, continue to step 2e.

Otherwise, determine verification command from `state.json` `project_type`:

| Project Type | Verification Command | Success Signal |
|-------------|---------------------|----------------|
| node/react/next | `npm test -- --bail` or `npm run build` | exit code 0 |
| python | `python -m pytest {test_file} -x` or `python -c "import {module}"` | exit code 0 |
| flutter | `flutter test {test_file}` or `flutter analyze` | exit code 0 |
| rust | `cargo check` or `cargo test {test_name}` | exit code 0 |
| go | `go build ./...` or `go test ./{pkg}` | exit code 0 |
| generic | Attempt build/compile command from project config | exit code 0 |

Choose the most targeted command for the specific step:
- Screen/Response steps: build/compile check
- Connection/Processing steps: unit test if available, else build
- Error steps: specific error case test if available
- Timeout: 30 seconds max

Execute and evaluate:
- Run the verification command
- Capture: exit code, stdout (last 20 lines), stderr (last 20 lines)
- Success = exit code 0 AND no error patterns in output
- On success: update spec.md status to `**Status:** verified (attempt 1)`, continue to step 2e
- On failure: enter retry loop (step 2d.2)

**d.2. Retry loop (on mini-verify failure):**

On mini-verify failure, enter the QA->Fix->Verify loop (max 3 attempts):

1. **Diagnose:** Analyze error output from verification command
   - Identify the specific failure (compile error, test assertion, runtime error)
   - Scope the fix to the failing step's code ONLY (do not modify other steps' files)

2. **Fix:** Apply targeted fix to the failing code
   - Modify ONLY files related to the current spec step
   - Keep SPEC comment in place

3. **Re-verify:** Run the same verification command again
   - If pass: update status `**Status:** verified (attempt {N})`, continue to 2e
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
   - Wait for user direction before continuing to next step

Note for `standard` strictness: user can also say "skip verification" after at least 1 retry attempt. Status becomes `**Status:** implemented (override: {justification})`.

**e. Re-display the checklist with updated progress:**
- Show updated `[x]`/`[ ]` markers
- Update "N/M steps done" count
- Show next pending step

**f. If implementation must differ from spec:**
- Go to Step 3 (Deviation Handling) before continuing
- Return here after deviation is resolved

## Step 3: Deviation Handling

**Purpose:** When implementation must differ from spec, record the deviation with user confirmation.

**Trigger:** Any substantive difference between what the spec says and what is being implemented -- endpoint changes, storage destination differs, additional processing steps, error conditions changed, UI layout differs.

**Workflow:**

**a. Pause implementation immediately.** Do not continue until the deviation is resolved.

**b. Present deviation to user in this exact format:**

```
Spec Deviation Detected
============================
Step: e2e-{feature}-{NNN} ({step title})
Spec says: {original spec detail}
Implementing as: {proposed change}
Reason: {why the change is needed}
Continue? (yes -> record deviation, continue / no -> implement per original spec)
```

**c. If user approves (yes):**
- Append to `## Deviations` section in spec.md:

```markdown
### DEV-NNN: {Brief description}
- **Spec step:** e2e-{feature}-{NNN}
- **Original:** {what the spec said}
- **Actual:** {what was implemented instead}
- **Reason:** {why the deviation was necessary}
- **Approved:** yes
```

**d. If user rejects (no):**
- Implement per original spec
- No deviation recorded

**e. DEV numbering:**
- Sequential within the feature: DEV-001, DEV-002, DEV-003, etc.
- Check existing deviations in the `## Deviations` section to determine the next number

**f. Return to Step 2** and continue the implementation loop.

**When NOT to record deviations:**
- Minor wording changes
- CSS/styling details not specified in the spec
- Implementation details not covered by the spec (e.g., variable names, internal function structure)

## Step 4: ADR Detection

**Purpose:** Detect moments where architectural decisions are being made and suggest ADR creation.

Two active triggers:

### Trigger 1: Deviation Trigger

When a deviation is recorded in Step 3 AND the deviation involves choosing a different technology, pattern, or approach (not minor field/path changes):

- **Qualifies:** Switching database (PostgreSQL -> MongoDB), choosing a different auth pattern, changing API architecture
- **Does NOT qualify:** Changing an endpoint path, renaming a field, adjusting error messages

### Trigger 2: Tradeoff Trigger

When the conversation contains tradeoff language:

- "A instead of B" / "A 대신 B"
- "chose X over Y" / "X로 결정"
- "tradeoff" / "trade-off" / "트레이드오프"
- "intentional tech debt" / "나중에 바꿔야"
- Comparing 2+ options with pros/cons

### ADR Suggestion Flow

**a. Present suggestion:**
```
이 결정은 ADR로 기록할 가치가 있습니다. ADR을 생성할까요?
```

**b. If user agrees:** Delegate to ADR skill's 6-step workflow. Do NOT hand-roll the ADR format -- the ADR skill handles:
1. Auto-detect next number
2. Extract decision from context
3. Write ADR document
4. Insert WHY+SEE comments in code
5. Update CLAUDE.md
6. Confirm

**c. After ADR created:** Insert WHY+SEE comments in code at the implementation site. When a step also has a SPEC comment, the comment order is:

```
// SPEC: e2e-{feature}-{NNN} -- {step title}
// WHY: {one-line decision rationale}
// SEE: docs/decisions/{NNN}-{slug}.md
```

`// SPEC:` always comes first (structural link), then `// WHY:` + `// SEE:` (decision context).

### ADR Threshold

A decision must meet BOTH criteria to qualify for an ADR:
1. Choosing between 2+ viable approaches
2. Having lasting consequences

Routine implementation choices (which loop construct, variable naming, minor refactors) do NOT qualify. Over-suggesting ADRs causes suggestion fatigue.

## Step 5: Completion

**Purpose:** Register outputs, update state, and announce readiness for TEST stage.

**Trigger:** All spec steps have `**Status:** verified, implemented, or implemented (override). No step has `**Status:** failed or pending.

**Actions:**

**a. Register outputs in manifest.json** under `artifacts.2_do.outputs`:

```json
[
  {
    "type": "code",
    "path": "{list of changed files}",
    "created_at": "{ISO-8601}",
    "status": "complete"
  },
  {
    "type": "spec-updated",
    "path": ".lifecycle/features/{feature-name}/spec.md",
    "created_at": "{ISO-8601}",
    "status": "complete"
  }
]
```

Include any ADR documents created as additional output entries with `"type": "adr"`.

**b. Set completion timestamp:**
- `artifacts.2_do.completed_at` = current ISO 8601 timestamp

**c. Update state.json:**
- `current.status` = `"completed"`
- `session.last_active` = current ISO 8601 timestamp
- `session.resume_hint` = `"DO complete. Ready for TEST stage."`

**d. Display final checklist** showing all steps as `[x]`, verification status, deviation count, ADR count, and mini-verify summary:

```
DO Stage -- {feature-name} -- COMPLETE
=======================================
Spec: .lifecycle/features/{feature-name}/spec.md
Status: implementing (M/M steps done)

Checklist:
  [x] e2e-{feature}-001: {title} -- verified (attempt 1)
  [x] e2e-{feature}-002: {title} -- verified (attempt 2)
  [x] e2e-{feature}-003: {title} -- implemented (override: build timeout)
  ...
Deviations: N recorded
ADRs created: N
Mini-verify: {passed}/{total} steps verified, {retries} total retries
```

**e. Announce:** "DO complete. Ready for TEST stage."

**f. Update memory files:**

1. If any settings/config files were changed during DO (`.env*`, `*.config.*`, `package.json`, `tsconfig.json`, `*.yaml`/`*.toml` config), append entries to `.lifecycle/settings-changelog.md`:
   - Format: `| {date time} | {file path} | {what changed} | {why it changed} |`
   - If file doesn't exist yet, copy from `$CLAUDE_SKILL_DIR/templates/settings-changelog.md` first
2. If any lightweight decisions were made during DO (direction changes, small tradeoffs that don't meet ADR threshold), append to `.lifecycle/decisions.md`:
   - Format: `| {date} | {decision} | {reason} |`
   - If file doesn't exist yet, copy from `$CLAUDE_SKILL_DIR/templates/decisions.md` first
3. Regenerate `.lifecycle/LIVING-STATE.md` with current state (per stage-transitions.md Step 6 procedure)

## Anti-Patterns

These are the most common mistakes. Do NOT do any of these:

| # | Anti-Pattern | Why It Is Wrong |
|---|-------------|-----------------|
| 1 | **Starting DO without approved spec** | Gate 1_to_2 exists for a reason -- implementing without a spec produces unverifiable code. |
| 2 | **Silently deviating from spec** | Defeats the traceability thread. Every substantive difference must be recorded and user-confirmed. |
| 3 | **Auto-approving deviations** | User confirmation is mandatory. The adapter must pause and wait for explicit yes/no. |
| 4 | **Skipping SPEC comments** | `// SPEC: e2e-{feature}-{NNN}` comments are the traceability thread from spec to code. Without them, TEST stage cannot verify step-by-step. |
| 5 | **Creating ADRs for every deviation** | Not all deviations are architectural decisions. Over-suggesting causes fatigue and devalues real ADRs. |
| 6 | **Working from memory instead of reading spec from disk** | Spec IDs in memory may not match the actual file. Always read `.lifecycle/features/{feature-name}/spec.md` from disk at Step 1. |
| 7 | **Retrying with the same approach** | Each retry must analyze the specific error and try a different fix. Repeating identical code wastes all 3 attempts. |
| 8 | **Mini-verify running full test suite** | Always prefer targeted commands (single test file, single module compile). Timeout is 30 seconds -- full suites will exceed this. |
| 9 | **Retry loop modifying unrelated code** | Fixes must be scoped to the current spec step's files. If the error requires changes outside the step's scope, escalate to user. |

## Relationship to Other Skills

- **dev-lifecycle:** Primary skill for Stage 2. This adapter orchestrates the DO Stage workflow.
- **PDCA:** Available as supporting skill. User can invoke `/pdca do` directly for quality cycles. dev-lifecycle does not auto-invoke PDCA during DO.
- **ADR:** Delegates to existing ADR skill for document creation (6-step workflow). DO Stage only handles detection triggers and suggestion -- ADR skill handles format, numbering, and WHY+SEE comments.
- **GSD:** Not involved in Stage 2. GSD handles project-level planning (roadmap, phases, milestones), not feature implementation.

## File Locations

| File | Path | Purpose |
|------|------|---------|
| This adapter | `$CLAUDE_SKILL_DIR/references/do-stage.md` | DO Stage workflow instructions |
| Spec format reference | `$CLAUDE_SKILL_DIR/references/e2e-spec.md` | ID convention, status lifecycle, deviation log format |
| Spec template | `$CLAUDE_SKILL_DIR/templates/spec-template.md` | Template with deviation section |
| Stage transitions | `$CLAUDE_SKILL_DIR/references/stage-transitions.md` | Gate conditions (1_to_2, 2_to_3) |
| State template | `$CLAUDE_SKILL_DIR/templates/state.json` | State schema |
| Manifest template | `$CLAUDE_SKILL_DIR/templates/manifest.json` | Artifact registry with gate rules |

**Runtime location:** `$CLAUDE_SKILL_DIR` = `~/.claude/skills/dev-lifecycle`
**Version-controlled location:** `skill/` directory in the project repository
