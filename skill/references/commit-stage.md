# COMMIT Stage Adapter

This reference defines the complete COMMIT Stage workflow -- Stage 4 of the dev-lifecycle pipeline. When Claude reads this file, it follows the instructions below to validate verification results, generate a why-centric commit message, and create a git commit for the feature.

## When This Runs

- User says "commit", "save", or similar
- `state.json` `current.stage = 4` (or transitioning from 3 to 4)
- SKILL.md Stage 4 section directs here via `Read: $CLAUDE_SKILL_DIR/references/commit-stage.md`

## Prerequisites

- `.lifecycle/` directory initialized (from PLAN stage)
- `state.json` and `manifest.json` exist
- TEST stage must have outputs -- gate 3_to_4 must pass (`artifacts.3_test.outputs.length > 0`)

## Step 0: Stage Initialization

Before any commit work, verify the pipeline state, enforce the verification gate, and validate all required artifacts.

**Actions:**

1. Read `.lifecycle/state.json` and verify:
   - `current.feature` has a value
   - `current.stage` = `4` (or transition from 3 to 4)

2. Check basic gate condition (3_to_4):
   - Read `.lifecycle/manifest.json`
   - Verify: `artifacts.3_test.outputs.length > 0`
   - If gate fails: set `current.status = "blocked"`, display missing artifacts, STOP

3. Content-level gate -- read `.lifecycle/features/{feature-name}/verification.json`:
   - Check `specVerification.overall === "passed"`
   - Check `userVerification.status === "passed"`
   - If `specVerification.overall === "failed"`: display gap list (see Step 1 blocked format), STOP
   - If `userVerification.status === "pending"`: display "User behavioral verification not completed", STOP
   - If `userVerification.status === "failed"`: display gap list with user-reported failures, STOP

4. Verify required artifacts exist on disk:
   - `.lifecycle/features/{feature-name}/spec.md`
   - `.lifecycle/features/{feature-name}/prototype.html`
   - `.lifecycle/features/{feature-name}/verification.json`
   - `.lifecycle/features/{feature-name}/verification.md`
   - If any missing: set `current.status = "blocked"`, list missing files, STOP

5. Update `state.json`:
   - `current.stage` = `4`
   - `current.stage_name` = `"COMMIT"`
   - `current.status` = `"in_progress"`
   - `progress.current_stage_started_at` = current ISO 8601 timestamp

## Step 1: Verification Gate Display

When blocked (any check in Step 0 fails), display this format:

```
COMMIT Stage -- {feature-name} -- BLOCKED
==========================================
Verification gate FAILED. Cannot commit.

Failed spec steps:
  [FAIL] e2e-{feature}-{NNN}: {title} -- {reason}
  [FAIL] e2e-{feature}-{NNN}: {title} -- {reason}

User behavioral verification: {PASSED/FAILED/PENDING}

Options:
  1. Return to DO stage to fix failures
  2. Return to TEST stage to re-verify
  3. Override and commit anyway (not recommended)
```

**If user chooses override (option 3):**

1. Display explicit warning: `"WARNING: Committing with verification failures. This will be recorded."`
2. Require user to type "override" to confirm
3. Continue to Step 2 with override flag set to true
4. Record override in commit message and manifest output

## Step 2: Commit Preparation

**Purpose:** Collect context, detect ADR references, and generate a why-centric commit message.

**Actions:**

1. Collect changed files from `manifest.json` `artifacts.2_do.outputs` (entries where `type = "code"`)
2. Detect ADR references from `manifest.json` `artifacts.2_do.outputs` (entries where `type = "adr"`)
3. Read `spec.md` for: feature name, step count, overall status
4. Read deviations count from spec.md `## Deviations` section
5. Generate why-centric commit message using this format:

```
{type}({feature-name}): {why-summary}

{1-2 sentence description of the user/business value delivered}

Spec: .lifecycle/features/{name}/spec.md
Steps: {N} verified ({M} deviations)
{If ADRs: (ADR-NNN) {adr title} -- one line per ADR}
{If override: WARNING: Committed with verification override}
```

6. Present commit message to user for approval or editing
7. Wait for user confirmation before proceeding

**Commit type selection:**

| Condition | Type |
|-----------|------|
| New feature implementation | `feat` |
| Bug fix or correction | `fix` |
| Refactoring without behavior change | `refactor` |
| Default for spec-driven work | `feat` |

## Step 3: Git Commit Execution

**Purpose:** Stage appropriate files, execute the commit, and optionally create a version tag.

**Actions:**

1. Stage files:
   - All source code changes (from DO stage)
   - `.lifecycle/features/{name}/spec.md`
   - `.lifecycle/features/{name}/prototype.html`
   - `.lifecycle/features/{name}/verification.json`
   - `.lifecycle/features/{name}/verification.md`
   - `docs/decisions/*.md` (any ADRs created during DO)

2. Exclude from staging:
   - `.env`, credentials, secrets
   - `node_modules/`, `.venv/`, etc.
   - `.lifecycle/state.json` (runtime state)
   - `.lifecycle/manifest.json` (runtime state)
   - `.lifecycle/history/` (runtime state)

3. Execute git commit with approved message
4. Capture commit hash
5. Optionally create version tag if user requests (format: `v{major}.{minor}.{patch}`)

## Step 4: Completion

**Purpose:** Register outputs, update pipeline state, and announce completion.

**Actions:**

1. Register outputs in `manifest.json` under `artifacts.4_commit.outputs`:

```json
[
  {
    "type": "commit-hash",
    "path": "{git-sha}",
    "created_at": "{ISO-8601}",
    "status": "complete"
  }
]
```

If tag created, add:
```json
{
  "type": "tag",
  "path": "{tag-name}",
  "created_at": "{ISO-8601}",
  "status": "complete"
}
```

If override was used, add `"override": true` to the commit-hash entry.

2. Set `artifacts.4_commit.completed_at` = current ISO 8601 timestamp

3. Update `state.json`:
   - `current.status` = `"completed"`
   - `session.last_active` = current ISO 8601 timestamp
   - `session.resume_hint` = `"COMMIT complete. Ready for DEPLOY stage."`

4. Write history entry to `.lifecycle/history/` (per stage-transitions.md format):

```json
{
  "timestamp": "{ISO-8601}",
  "from_stage": 3,
  "to_stage": 4,
  "gate_result": "passed",
  "missing_artifacts": [],
  "mode": "{current mode}"
}
```

5. Announce completion:

```
COMMIT Stage -- {feature-name} -- COMPLETE
==========================================
Commit: {short-sha} {first line of message}
{If tag: Tag: {tag-name}}
{If override: NOTE: Committed with verification override}

Inner loop complete. Ready for outer loop (DEPLOY) or next feature.
```

## Anti-Patterns

These are the most common mistakes. Do NOT do any of these:

| # | Anti-Pattern | Why It Is Wrong |
|---|-------------|-----------------|
| 1 | **Auto-committing without user confirmation** | COMMIT must present the commit message and get user approval before executing git commit. User may want to edit the message or abort. |
| 2 | **Skipping verification.json content check** | The basic `outputs.length > 0` gate is insufficient. COMMIT must read verification.json and check `specVerification.overall` and `userVerification.status`. A file existing does not mean verification passed. |
| 3 | **Including sensitive files in commit** | Must exclude `.env`, credentials, `.lifecycle/state.json`, `.lifecycle/manifest.json`, and `.lifecycle/history/`. These are runtime state or secrets, not feature artifacts. |
| 4 | **Committing when userVerification is "pending"** | Dual gate requires BOTH Claude structural verification AND user behavioral verification to pass. "Pending" means the user has not yet performed their checks. |
| 5 | **What-centric commit messages** | Focus on WHY (business value delivered), not WHAT (files changed). Let `git diff` handle the "what". Commit messages should answer "why does this change exist?" |
| 6 | **Forgetting to register commit outputs in manifest** | The commit hash must be registered in `artifacts.4_commit.outputs`. Without this, gate 4_to_5 for DEPLOY stage will fail. |

## Relationship to Other Skills

- **dev-lifecycle:** Primary skill for Stage 4. This adapter orchestrates the COMMIT Stage workflow, enforces the verification gate, and manages artifact registration.
- **Git:** Direct tool for commit execution (per role-matrix). Claude uses git commands to stage files, create commits, and optionally create tags.
- **ADR:** Referenced in commit messages when ADRs exist from DO stage. ADR documents are included in the commit but ADR skill is not invoked during COMMIT.
- **GSD:** Not directly involved in Stage 4. GSD handles project-level planning, not feature commits.
- **PDCA:** Not directly involved in Stage 4. PDCA analysis may inform future cycles but does not participate in the commit flow.

## File Locations

| File | Path | Purpose |
|------|------|---------|
| This adapter | `$CLAUDE_SKILL_DIR/references/commit-stage.md` | COMMIT Stage workflow instructions |
| Test stage adapter | `$CLAUDE_SKILL_DIR/references/test-stage.md` | verification.json schema, dual gate pattern |
| Stage transitions | `$CLAUDE_SKILL_DIR/references/stage-transitions.md` | Gate conditions (3_to_4, 4_to_5) |
| State template | `$CLAUDE_SKILL_DIR/templates/state.json` | State schema with stage/status fields |
| Manifest template | `$CLAUDE_SKILL_DIR/templates/manifest.json` | Artifact registry with gate rules |
| Verification input | `.lifecycle/features/{name}/verification.json` | Machine-parseable TEST results (consumed by Step 0) |
| Verification report | `.lifecycle/features/{name}/verification.md` | Human-readable TEST report (committed as artifact) |

**Runtime location:** `$CLAUDE_SKILL_DIR` = `~/.claude/skills/dev-lifecycle`
**Version-controlled location:** `skill/` directory in the project repository
