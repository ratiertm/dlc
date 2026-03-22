# Stage Transition Rules

This document defines how dev-lifecycle moves between stages, what conditions must be met (gates), and which stages can be skipped based on execution mode.

## Status Values

Each stage has one of these statuses:

| Status | Meaning |
|--------|---------|
| `not_started` | Stage has not begun |
| `in_progress` | Stage work is actively being done |
| `completed` | Stage gate conditions met, all required outputs registered |
| `blocked` | Gate check failed -- missing required artifacts |
| `skipped` | Stage intentionally skipped (allowed by current mode) |

## Transition Gates

Each transition from one stage to the next requires a gate condition to pass. Gates verify that the current stage produced the required outputs before allowing advancement.

| From | To | Gate Condition |
|------|----|----------------|
| 1 PLAN | 2 DO | `artifacts.1_plan.outputs.length > 0 && artifacts.1_plan.completed_at != null` |
| 2 DO | 3 TEST | `artifacts.2_do.outputs.length > 0` |
| 3 TEST | 4 COMMIT | `artifacts.3_test.outputs.length > 0` |
| 4 COMMIT | 5 DEPLOY | `artifacts.4_commit.outputs.length > 0` |
| 5 DEPLOY | 6 DEPLOY TEST | `artifacts.5_deploy.outputs.length > 0` |
| 6 DEPLOY TEST | 7 DOCUMENT | `artifacts.6_deploy_test.outputs.length > 0` |
| 7 DOCUMENT | 8 RETROSPECT | `artifacts.7_document.outputs.length > 0` |
| 8 RETROSPECT | 9 PROMOTE | `artifacts.8_retrospect.outputs.length > 0` |

Note: Gate conditions in Phase 1 use simple existence checks ("does the stage have at least one output?"). More sophisticated verification (e.g., checking spec step pass/fail status, prototype coverage) will be implemented in later phases when the E2E spec system is built.

## Mode-Based Stage Skipping

Different execution modes allow different stages to be skipped:

| Mode | Skippable Stages | Use Case |
|------|-----------------|----------|
| `hotfix` | 1 (PLAN), 7 (DOCUMENT), 8 (RETROSPECT), 9 (PROMOTE) | Urgent bug fix -- skip planning and post-deploy documentation |
| `feature` | 5 (DEPLOY), 6 (DEPLOY TEST), 9 (PROMOTE) | Feature development -- skip deployment and promotion |
| `release` | 9 (PROMOTE) | Full release -- only promotion is optional |
| `milestone` | None | Complete lifecycle -- all stages required |

When a stage is skipped:
1. Its status is set to `skipped` in state.json
2. The gate condition for the next stage is automatically satisfied
3. The skip is recorded in the transition history

## Transition Procedure

When transitioning from Stage N to Stage N+1:

1. **Read state.json** -- get current stage number and status
2. **Check gate condition** -- evaluate the gate rule for current -> next transition against manifest.json
3. **If gate passes:**
   - Update state.json: `current.stage = N+1`, `current.stage_name = <next stage name>`, `current.status = "not_started"`
   - Update `progress.last_transition_at` with current ISO 8601 timestamp
   - Add N to `progress.stages_completed` array
4. **If gate fails:**
   - Set `current.status = "blocked"`
   - Display missing artifacts to user:
     ```
     Stage N gate check FAILED.
     Missing artifacts for stage N:
     - [list of missing required outputs]
     Complete these before transitioning.
     ```
5. **Write history entry** to `.lifecycle/history/` as a timestamped JSON file:
   ```json
   {
     "timestamp": "ISO-8601",
     "from_stage": N,
     "to_stage": N+1,
     "gate_result": "passed|failed",
     "missing_artifacts": [],
     "mode": "feature|hotfix|release|milestone"
   }
   ```
6. **Update Living State** -- After successful transition, regenerate `.lifecycle/LIVING-STATE.md`:
   - Read current `state.json` for stage/status/feature/mode
   - Read last 20 entries from `.lifecycle/settings-changelog.md` (if exists)
   - Read last 10 entries from `.lifecycle/decisions.md` (if exists)
   - Read `.lifecycle/history/` recent entries for Event Timeline
   - Read `manifest.json` for stage completion timestamps
   - List ADRs from `docs/decisions/` (if directory exists)
   - Read `state.json` `session.resume_hint` for Resume Hint section
   - Write complete `.lifecycle/LIVING-STATE.md` using template structure from `$CLAUDE_SKILL_DIR/templates/living-state.md`
   - Target: ~100 lines max. Truncate older changelog/decision entries if needed.

## Backward Transitions

Stages can move backward (e.g., TEST -> DO when tests fail). Backward transitions:
- Do NOT require gate checks
- Reset the target stage status to `in_progress`
- Remove the target stage from `stages_completed` if present
- Record the backward transition in history with reason

## Artifact Output Entry Format

When an artifact is registered in manifest.json, each output entry has this shape:

```json
{
  "type": "plan|spec|prototype|code|test|commit|deploy|report|retro|demo",
  "path": "relative/path/to/artifact",
  "created_at": "ISO-8601",
  "status": "complete|partial|failed"
}
```
