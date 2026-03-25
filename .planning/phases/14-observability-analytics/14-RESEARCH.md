# Phase 14: Observability & Analytics - Research

**Researched:** 2026-03-25
**Domain:** Session context tracking, stage transition analytics, rework detection, spec-vs-implementation diffing for Claude Code skill
**Confidence:** HIGH

## Summary

Phase 14 adds observability to the dev-lifecycle skill -- enabling users to see how their development process unfolds across sessions, stages, and features. The core challenge is that dev-lifecycle already tracks some data (`.lifecycle/history/` transition logs, `manifest.json` timestamps, `state.json` progress) but this data is scattered and not queryable. Phase 14 consolidates this into structured analytics files and adds two new data streams: session context files and spec-vs-implementation diffs.

The architecture follows the established dev-lifecycle pattern: a new reference file (`references/observability.md`) contains all logic, SKILL.md gets minimal additions (Read directives and brief section), and new data lands in `.lifecycle/sessions/` and `.lifecycle/analytics/`. No external libraries are needed -- this is file-based analytics that Claude reads and summarizes on demand.

The key insight is that most of the raw data already exists. `.lifecycle/history/` has stage transitions with timestamps. `manifest.json` has stage completion timestamps. `state.json` has the current stage started_at. Phase 14 transforms this existing data into structured JSONL files and adds the missing pieces: session context capture, rework event detection, and spec baseline snapshots.

**Primary recommendation:** Create `skill/references/observability.md` containing all analytics logic (session context creation, JSONL append patterns, rework detection, spec diff snapshots, time-per-stage calculation). Add a brief Observability section to SKILL.md with Read directives. Data files go in `.lifecycle/sessions/` and `.lifecycle/analytics/`.

<phase_requirements>
## Phase Requirements

| ID | Description | Research Support |
|----|-------------|-----------------|
| OBSV-01 | Each session creates a context file in .lifecycle/sessions/ with decisions and artifacts touched | Session context file created at session start, updated at session end; captures decisions from decisions.md delta and artifacts from manifest.json delta |
| OBSV-02 | Stage transitions logged to .lifecycle/analytics/stage-transitions.jsonl with timestamps | JSONL append on every forward transition; data already partially exists in .lifecycle/history/ -- analytics extracts and normalizes into queryable format |
| OBSV-03 | Rework events (backward transitions) tracked in .lifecycle/analytics/rework-events.jsonl | Backward transition detection already exists in stage-transitions.md; add JSONL append with reason and impact fields |
| OBSV-04 | Before/after snapshot diff comparing spec baseline against implementation delta | Spec baseline snapshot at PLAN completion; diff generated on demand by comparing baseline spec against current spec (with deviations) |
| OBSV-05 | Time-per-stage metrics available for bottleneck identification | Computed from state.json progress.current_stage_started_at and manifest.json completed_at timestamps; summarized in analytics report |
</phase_requirements>

## Standard Stack

This phase does not introduce external libraries. Everything is file-based logic within the Claude Code skill system (markdown references read by Claude, not executable code).

### Core Files to Create/Modify

| File | Purpose | Why |
|------|---------|-----|
| `skill/references/observability.md` | Full observability logic: session context, JSONL analytics, rework tracking, spec diff, time metrics | Progressive disclosure -- keeps SKILL.md slim |
| `skill/SKILL.md` (modify) | Add Observability section with Read directives; hook session context into Session Start | Entry point for observability features |

### New Data Files (per-project .lifecycle/)

| File/Directory | Purpose | Format |
|----------------|---------|--------|
| `.lifecycle/sessions/{ISO-date}-{feature}.md` | Session context file | Markdown (human-readable) |
| `.lifecycle/analytics/stage-transitions.jsonl` | Stage transition log | JSONL (one JSON object per line, append-only) |
| `.lifecycle/analytics/rework-events.jsonl` | Rework (backward transition) events | JSONL (append-only) |
| `.lifecycle/analytics/spec-baselines/{feature}.baseline.md` | Spec baseline snapshot at PLAN approval | Markdown (copy of approved spec) |

## Architecture Patterns

### Recommended Structure

```
skill/
  references/
    observability.md           # NEW: all observability logic
  templates/
    (no new templates needed -- session files are generated, not templated)

.lifecycle/                    # Per-project (new directories/files)
  sessions/                    # OBSV-01: session context files
    2026-03-25T14-00-00Z-user-auth.md
    2026-03-26T09-30-00Z-user-auth.md
  analytics/                   # OBSV-02/03/05: structured analytics
    stage-transitions.jsonl
    rework-events.jsonl
    spec-baselines/            # OBSV-04: spec snapshots
      user-auth.baseline.md
```

### Pattern 1: Session Context File (OBSV-01)

**What:** A markdown file created at session start, updated at session end, capturing what happened in this session.

**When to use:** Every session where dev-lifecycle is active.

**How it works:**

At Session Start (after existing steps 1-5):
1. Create session file: `.lifecycle/sessions/{ISO-timestamp}-{feature}.md`
2. Record: feature name, current stage, mode, session start time
3. Snapshot current state: artifact count, decision count, current completeness

At Session End (or before compaction, or at stage transitions):
1. Re-read session file
2. Append: decisions made (delta from decisions.md), artifacts touched (delta from manifest.json), stage transitions that occurred, completeness change
3. Record session end timestamp

**Session file format:**
```markdown
# Session: {ISO-timestamp}

**Feature:** {feature-name}
**Stage at start:** {N} {NAME}
**Stage at end:** {N} {NAME}
**Duration:** {computed from start/end}
**Mode:** {mode}

## Decisions Made
- {decision 1}
- {decision 2}

## Artifacts Touched
- {path} ({type}, {status})

## Stage Transitions
- {from} -> {to} at {time}

## Notes
{any session-specific context from resume_hint}
```

**Key design choice:** Markdown format (not JSON) because session context files are primarily for human review. They are the "session diary" that helps users understand what happened.

### Pattern 2: JSONL Analytics (OBSV-02, OBSV-03, OBSV-05)

**What:** Append-only JSONL files for machine-queryable analytics. Each line is a self-contained JSON object.

**Why JSONL, not JSON array:**
- Append-only is safe (no read-modify-write race)
- Each line is independent (partial file reads work)
- Claude can read and summarize (filter by feature, date range, stage)
- No closing bracket to manage

**stage-transitions.jsonl format:**
```json
{"timestamp":"2026-03-25T14:00:00Z","feature":"user-auth","from_stage":1,"from_name":"PLAN","to_stage":2,"to_name":"DO","direction":"forward","mode":"feature","gate_result":"passed","duration_in_stage_seconds":3600}
```

Fields:
- `timestamp`: ISO 8601
- `feature`: feature name from state.json
- `from_stage`/`from_name`: source stage
- `to_stage`/`to_name`: target stage
- `direction`: "forward", "backward", or "skip"
- `mode`: execution mode at time of transition
- `gate_result`: "passed", "failed", "skipped"
- `duration_in_stage_seconds`: time spent in from_stage (computed from `progress.current_stage_started_at` to transition timestamp)

**rework-events.jsonl format:**
```json
{"timestamp":"2026-03-25T16:00:00Z","feature":"user-auth","from_stage":3,"from_name":"TEST","to_stage":2,"to_name":"DO","reason":"3 spec steps failed verification","impact":"re-implement steps 002, 004","rework_count":1}
```

Fields:
- All fields from stage-transitions plus:
- `reason`: why the backward transition occurred
- `impact`: what needs to be redone
- `rework_count`: how many times this feature has gone backward (running count)

**Integration point:** The stage-transitions.md Transition Procedure (Step 5 "Write history entry") already writes to `.lifecycle/history/`. The new JSONL append happens alongside this existing step -- not replacing it. History entries continue for backward compatibility; JSONL adds queryability.

### Pattern 3: Spec Baseline Snapshot (OBSV-04)

**What:** At PLAN stage completion (when spec is approved), copy the spec to a baseline directory. Later, users can diff baseline against current spec to see all changes.

**When baseline is created:** At the end of PLAN stage, after user approval gate passes, before transitioning to DO.

**Baseline file:** `.lifecycle/analytics/spec-baselines/{feature}.baseline.md`

**Diff generation (on demand):** When user requests spec diff:
1. Read baseline: `.lifecycle/analytics/spec-baselines/{feature}.baseline.md`
2. Read current: `.lifecycle/features/{feature}/spec.md`
3. Compare section by section:
   - Status changes (pending -> implemented -> verified)
   - Deviations added
   - New steps added or removed
4. Present as a structured diff report:

```markdown
# Spec Diff: {feature}

**Baseline:** {baseline date}
**Current:** {current date}

## Status Changes
| Step | Baseline | Current |
|------|----------|---------|
| e2e-{feature}-001 | pending | verified (attempt 1) |
| e2e-{feature}-002 | pending | implemented (override: build timeout) |

## Deviations from Baseline
- DEV-001: {description} (approved)
- DEV-002: {description} (approved)

## Summary
- Steps: {total} total, {verified} verified, {implemented} implemented, {failed} failed
- Deviations: {count}
- Completeness delta: baseline N/A -> current {score}/10
```

### Pattern 4: Time-Per-Stage Metrics (OBSV-05)

**What:** Compute time spent in each stage using existing timestamps. Present as a summary report.

**Data sources (already exist):**
- `state.json` `progress.current_stage_started_at` -- when current stage began
- `manifest.json` `artifacts.{N}_{name}.completed_at` -- when each stage completed
- `.lifecycle/history/` transition entries -- timestamps of all transitions
- `.lifecycle/analytics/stage-transitions.jsonl` -- duration_in_stage_seconds (new, computed at transition time)

**Report format (on demand):**
```markdown
# Time-Per-Stage: {feature}

| Stage | Name | Started | Completed | Duration | Status |
|-------|------|---------|-----------|----------|--------|
| 1 | PLAN | 2026-03-25 14:00 | 2026-03-25 15:00 | 1h 0m | completed |
| 2 | DO | 2026-03-25 15:00 | 2026-03-25 17:30 | 2h 30m | completed |
| 3 | TEST | 2026-03-25 17:30 | 2026-03-25 18:00 | 30m | completed |

**Bottleneck:** Stage 2 (DO) -- 2h 30m (62% of total time)
**Total cycle time:** 4h 0m
**Rework overhead:** 30m (1 rework event)
```

**Key:** No new data collection needed for basic time metrics. The `duration_in_stage_seconds` field in stage-transitions.jsonl provides the queryable version.

### Anti-Patterns to Avoid

- **Replacing existing .lifecycle/history/ with JSONL:** History files continue as-is for backward compatibility. JSONL is an additional analytics layer, not a replacement.
- **Creating executable analytics scripts:** dev-lifecycle is a Claude Code skill. Claude reads JSONL and computes summaries inline. No bash scripts or Node.js analytics processors.
- **Session files that duplicate Living State:** Session context files capture the delta (what changed THIS session). Living State captures the full current state. They complement, not duplicate.
- **Blocking stage transitions on analytics writes:** Analytics writes are fire-and-forget. If JSONL write fails, the stage transition still proceeds. Analytics is observability, not a gate.
- **Over-engineering the spec diff:** The diff is a structured comparison, not a line-by-line git diff. Claude reads both files and summarizes the differences in a human-readable format.

## Don't Hand-Roll

| Problem | Don't Build | Use Instead | Why |
|---------|-------------|-------------|-----|
| Time computation | Duration calculation library | ISO 8601 timestamp subtraction (Claude computes inline) | Simple date math; Claude handles this natively |
| JSONL parsing | Custom parser | Claude reads file, processes line by line | Claude is the runtime; it reads text natively |
| Spec diffing | Line-by-line diff algorithm | Section-by-section structured comparison | Spec has known structure (YAML frontmatter, ## sections); structural diff is more useful than line diff |
| Session tracking | Database or complex state machine | Append-only markdown file per session | One file per session is simple, debuggable, no corruption risk |

**Key insight:** Claude IS the analytics engine. It reads JSONL files, computes summaries, identifies patterns. The reference file tells Claude what to compute and how to present it. No external tooling needed.

## Common Pitfalls

### Pitfall 1: SKILL.md Line Budget Exceeded
**What goes wrong:** Adding observability logic to SKILL.md pushes it over 500 lines (currently 365).
**Why it happens:** Trying to inline analytics logic instead of using references/.
**How to avoid:** SKILL.md gets only a brief Observability section (~10 lines max) with Read directives. All logic in `references/observability.md`.
**Warning signs:** SKILL.md exceeds 380 lines after phase 14 additions.

### Pitfall 2: Session Context File Grows Unbounded
**What goes wrong:** Long sessions with many stage transitions create huge session files.
**Why it happens:** Appending every artifact and decision without limits.
**How to avoid:** Cap session file to ~50 lines. Summarize if it exceeds. Only record deltas (new decisions, new artifacts), not full state.
**Warning signs:** Session files exceeding 100 lines.

### Pitfall 3: JSONL Files Break on Partial Write
**What goes wrong:** Malformed JSON line at end of file causes all reads to fail.
**Why it happens:** Interrupted write or error during JSON serialization.
**How to avoid:** Claude reads JSONL line by line, skipping malformed lines. Each line is independent. Document this tolerance in the reference file.
**Warning signs:** Analytics commands failing on "JSON parse error."

### Pitfall 4: Analytics Duplicates After Rework
**What goes wrong:** A feature that goes DO -> TEST -> DO -> TEST records duplicate transition entries.
**Why it happens:** Each transition is recorded independently.
**How to avoid:** This is correct behavior -- each transition IS a separate event. The `rework_count` field and "backward" direction flag enable filtering. Do not deduplicate.
**Warning signs:** User confused by "duplicate" entries (educate via report format).

### Pitfall 5: Spec Baseline Overwritten on Re-PLAN
**What goes wrong:** If user re-enters PLAN stage (e.g., after rework from DO back to PLAN), baseline is overwritten, losing the original.
**Why it happens:** Baseline creation doesn't check for existing baseline.
**How to avoid:** Only create baseline if it doesn't already exist. If user explicitly wants to reset baseline (new scope), append a version number: `{feature}.baseline.v2.md`.
**Warning signs:** User asks "what was the original spec?" and baseline reflects the revised version.

### Pitfall 6: Conflating Session Start Hook with Analytics
**What goes wrong:** Session Start becomes slow because it tries to compute analytics on every start.
**Why it happens:** Adding analytics computation to the mandatory session start flow.
**How to avoid:** Session Start only creates the session context file (fast, no computation). Analytics reports are generated on demand, not at session start.
**Warning signs:** Session Start takes noticeably longer after Phase 14.

## Code Examples

### Session Context File Creation (Session Start addition)

After existing Session Start steps complete, add:
```markdown
6. **Create session context:** (if `.lifecycle/` exists and feature is active)
   - Create `.lifecycle/sessions/` directory if not exists
   - Create `.lifecycle/analytics/` directory if not exists
   - Create session file: `.lifecycle/sessions/{ISO-timestamp}-{feature}.md`
   - Record: feature, stage, mode, start time, current artifact count
   - Read: `$CLAUDE_SKILL_DIR/references/observability.md` for full session tracking logic
```

### JSONL Append on Stage Transition

Addition to stage-transitions.md Transition Procedure (after existing Step 5):
```markdown
5b. **Append analytics JSONL** (if `.lifecycle/analytics/` exists):
   - Compute `duration_in_stage_seconds` from `progress.current_stage_started_at` to current timestamp
   - Append one line to `.lifecycle/analytics/stage-transitions.jsonl`:
     ```json
     {"timestamp":"{ISO}","feature":"{name}","from_stage":{N},"from_name":"{NAME}","to_stage":{N+1},"to_name":"{NAME}","direction":"forward","mode":"{mode}","gate_result":"{result}","duration_in_stage_seconds":{seconds}}
     ```
   - If backward transition: also append to `.lifecycle/analytics/rework-events.jsonl` with `reason` and `impact` fields
```

### Spec Baseline Snapshot (PLAN stage completion addition)

Addition to plan-stage.md completion:
```markdown
After spec approval gate and before transition to DO:
1. Create `.lifecycle/analytics/spec-baselines/` directory if not exists
2. If baseline does not exist for this feature:
   - Copy `.lifecycle/features/{feature}/spec.md` to `.lifecycle/analytics/spec-baselines/{feature}.baseline.md`
3. If baseline already exists: skip (preserve original baseline)
```

### On-Demand Analytics Report Trigger

New phase transition detection entries for SKILL.md:
```markdown
| "analytics" / "metrics" / "time per stage" | Analytics query | Read observability.md, generate requested report |
| "spec diff" / "what changed" | Spec comparison | Read observability.md, generate spec baseline diff |
| "session history" / "sessions" | Session review | Read observability.md, list/summarize session files |
```

## State of the Art

| Old Approach | Current Approach | When Changed | Impact |
|--------------|------------------|--------------|--------|
| History only in .lifecycle/history/ (per-file JSON) | JSONL analytics alongside history/ | Phase 14 | Queryable transition data without changing existing format |
| No session tracking | Session context files in .lifecycle/sessions/ | Phase 14 | Full visibility into per-session activity |
| No rework tracking | rework-events.jsonl with reason/impact | Phase 14 | Users can identify rework patterns and causes |
| Spec changes invisible | Baseline snapshot + on-demand diff | Phase 14 | Before/after visibility on spec evolution |
| Time metrics not computed | duration_in_stage_seconds + on-demand reports | Phase 14 | Bottleneck identification |

## Validation Architecture

### Test Framework

| Property | Value |
|----------|-------|
| Framework | Manual verification (Claude Code skill -- no executable test framework) |
| Config file | N/A |
| Quick run command | Start a session, verify session file created in .lifecycle/sessions/ |
| Full suite command | Full feature lifecycle: PLAN through COMMIT, verify all 5 OBSV requirements produce expected files |

### Phase Requirements -> Test Map

| Req ID | Behavior | Test Type | Automated Command | File Exists? |
|--------|----------|-----------|-------------------|-------------|
| OBSV-01 | Session context file created | manual | Start session, check .lifecycle/sessions/ for new file | N/A |
| OBSV-02 | Stage transitions in JSONL | manual | Complete a stage transition, verify stage-transitions.jsonl has new line | N/A |
| OBSV-03 | Rework events tracked | manual | Trigger backward transition (TEST->DO), verify rework-events.jsonl | N/A |
| OBSV-04 | Spec baseline diff | manual | Complete PLAN, verify baseline exists; make changes in DO, request diff | N/A |
| OBSV-05 | Time-per-stage metrics | manual | Complete 2+ stages, request "time per stage" report, verify durations | N/A |

### Sampling Rate
- **Per task commit:** Read modified reference files, verify internal consistency
- **Per wave merge:** Walk through full feature lifecycle scenario mentally, verify all JSONL formats
- **Phase gate:** Manual end-to-end test with a mock feature lifecycle

### Wave 0 Gaps
None -- Claude Code skill with markdown-based verification. No test framework infrastructure needed.

## Open Questions

1. **Session file naming collision**
   - What we know: ISO timestamp + feature name should be unique
   - What's unclear: If a user starts two sessions on the same feature within the same second (unlikely but possible)
   - Recommendation: Use full ISO 8601 with timezone. In practice, two Claude sessions on the same feature at the same second is not a realistic scenario.

2. **Analytics file size over time**
   - What we know: JSONL files grow indefinitely. A feature with 20 stage transitions produces 20 lines (~3KB).
   - What's unclear: Whether long-running projects with many features need rotation/archival.
   - Recommendation: Not a concern for v2.0. A project with 100 features would have ~2000 JSONL lines (~300KB). Defer rotation to v3.0 if needed.

3. **Backward compatibility with existing .lifecycle/history/**
   - What we know: History files continue unchanged. JSONL is additive.
   - What's unclear: Whether to backfill JSONL from existing history/ entries.
   - Recommendation: No backfill. JSONL starts empty; new transitions populate it going forward. Existing history/ entries remain accessible for manual review.

## Sources

### Primary (HIGH confidence)
- `skill/SKILL.md` (365 lines) -- current session start flow, state management table, stage transition detection table
- `skill/references/stage-transitions.md` -- existing transition procedure with history writes, backward transition rules, auto-skip
- `skill/references/do-stage.md` -- spec implementation flow, deviation handling, mini-verify
- `skill/references/retrospect-stage.md` -- existing analytics-adjacent patterns (work log, ADR gap check)
- `skill/templates/state.json` -- v2.0 schema with progress.current_stage_started_at timestamp
- `skill/templates/manifest.json` -- artifact registry with completed_at timestamps
- `skill/templates/living-state.md` -- session restoration pattern (complement to session context)
- `skill/references/completeness-scoring.md` -- per-stage quality metrics pattern
- `skill/references/lifecycle-config.md` -- config resolution pattern (reused for analytics settings)

### Secondary (MEDIUM confidence)
- `.planning/REQUIREMENTS.md` -- OBSV-01 through OBSV-05 requirement definitions
- `.planning/STATE.md` -- SKILL.md line budget (365/500, ~135 remaining)
- Phase 13 RESEARCH.md -- established pattern for reference file + SKILL.md modification approach

## Metadata

**Confidence breakdown:**
- Standard stack: HIGH -- no external dependencies; file-based patterns well understood from existing phases
- Architecture: HIGH -- builds directly on existing state.json/manifest.json/history patterns; JSONL is a standard append-only format
- Pitfalls: HIGH -- identified from analyzing current codebase (line budget, session lifecycle, backward transitions)

**Research date:** 2026-03-25
**Valid until:** 2026-04-25 (stable domain -- skill file patterns don't change rapidly)
