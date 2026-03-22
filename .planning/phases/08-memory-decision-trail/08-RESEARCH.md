# Phase 8: Memory & Decision Trail - Research

**Researched:** 2026-03-22
**Domain:** AI memory persistence / session continuity / decision tracking
**Confidence:** HIGH

## Summary

Phase 8 implements four memory mechanisms that directly address the project's core problem: "AI memory loss causes rework." The four mechanisms are: (1) settings change auto-recording, (2) lightweight decision log, (3) Living State Document for instant context restoration, and (4) WHY+SEE code comment integration with ADRs. All four are implemented as markdown files + skill instructions within the existing `.lifecycle/` namespace -- no external libraries or runtime code needed.

The key insight is that all four mechanisms are **write-time** solutions (capture context when decisions happen) rather than **read-time** solutions (reconstruct context later). This aligns with the project's core value: "development artifacts serve as AI's external memory." The Living State Document is the read-time complement -- a single file that aggregates the other three into a session-start snapshot.

**Primary recommendation:** Implement as four markdown-based artifacts within `.lifecycle/`, with update triggers woven into existing stage adapter instructions (do-stage.md, commit-stage.md, etc.) and a new session-start integration point.

<user_constraints>

## User Constraints (from CONTEXT.md)

### Locked Decisions
- Living State Document includes: current state + recent decisions + active settings list + major event summary from project start to now
- Living State updated at: every Stage transition + every session end (both)
- Purpose: read this one file at next session start to instantly restore full context
- Settings changes: ALL changes auto-recorded (.env, config, framework settings, etc.)
- Each settings record includes what (what changed) + why (why it changed)
- Settings records are lighter than ADR -- one line is sufficient
- Lightweight Decision Log: one-line decision entries, lighter than ADR, for direction changes, settings changes, small tradeoffs
- WHY+SEE code comments: already defined in Phase 5 do-stage.md, coexists with `// SPEC:` comments

### Claude's Discretion
- Living State Document's specific markdown structure
- settings-changelog storage location (within .lifecycle/)
- Lightweight decision log file location and format
- SessionStart hook implementation approach

### Deferred Ideas (OUT OF SCOPE)
- SessionStart hook implementation -- Phase 10 (ARCH-03)
- Compaction auto-recovery -- v2 (AUTO-03)

</user_constraints>

<phase_requirements>

## Phase Requirements

| ID | Description | Research Support |
|----|-------------|-----------------|
| MEMO-01 | Settings/config change auto-recording with context (why) + value (what) | Settings changelog file in `.lifecycle/`, update instructions in stage adapters |
| MEMO-02 | Lightweight decision log (lighter than ADR, one-line entries) | Decision log file in `.lifecycle/`, append-only format, ADR threshold as lower bound |
| MEMO-03 | Living State Document -- read once at session start, restore full context | Aggregation document in `.lifecycle/`, updated at stage transitions + session end |
| MEMO-04 | WHY+SEE code comments when ADR created | Already defined in do-stage.md Step 4 and ADR skill; phase needs to formalize the connection |

</phase_requirements>

## Standard Stack

This phase requires no external libraries. All deliverables are markdown files and skill instruction updates within the existing Claude Code skill infrastructure.

### Core Deliverables
| Artifact | Location | Purpose | Why This Location |
|----------|----------|---------|-------------------|
| Settings Changelog | `.lifecycle/settings-changelog.md` | Auto-record all settings changes | `.lifecycle/` namespace established in Phase 1 |
| Decision Log | `.lifecycle/decisions.md` | Lightweight one-line decision entries | Same namespace, feature-independent |
| Living State Document | `.lifecycle/LIVING-STATE.md` | Session-start context snapshot | Top-level within `.lifecycle/` for easy discovery |
| Stage adapter updates | `skill/references/*.md` | Embed update triggers into existing workflows | Follows established adapter pattern |

### Supporting Files
| Artifact | Location | Purpose | When Used |
|----------|----------|---------|-----------|
| Living State template | `skill/templates/living-state.md` | Starter template for new projects | First project initialization |
| Settings changelog template | `skill/templates/settings-changelog.md` | Starter with header + format example | First settings change |

### No Alternatives Needed
This phase is entirely about file formats and skill instructions. There are no library choices to make.

## Architecture Patterns

### Recommended File Structure
```
.lifecycle/
  state.json              # (existing) Stage tracking
  manifest.json           # (existing) Artifact registry
  LIVING-STATE.md         # (NEW - MEMO-03) Session context snapshot
  settings-changelog.md   # (NEW - MEMO-01) Settings change history
  decisions.md            # (NEW - MEMO-02) Lightweight decision log
  history/                # (existing) Stage transition log
  features/               # (existing) Feature specs
```

### Pattern 1: Append-Only Changelog
**What:** Settings changes and decisions are appended to their respective files, never edited or removed. Each entry is a single line (or small block) with timestamp.
**When to use:** Every settings change (MEMO-01) and every lightweight decision (MEMO-02).
**Example:**
```markdown
## Settings Changelog

| Date | File | Change | Reason |
|------|------|--------|--------|
| 2026-03-22 | .env | `DB_HOST=localhost` -> `DB_HOST=prod.db.com` | Deploy to production |
| 2026-03-22 | vite.config.ts | `port: 3000` -> `port: 5173` | Conflict with existing service |
```

### Pattern 2: Living State as Aggregation Document
**What:** LIVING-STATE.md is not a new data source but an aggregation of existing state. It pulls from state.json, settings-changelog.md, decisions.md, manifest.json, and recent ADRs.
**When to use:** Regenerated at every stage transition and session end.
**Structure:**
```markdown
# Living State

**Updated:** {timestamp}
**Feature:** {current feature name}
**Stage:** {N} {NAME} -- {status}

## Current State
- Stage: {N} {NAME}
- Mode: {execution mode}
- Progress: {stages completed} / {total}

## Active Settings
| Setting | Value | Changed | Reason |
|---------|-------|---------|--------|
{last N settings from settings-changelog.md}

## Recent Decisions
{last N entries from decisions.md}

## Event Timeline
| Date | Event | Detail |
|------|-------|--------|
{key events: stage transitions, ADRs created, deviations approved}

## Key ADRs
{list of active ADRs with one-line summaries}

## Resume Hint
{what to do next -- from state.json session.resume_hint}
```

### Pattern 3: Trigger-Based Updates (Embedded in Stage Adapters)
**What:** Rather than a separate "memory system," update triggers are embedded directly into existing stage adapter instructions. When a stage adapter runs, it includes a step to update the relevant memory files.
**When to use:** This is the implementation strategy, not something the user invokes.
**Integration points:**
- `do-stage.md` Step 5 (Completion): Update Living State + append decisions
- `commit-stage.md` Step 5 (Completion): Update Living State + append settings changes
- `stage-transitions.md`: Add Living State update to transition procedure
- Session end: Update Living State with resume hint

### Pattern 4: WHY+SEE Comment Chain (Already Established)
**What:** Code comments linking implementation to ADR decisions.
**Already defined in:** `skill/references/do-stage.md` Step 4 and `~/.claude/skills/adr/SKILL.md` Step 4.
**Comment order when coexisting with SPEC:**
```
// SPEC: e2e-{feature}-{NNN} -- {step title}
// WHY: {one-line decision rationale}
// SEE: docs/decisions/{NNN}-{slug}.md
```
**Phase 8 work:** Formalize this as a cross-reference in SKILL.md and ensure all stage adapters that create ADRs follow this pattern consistently.

### Anti-Patterns to Avoid
- **Over-engineering the Living State:** It should be a flat markdown file regenerated from existing data, not a database or complex state machine.
- **Making memory files the source of truth:** state.json and manifest.json remain the source of truth. Memory files (settings-changelog, decisions, Living State) are derived/supplementary.
- **Requiring user action to update memory:** All updates must be automatic within existing workflows. The user should never need to "remember to update the decision log."
- **Mixing feature-specific and project-level concerns:** Settings changelog and decision log are project-level (in `.lifecycle/`), not per-feature (not in `.lifecycle/features/{name}/`).

## Don't Hand-Roll

| Problem | Don't Build | Use Instead | Why |
|---------|-------------|-------------|-----|
| ADR creation workflow | Custom ADR format | Existing `~/.claude/skills/adr/SKILL.md` 6-step workflow | Already handles numbering, format, WHY+SEE comments |
| Stage transition tracking | Custom transition log | Existing `.lifecycle/history/` JSON entries | Already records timestamps, gate results, modes |
| State persistence | Custom state format | Existing `state.json` + `manifest.json` | Already established in Phase 1 |
| Session resume | Custom resume mechanism | `state.json` `session.resume_hint` field | Already exists in template |

**Key insight:** Phase 8 adds memory **on top of** existing infrastructure, not alongside it. Every new file draws from or extends existing artifacts.

## Common Pitfalls

### Pitfall 1: Living State Becomes Stale
**What goes wrong:** Living State is generated once but not updated, so the next session reads outdated information.
**Why it happens:** Update triggers are forgotten or skipped during rapid iteration.
**How to avoid:** Embed Living State update as a mandatory step in stage-transitions.md (not optional). Make it part of the transition procedure checklist.
**Warning signs:** `LIVING-STATE.md` timestamp is older than `state.json` `session.last_active`.

### Pitfall 2: Settings Changelog Grows Unbounded
**What goes wrong:** After months of use, settings-changelog.md becomes thousands of lines.
**Why it happens:** Append-only with no compaction.
**How to avoid:** Living State only shows the **last N** settings (e.g., last 20). The full changelog remains as history but is not loaded in full at session start. Compaction is deferred to v2 (AUTO-03).
**Warning signs:** File exceeds 200 lines.

### Pitfall 3: Decision Log Duplicates ADRs
**What goes wrong:** The same decision appears in both the lightweight decision log AND as a full ADR, causing confusion about which is authoritative.
**Why it happens:** Unclear threshold between "lightweight decision" and "ADR-worthy decision."
**How to avoid:** Define clear criteria: lightweight decisions are single-line entries for small choices. If a decision meets BOTH ADR criteria (2+ viable approaches AND lasting consequences), it goes to ADR, NOT the lightweight log. The decision log entry can reference the ADR: "See ADR-005."
**Warning signs:** Decision log entries exceed 2 lines or mention alternatives.

### Pitfall 4: WHY+SEE Comments Orphaned After Refactor
**What goes wrong:** Code is refactored, WHY+SEE comments are lost or moved to irrelevant locations.
**Why it happens:** Automated refactoring tools don't understand semantic comments.
**How to avoid:** This is inherent to code comments. The ADR document itself (`docs/decisions/`) is the durable record. WHY+SEE comments are convenience pointers, not the source of truth. Stage 8 RETROSPECT can include a "check for orphaned WHY+SEE" step.
**Warning signs:** `grep -r "WHY:" src/` returns fewer results than `ls docs/decisions/` entries.

### Pitfall 5: Session-Start Overload
**What goes wrong:** Loading Living State + state.json + manifest.json + settings-changelog + decision log at session start consumes too much context.
**Why it happens:** Each mechanism adds more files to read.
**How to avoid:** Living State is the ONLY file read at session start. It aggregates everything needed. Other files are consulted only when detailed history is needed. This is the entire point of MEMO-03.
**Warning signs:** Session start instructions list more than 2 files to read.

## Code Examples

### Settings Changelog Entry Format
```markdown
| 2026-03-22 14:30 | `.env` | `API_KEY=old...` -> `API_KEY=new...` | Rotated expired API key |
| 2026-03-22 15:00 | `vite.config.ts` | Added `proxy: { '/api': 'http://localhost:8080' }` | Backend running on separate port during dev |
```

### Lightweight Decision Log Entry Format
```markdown
| 2026-03-22 | Use Zustand over Redux for state management | Simpler API, no boilerplate |
| 2026-03-22 | Skip SSR for MVP | Complexity not justified for initial launch |
| 2026-03-22 | See ADR-005 | Database migration strategy (promoted to ADR) |
```

### Living State Document (Full Example)
```markdown
# Living State

**Updated:** 2026-03-22T15:30:00Z
**Feature:** user-authentication
**Stage:** 3 TEST -- in_progress
**Mode:** feature

## Current State

- Inner loop progress: PLAN done, DO done, TEST in progress
- 3 spec steps verified, 2 remaining
- 1 deviation approved (DEV-001: OAuth instead of email/password)

## Active Settings

| Setting | Value | Changed | Reason |
|---------|-------|---------|--------|
| `.env` AUTH_PROVIDER | `google-oauth` | 2026-03-22 | DEV-001: OAuth chosen over email/password |
| `next.config.js` redirects | Added `/api/auth/*` | 2026-03-22 | OAuth callback routing |

## Recent Decisions

| Date | Decision | Reason |
|------|----------|--------|
| 2026-03-22 | OAuth over email/password | Faster MVP, no password reset flow needed |
| 2026-03-21 | Next.js App Router | Recommended by framework, better streaming |

## Event Timeline

| Date | Event |
|------|-------|
| 2026-03-22 15:00 | TEST stage started |
| 2026-03-22 14:00 | DO stage completed (5/5 spec steps) |
| 2026-03-22 13:00 | DEV-001 approved (OAuth deviation) |
| 2026-03-22 12:00 | ADR-005 created (OAuth decision) |
| 2026-03-22 10:00 | PLAN stage completed, spec agreed |

## Key ADRs

- ADR-005: Use Google OAuth instead of email/password authentication [Accepted]

## Resume Hint

DO complete. TEST in progress -- 3/5 spec steps verified. Next: verify e2e-auth-004 (error handling).
```

### Stage Adapter Update Trigger (Added to do-stage.md Step 5)
```markdown
**f. Update memory files:**

1. Append any settings changes made during DO to `.lifecycle/settings-changelog.md`
2. Append any lightweight decisions to `.lifecycle/decisions.md`
3. Regenerate `.lifecycle/LIVING-STATE.md` with current state
```

### Stage Transition Update Trigger (Added to stage-transitions.md)
```markdown
6. **Update Living State** -- After successful transition, regenerate `.lifecycle/LIVING-STATE.md`:
   - Read current `state.json` for stage/status/feature
   - Read last 20 entries from `settings-changelog.md`
   - Read last 10 entries from `decisions.md`
   - Read `manifest.json` for event timeline
   - Write aggregated LIVING-STATE.md
```

## State of the Art

| Old Approach | Current Approach | When Changed | Impact |
|--------------|------------------|--------------|--------|
| Manual "remember this" messages | Structured artifacts as external memory | Phase 1 (foundation) | Artifacts persist across sessions |
| No settings history | Settings auto-recorded in changelog | Phase 8 (this phase) | Never re-explain why a setting was changed |
| Full ADR for every decision | Lightweight log + ADR for significant decisions | Phase 8 (this phase) | Lower friction, higher recording rate |
| Read multiple files at session start | Single Living State Document | Phase 8 (this phase) | Instant context restoration |

## Open Questions

1. **Living State size limit**
   - What we know: Living State aggregates from multiple sources. Must stay readable in one pass.
   - What's unclear: Exact line count limit before it becomes unwieldy. Training data suggests ~100-150 lines is comfortable for a single context-loading file.
   - Recommendation: Target 100 lines max. Use "last N" truncation for changelog/decisions sections. Full history stays in source files.

2. **Settings detection scope**
   - What we know: User decided "ALL changes auto-recorded (.env, config, framework settings, etc.)"
   - What's unclear: How Claude detects that a settings change happened during DO stage. It relies on Claude's awareness of which files are "settings files."
   - Recommendation: Provide a non-exhaustive list of common settings file patterns (.env*, *.config.*, *.json config, *.yaml config, *.toml) as guidance in the adapter instructions. Claude uses judgment for unlisted files.

3. **MEMO-04 scope -- is new work needed?**
   - What we know: WHY+SEE is already fully defined in do-stage.md Step 4 and ADR skill Step 4.
   - What's unclear: Whether MEMO-04 requires any NEW implementation or just formal documentation that the pattern exists.
   - Recommendation: MEMO-04 should add a brief cross-reference section to SKILL.md and ensure consistency across all stage adapters that might create ADRs. Minimal new code.

## Validation Architecture

### Test Framework
| Property | Value |
|----------|-------|
| Framework | Manual verification (skill files are markdown instructions, not executable code) |
| Config file | none |
| Quick run command | `ls -la .lifecycle/LIVING-STATE.md .lifecycle/settings-changelog.md .lifecycle/decisions.md` |
| Full suite command | Manual: verify file existence + format + update triggers in adapters |

### Phase Requirements -> Test Map
| Req ID | Behavior | Test Type | Automated Command | File Exists? |
|--------|----------|-----------|-------------------|-------------|
| MEMO-01 | Settings changelog file created with correct format | smoke | `head -5 .lifecycle/settings-changelog.md` | Wave 0 |
| MEMO-02 | Decision log file created with correct format | smoke | `head -5 .lifecycle/decisions.md` | Wave 0 |
| MEMO-03 | Living State document created with all sections | smoke | `grep -c "^##" .lifecycle/LIVING-STATE.md` | Wave 0 |
| MEMO-04 | WHY+SEE pattern referenced in SKILL.md and adapters | manual-only | `grep "WHY+SEE\|WHY.*SEE" skill/references/do-stage.md` | Existing |

### Sampling Rate
- **Per task commit:** Verify created/modified files exist and have correct headers
- **Per wave merge:** Manual review of all 4 artifacts + adapter update points
- **Phase gate:** All MEMO-01 through MEMO-04 files exist with correct format

### Wave 0 Gaps
- [ ] `.lifecycle/settings-changelog.md` -- template for MEMO-01
- [ ] `.lifecycle/decisions.md` -- template for MEMO-02
- [ ] `.lifecycle/LIVING-STATE.md` -- template for MEMO-03
- [ ] `skill/templates/living-state.md` -- reusable template
- [ ] `skill/templates/settings-changelog.md` -- reusable template
- [ ] `skill/templates/decisions.md` -- reusable template

## Sources

### Primary (HIGH confidence)
- `skill/templates/state.json` -- existing state schema with session fields
- `skill/references/do-stage.md` -- ADR detection + WHY+SEE pattern (Step 4)
- `skill/references/stage-transitions.md` -- transition procedure (update trigger integration point)
- `skill/references/commit-stage.md` -- commit workflow (settings detection integration point)
- `~/.claude/skills/adr/SKILL.md` -- WHY+SEE comment format and placement rules
- `skill/SKILL.md` -- Session Start section (Living State read point)
- `.planning/phases/08-memory-decision-trail/08-CONTEXT.md` -- user decisions
- `.planning/phases/01-foundation/01-CONTEXT.md` -- `.lifecycle/` namespace decision

### Secondary (MEDIUM confidence)
- `.planning/PROJECT.md` -- core problem statement (memory loss causes rework)

### Tertiary (LOW confidence)
- None

## Metadata

**Confidence breakdown:**
- Standard stack: HIGH -- no libraries needed, all markdown-based within established patterns
- Architecture: HIGH -- follows established `.lifecycle/` namespace and adapter pattern from Phases 1-7
- Pitfalls: HIGH -- identified from direct analysis of the design (staleness, unbounded growth, duplication)

**Research date:** 2026-03-22
**Valid until:** Indefinite (stable domain -- file format design, no external dependencies)
