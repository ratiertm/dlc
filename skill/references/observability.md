# Observability & Analytics

This reference defines all observability features: session context tracking, stage transition analytics, rework event detection, spec baseline diffing, and time-per-stage metrics.

Data files live in `.lifecycle/sessions/` and `.lifecycle/analytics/`. All analytics are append-only and non-blocking (analytics write failure never blocks stage transitions).

## Session Context Files

Session context files capture what happened during each dev-lifecycle session -- decisions made, artifacts touched, and stage transitions. They are the "session diary" for human review.

### Creation (at Session Start step 6)

1. Directory: `.lifecycle/sessions/` (create if not exists)
2. Filename: `{ISO-date}T{HH-MM-SS}Z-{feature}.md` (e.g., `2026-03-25T14-00-00Z-user-auth.md`)
3. Initial content:

```markdown
# Session: {ISO-timestamp}

**Feature:** {feature-name}
**Stage at start:** {N} {NAME}
**Mode:** {mode}
**Started:** {ISO-timestamp}

## Decisions Made
(none yet)

## Artifacts Touched
(none yet)

## Stage Transitions
(none yet)

## Notes
{resume_hint from state.json, or "New session"}
```

### Update (at stage transitions and session end)

- Append new decisions (delta from `.lifecycle/decisions.md` -- compare line count at session start vs now)
- Append artifacts touched (new entries in manifest.json since session start)
- Append stage transitions that occurred
- Update "Stage at end" field
- Add computed duration

### Constraints

- Cap session file at ~50 lines. If exceeding, summarize older entries.
- Only record deltas (new items since session start), not full state.

## Stage Transition Analytics

JSONL append logic that runs alongside existing history writes. This provides queryable transition data without changing the existing `.lifecycle/history/` format.

### When

After Transition Procedure Step 5 (write history entry), as Step 5b.

### Directory

`.lifecycle/analytics/` (create if not exists)

### File

`.lifecycle/analytics/stage-transitions.jsonl`

### Format (one JSON line per transition)

```json
{"timestamp":"ISO-8601","feature":"{name}","from_stage":{N},"from_name":"{NAME}","to_stage":{M},"to_name":"{NAME}","direction":"{forward|backward|skip}","mode":"{mode}","gate_result":"{passed|failed|skipped}","duration_in_stage_seconds":{seconds}}
```

### Field definitions

- `timestamp`: ISO 8601 at time of transition
- `feature`: feature name from `state.json` `current.feature`
- `from_stage`/`from_name`: source stage number and name
- `to_stage`/`to_name`: target stage number and name
- `direction`: "forward" if to_stage > from_stage, "backward" if to_stage < from_stage, "skip" if auto-skipped
- `mode`: execution mode at time of transition
- `gate_result`: "passed", "failed", or "skipped"
- `duration_in_stage_seconds`: current timestamp minus `progress.current_stage_started_at` from state.json, converted to integer seconds

### Non-blocking

If JSONL write fails, log warning and continue transition normally. Analytics must never delay or block stage transitions.

## Rework Event Tracking

Backward transition detection and logging. Rework events are a subset of stage transitions where `to_stage < from_stage`.

### When

On any backward stage transition (to_stage < from_stage).

### File

`.lifecycle/analytics/rework-events.jsonl`

### Format

```json
{"timestamp":"ISO-8601","feature":"{name}","from_stage":{N},"from_name":"{NAME}","to_stage":{M},"to_name":"{NAME}","reason":"{why backward}","impact":"{what needs redo}","rework_count":{running_total}}
```

### Field definitions

- `timestamp`, `feature`, `from_stage`, `from_name`, `to_stage`, `to_name`: same as stage-transitions.jsonl
- `reason`: from the backward transition trigger (e.g., "3 spec steps failed verification", "user requested re-plan")
- `impact`: what work is affected (e.g., "re-implement steps 002, 004")
- `rework_count`: count existing entries in rework-events.jsonl for this feature, add 1

### Dual-write

Backward transitions write to BOTH files:
1. `stage-transitions.jsonl` with `direction: "backward"`
2. `rework-events.jsonl` with `reason`, `impact`, and `rework_count`

## Spec Baseline & Diff

Baseline creation preserves the approved spec snapshot. On-demand diff compares baseline against current spec to show all changes that occurred during implementation.

### Baseline creation (at PLAN completion, before DO transition)

1. Directory: `.lifecycle/analytics/spec-baselines/` (create if not exists)
2. File: `.lifecycle/analytics/spec-baselines/{feature}.baseline.md`
3. Action: Copy `.lifecycle/features/{feature}/spec.md` to baseline path
4. Guard: Only create if baseline does NOT already exist for this feature (preserve original)
5. If user explicitly wants a reset: use `{feature}.baseline.v{N}.md` versioned naming

### On-demand diff (when user asks "spec diff" or "what changed")

1. Read baseline: `.lifecycle/analytics/spec-baselines/{feature}.baseline.md`
2. Read current: `.lifecycle/features/{feature}/spec.md`
3. Compare section by section:
   - Status changes (pending -> implemented -> verified)
   - Deviations added
   - Steps added or removed
4. Present structured diff report:

```markdown
# Spec Diff: {feature}

**Baseline:** {date}
**Current:** {date}

## Status Changes
| Step | Baseline | Current |
|------|----------|---------|
| e2e-{feature}-001 | pending | verified (attempt 1) |

## Deviations from Baseline
- DEV-001: {description} (approved)

## Summary
- Steps: {total} total, {verified} verified, {implemented} implemented
- Deviations: {count}
- Completeness delta: baseline N/A -> current {score}/10
```

## Time-Per-Stage Metrics

Time computation and on-demand reporting from stage transition data.

### Data source

`stage-transitions.jsonl` `duration_in_stage_seconds` field (computed at each transition).

### On-demand report (when user asks "time per stage" or "metrics" or "analytics")

1. Read `.lifecycle/analytics/stage-transitions.jsonl`
2. Filter by feature (or show all features if requested)
3. Compute per-stage durations
4. Identify bottleneck (stage with longest duration)
5. Present report:

```markdown
# Time-Per-Stage: {feature}

| Stage | Name | Duration | % of Total |
|-------|------|----------|------------|
| 1 | PLAN | 1h 0m | 25% |
| 2 | DO | 2h 30m | 62% |
| 3 | TEST | 30m | 13% |

**Bottleneck:** Stage 2 (DO) -- 2h 30m (62% of total)
**Total cycle time:** 4h 0m
**Rework overhead:** 30m (1 rework event)
```

### Rework overhead

Sum durations of stages entered via backward transitions (`direction: "backward"` in JSONL).

## Analytics Commands

Trigger keywords and corresponding actions for on-demand analytics:

| User Says | Action |
|-----------|--------|
| "analytics" / "metrics" / "time per stage" | Generate time-per-stage report for current feature |
| "spec diff" / "what changed from spec" | Generate spec baseline diff for current feature |
| "session history" / "sessions" | List session files, summarize recent sessions |
| "rework report" / "rework events" | Summarize rework-events.jsonl for current feature |
| "stage transitions" / "transition history" | Summarize stage-transitions.jsonl for current feature |

## Directory Structure

```
.lifecycle/
  sessions/                          # Session context files (OBSV-01)
    {ISO-date}T{HH-MM-SS}Z-{feature}.md
  analytics/                         # Structured analytics (OBSV-02/03/05)
    stage-transitions.jsonl
    rework-events.jsonl
    spec-baselines/                  # Spec snapshots (OBSV-04)
      {feature}.baseline.md
```

Note: directories are created on first write (not eagerly at project init).

## Error Handling

- **JSONL reads:** skip malformed lines (each line is independent)
- **Missing analytics directory:** create on first write
- **Missing session file:** create fresh (mid-session recovery)
- **Analytics write failure:** warn user, continue normal operation (non-blocking)
- **Empty JSONL file:** report "No data yet" (not an error)
