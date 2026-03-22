# DEPLOY TEST Stage Adapter

This reference defines the complete DEPLOY TEST Stage workflow -- Stage 6 of the dev-lifecycle pipeline. When Claude reads this file, it follows the instructions below to guide the user through smoke testing a deployed feature across three verification categories: health, core flow, and resources.

## When This Runs

- User says "smoke test", "deploy test", "verify deployment", or similar
- `state.json` `current.stage = 6` (or transitioning from 5 to 6)
- SKILL.md Stage 6 section directs here via `Read: $CLAUDE_SKILL_DIR/references/deploy-test-stage.md`

## Prerequisites

- `.lifecycle/` directory initialized (from PLAN stage)
- `state.json` and `manifest.json` exist
- DEPLOY stage must have outputs -- gate 5_to_6 must pass (`artifacts.5_deploy.outputs.length > 0`)

## Step 0: Stage Initialization

Before any smoke testing, verify the pipeline state and check mode-based skipping.

**Actions:**

1. Read `.lifecycle/state.json` and verify:
   - `current.feature` has a value
   - `current.stage` = `6` (or transition from 5 to 6)

2. Check gate condition (5_to_6):
   - Read `.lifecycle/manifest.json`
   - Verify: `artifacts.5_deploy.outputs.length > 0`
   - If gate fails: set `current.status = "blocked"`, display missing artifacts, STOP

3. Check mode-based skip:
   - Read `current.mode` from `state.json`
   - If mode is `feature`: offer skip -- "Feature mode allows skipping DEPLOY TEST. Skip? (yes/no)"
   - If user chooses skip: register `{ "type": "skipped", "path": "N/A", "status": "skipped" }` in `artifacts.6_deploy_test.outputs`, set `artifacts.6_deploy_test.completed_at`, set `current.status = "skipped"`, announce skip, STOP

4. Check if DEPLOY was skipped:
   - If `artifacts.5_deploy.outputs[0].type === "skipped"`: DEPLOY TEST should also skip. Register skipped artifact and STOP.

5. Update `state.json`:
   - `current.stage` = `6`
   - `current.stage_name` = `"DEPLOY TEST"`
   - `current.status` = `"in_progress"`
   - `progress.current_stage_started_at` = current ISO 8601 timestamp

## Step 1: Smoke Test Plan

**Purpose:** Generate a structured smoke test plan based on the deployed feature.

**Actions:**

1. Read deploy details from `artifacts.5_deploy.outputs`:
   - Deploy URL or artifact path
   - Platform/environment
2. Read feature spec from `.lifecycle/features/{feature-name}/spec.md` to identify:
   - Core user flows
   - Key endpoints or screens
   - Auth requirements (if any)
3. Generate a 3-category smoke test checklist:

```
DEPLOY TEST -- {feature-name} -- Smoke Test Plan
=================================================
Target: {deploy-url}
Platform: {platform}

[ ] HEALTH (3 checks)
  [ ] H1: Main endpoint returns 2xx
  [ ] H2: Response time < 5s
  [ ] H3: No error messages in response body

[ ] CORE FLOW ({N} checks)
  [ ] C1: Entry point reachable
  [ ] C2: {feature-specific check based on spec}
  [ ] C3: {feature-specific check based on spec}

[ ] RESOURCES (3 checks)
  [ ] R1: No OOM or crash errors in logs
  [ ] R2: Disk/storage within limits
  [ ] R3: Database/external connections available
```

4. Present plan to user for review before executing

## Step 2: Health Check

**Purpose:** Verify the deployment is alive and responding.

**Actions:**

1. **Main endpoint 2xx:** Check that the primary URL/endpoint returns a successful HTTP status code (200-299)
   - For web apps: main page loads
   - For APIs: health endpoint (`/health`, `/`, or `/api`) returns 2xx
   - For mobile: app launches without crash
2. **Response time < 5s:** Verify the response is received within 5 seconds
   - If over 5s but under 30s: mark as WARNING, note performance concern
   - If over 30s or timeout: mark as FAIL
3. **No error in body:** Check response body for error indicators
   - No "500", "Internal Server Error", stack traces, or error JSON in response
   - For web: page renders without visible error messages

Record result for each check: PASS / FAIL / WARNING

## Step 3: Core Flow Verification

**Purpose:** Verify the deployed feature's primary user flow works end-to-end.

**Actions:**

1. **Entry point reachable:** Navigate to the feature's entry point (page, screen, API endpoint)
2. **Auth works (if applicable):** If the feature requires authentication, verify login flow succeeds
3. **Primary feature works:** Execute the main user flow described in the spec
   - For CRUD features: verify create, read, update operations
   - For display features: verify data renders correctly
   - For API features: verify request/response matches spec
4. **Data persists:** If the feature writes data, verify it persists after creation
   - Refresh page / re-query endpoint to confirm data was saved

Record result for each check: PASS / FAIL / SKIP (if not applicable)

## Step 4: Resource Check

**Purpose:** Verify the deployment has adequate resources and no system-level issues.

**Actions:**

1. **No OOM errors:** Check application logs for out-of-memory errors or crashes
   - If log access is not available: ask user to check platform dashboard
2. **Disk OK:** Verify disk/storage usage is within acceptable limits
   - For cloud platforms: check dashboard for storage alerts
   - For self-hosted: verify disk usage < 80%
3. **DB connections available:** Verify database or external service connections are working
   - For API apps: an endpoint that queries DB returns data
   - For serverless: cold start + warm request both succeed
4. **No rate limiting:** Verify the app is not being rate-limited by external services
   - Multiple sequential requests succeed without 429 errors

Record result for each check: PASS / FAIL / SKIP (if not applicable) / N/A (no external deps)

## Step 5: Completion

**Purpose:** Compile test results, register outputs, and announce completion.

**Actions:**

1. Compile smoke test report:

```markdown
# Smoke Test Report -- {feature-name}

**Date:** {ISO-8601}
**Target:** {deploy-url}
**Result:** {PASS / FAIL / PARTIAL}

## Health Check
- H1 Main endpoint 2xx: {PASS/FAIL}
- H2 Response time < 5s: {PASS/FAIL/WARNING}
- H3 No errors in body: {PASS/FAIL}

## Core Flow
- C1 Entry point reachable: {PASS/FAIL}
- C2 {check}: {PASS/FAIL/SKIP}
- C3 {check}: {PASS/FAIL/SKIP}

## Resources
- R1 No OOM/crash errors: {PASS/FAIL/SKIP}
- R2 Disk within limits: {PASS/FAIL/SKIP/N/A}
- R3 DB connections OK: {PASS/FAIL/SKIP/N/A}

## Summary
{N}/{M} checks passed. {Any notes on failures or warnings.}
```

2. Save report to `.lifecycle/features/{feature-name}/smoke-test-report.md`

3. Register outputs in `manifest.json` under `artifacts.6_deploy_test.outputs`:

```json
[
  {
    "type": "smoke-test-report",
    "path": ".lifecycle/features/{feature-name}/smoke-test-report.md",
    "created_at": "{ISO-8601}",
    "status": "complete"
  }
]
```

4. Set `artifacts.6_deploy_test.completed_at` = current ISO 8601 timestamp

5. Update `state.json`:
   - `current.status` = `"completed"`
   - `session.last_active` = current ISO 8601 timestamp
   - `session.resume_hint` = `"DEPLOY TEST complete. Ready for DOCUMENT stage."`

6. Write history entry to `.lifecycle/history/`:

```json
{
  "timestamp": "{ISO-8601}",
  "from_stage": 5,
  "to_stage": 6,
  "gate_result": "passed",
  "missing_artifacts": [],
  "mode": "{current mode}"
}
```

7. Regenerate `.lifecycle/LIVING-STATE.md` with current state (per stage-transitions.md Step 6 procedure)

8. Announce completion:

```
DEPLOY TEST Stage -- {feature-name} -- COMPLETE
================================================
Result: {PASS/FAIL/PARTIAL}
Checks: {N}/{M} passed
Report: .lifecycle/features/{feature-name}/smoke-test-report.md

Ready for DOCUMENT stage.
```

## Anti-Patterns

These are the most common mistakes. Do NOT do any of these:

| # | Anti-Pattern | Why It Is Wrong |
|---|-------------|-----------------|
| 1 | **Skipping smoke tests because "it worked locally"** | Local and production environments differ. Smoke tests catch environment-specific issues (missing env vars, wrong DB, network policies, etc.). |
| 2 | **Testing only health without core flow** | A 200 response does not mean the feature works. Core flow verification catches logic errors, missing data, broken auth, and rendering issues. |
| 3 | **Not recording test results** | The smoke test report is a required artifact for gate 6_to_7. Without it, DOCUMENT stage cannot proceed. Results also provide historical deploy quality data. |
| 4 | **Auto-passing resource checks without verification** | Resource issues (OOM, disk full, DB connection pool exhaustion) cause production outages. Check logs or dashboards, do not assume resources are fine. |
| 5 | **Proceeding when health check fails** | If the deployment is not responding, core flow and resource checks are meaningless. Health must pass before proceeding to other categories. |

## Relationship to Other Skills

- **dev-lifecycle:** Primary skill for Stage 6. This adapter orchestrates the DEPLOY TEST Stage workflow, generates smoke test plans, and compiles results.
- **Deploy stage adapter:** Provides deploy URL and platform information consumed by smoke tests.
- **Git:** Not directly involved in Stage 6. Smoke test reports are tracked in manifest but not committed until COMMIT-equivalent in the outer loop.
- **GSD:** Not directly involved in Stage 6.

## File Locations

| File | Path | Purpose |
|------|------|---------|
| This adapter | `$CLAUDE_SKILL_DIR/references/deploy-test-stage.md` | DEPLOY TEST Stage workflow instructions |
| Deploy stage adapter | `$CLAUDE_SKILL_DIR/references/deploy-stage.md` | Previous stage (gate 5_to_6 source, deploy URL) |
| Document stage adapter | `$CLAUDE_SKILL_DIR/references/document-stage.md` | Next stage (consumes smoke test results) |
| Stage transitions | `$CLAUDE_SKILL_DIR/references/stage-transitions.md` | Gate conditions (5_to_6, 6_to_7) |
| State template | `$CLAUDE_SKILL_DIR/templates/state.json` | State schema with stage/status fields |
| Manifest template | `$CLAUDE_SKILL_DIR/templates/manifest.json` | Artifact registry with gate rules |

**Runtime location:** `$CLAUDE_SKILL_DIR` = `~/.claude/skills/dev-lifecycle`
**Version-controlled location:** `skill/` directory in the project repository
