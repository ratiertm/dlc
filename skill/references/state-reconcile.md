# State Reconcile Procedure

Recovers `state.json` from `.lifecycle/` filesystem artifacts when the state file is missing or corrupted.

## When Triggered

- `.lifecycle/` directory exists BUT `.lifecycle/state.json` is missing or unparseable
- Detected during Session Start step 1 (see SKILL.md)

## Algorithm

### Step 1: Check manifest.json

Read `.lifecycle/manifest.json`.

- **If missing:** The lifecycle directory has no meaningful state. Initialize fresh from `$CLAUDE_SKILL_DIR/templates/state.json` and `$CLAUDE_SKILL_DIR/templates/manifest.json`. STOP here (no further reconcile needed).
- **If exists:** Proceed to Step 2.

### Step 2: Scan for highest completed stage

Scan manifest.json artifacts from stage 9 down to stage 1. Find the highest completed stage:

```
For stage_key in [9_promote, 8_retrospect, 7_document, 6_deploy_test, 5_deploy, 4_commit, 3_test, 2_do, 1_plan]:
  If artifacts[stage_key].outputs.length > 0 AND artifacts[stage_key].completed_at != null:
    last_completed = stage_number
    BREAK
```

### Step 3: Determine current stage

- **If `last_completed` found:** `current_stage = last_completed + 1`
- **If `last_completed` is 9:** Cycle complete. Reset to stage 1 (new feature cycle).
- **If no completed stages found:** `current_stage = 1`

### Step 4: Infer execution mode

Check `.lifecycle/history/` for skip entries. Match skip patterns to known modes:

| Skipped Stages | Inferred Mode |
|----------------|---------------|
| 1, 7, 8, 9 | `hotfix` |
| 5, 6, 9 | `feature` |
| 9 only | `release` |
| None | `milestone` |

- If `.lifecycle/history/` is missing or empty, or skip pattern does not match any known mode: default to `"feature"`.

### Step 5: Reconstruct state.json

Build state.json with conservative defaults:

```json
{
  "version": "2.0",
  "project": {
    "name": "<from manifest.json feature field, or parent directory name>",
    "type": "auto",
    "detected_type": null,
    "confirmed": false
  },
  "current": {
    "stage": "<from Step 3>",
    "stage_name": "<stage name lookup>",
    "status": "not_started",
    "feature": "<from manifest.json feature field or null>",
    "mode": "<from Step 4>",
    "completeness": null
  },
  "progress": {
    "stages_completed": "<list of completed stage numbers from manifest scan>",
    "current_stage_started_at": null,
    "last_transition_at": "<latest completed_at from manifest, or null>"
  },
  "session": {
    "last_active": null,
    "resume_hint": "State recovered via reconcile. Verify current position."
  }
}
```

Key: Use `"not_started"` status (never `"in_progress"`) to avoid assuming work is underway.

### Step 6: Write and log

1. Write the reconstructed `state.json` to `.lifecycle/state.json`
2. Log the reconcile event to `.lifecycle/history/` with entry:

```json
{
  "timestamp": "ISO-8601",
  "event": "reconcile",
  "inferred_stage": "<current_stage>",
  "inferred_mode": "<mode>",
  "source": "manifest.json",
  "last_completed_stage": "<last_completed or null>"
}
```

## Post-Reconcile

After reconcile completes, display the result to the user:

```
State recovered via reconcile:
  Stage: {N} {NAME} -- not_started
  Mode: {mode}
  Last completed: Stage {last_completed} ({name})

Verify this is correct. If wrong, tell me the correct stage and I'll update.
```

The user can correct the stage or mode if the inference was wrong.

## Edge Cases

| Scenario | Behavior |
|----------|----------|
| Empty manifest (no outputs anywhere) | Treat as fresh start: stage 1, mode "feature" |
| Partial history (some entries missing) | Use manifest as primary source; history is secondary for mode inference |
| No `.lifecycle/history/` directory | Skip mode inference from history, default to "feature" |
| manifest.json has outputs but no `completed_at` | Stage is NOT considered completed (in-progress work). Move to that stage. |
| All 9 stages completed | Reset to stage 1 for new feature cycle |

## Stage Name Lookup

| Stage | Name |
|-------|------|
| 1 | PLAN |
| 2 | DO |
| 3 | TEST |
| 4 | COMMIT |
| 5 | DEPLOY |
| 6 | DEPLOY TEST |
| 7 | DOCUMENT |
| 8 | RETROSPECT |
| 9 | PROMOTE |
