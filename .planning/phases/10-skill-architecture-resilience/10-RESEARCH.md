# Phase 10: Skill Architecture & Resilience - Research

**Researched:** 2026-03-22
**Domain:** Claude Code skill architecture, state resilience, session automation
**Confidence:** HIGH

## Summary

Phase 10 is a hardening and verification phase. The project has completed 9 phases of implementation; all core functionality exists. This phase addresses 5 remaining requirements: ARCH-01 (progressive disclosure), ARCH-02 (adapter pattern verification), ARCH-03 (SessionStart hook), ARCH-05 (state reconcile), and FOUND-04 (execution modes with stage skipping).

The most technically interesting work is the SessionStart hook (ARCH-03) which leverages Claude Code's native `hooks.SessionStart` in `.claude/settings.json`. The reconcile logic (ARCH-05) requires a filesystem-scanning algorithm to infer state from `.lifecycle/` artifacts when `state.json` is lost. The remaining items (ARCH-01, ARCH-02, FOUND-04) are primarily verification and documentation of already-implemented patterns.

**Primary recommendation:** Implement SessionStart hook as a shell script invoked via `.claude/settings.json` that cats `.lifecycle/LIVING-STATE.md` and `.lifecycle/state.json` into stdout. Add reconcile logic as a documented procedure in a new reference file. Verify ARCH-01/ARCH-02 against existing code. Wire execution modes into stage-transitions.md with skip logic.

<user_constraints>

## User Constraints (from CONTEXT.md)

### Locked Decisions
- **ARCH-01 (SKILL.md Progressive Disclosure):** Already at ~300 lines, 500-line limit maintained. All Stage adapters separated to references/.
- **ARCH-02 (Adapter Pattern):** Already implemented. Phase 10 is verification + cleanup only.
- **ARCH-03 (SessionStart Hook):** Phase 8 added Step 1.5 manual reading. Phase 10 upgrades to automatic hook.
- **ARCH-05 (State Reconcile):** state.json loss recovery via .lifecycle/ filesystem scanning. Uses manifest.json, spec files, verification results to infer current Stage.
- **FOUND-04 (Execution Modes):** hotfix/feature/release/milestone modes with stage-skip rules already defined in stage-transitions.md.

### Claude's Discretion
- SessionStart hook implementation method (settings.json hook vs SKILL.md directive)
- Reconcile algorithm details
- SKILL.md final cleanup/optimization scope

### Deferred Ideas (OUT OF SCOPE)
None -- final phase.

</user_constraints>

<phase_requirements>

## Phase Requirements

| ID | Description | Research Support |
|----|-------------|-----------------|
| ARCH-01 | SKILL.md 500 lines max, details in references/ (progressive disclosure) | Current count: 324 lines. 15 reference files + 7 templates already exist. Verification task only. |
| ARCH-02 | Adapter pattern -- existing skills called via interface, no modification | All 9 stages use identical section format (Purpose/Primary/Supporting/Outputs/Read/Pipeline). Verification task only. |
| ARCH-03 | SessionStart hook for automatic Living State loading | Claude Code supports `hooks.SessionStart` in settings.json. Shell script outputs to stdout, injected as context. See Architecture Patterns section. |
| ARCH-05 | state.json loss recovery via filesystem reconcile | Reconcile algorithm infers stage from manifest.json existence, spec files, verification results, history/ entries. See Reconcile Pattern section. |
| FOUND-04 | Execution modes (hotfix/feature/release/milestone) with stage skipping | Mode definitions exist in stage-transitions.md. Need to wire skip logic into SKILL.md Session Start flow and state.json mode field. |

</phase_requirements>

## Standard Stack

This phase involves no external libraries. All work is within Claude Code's skill system (Markdown files, JSON templates, shell scripts).

### Core
| Component | Location | Purpose | Why Standard |
|-----------|----------|---------|--------------|
| SKILL.md | `skill/SKILL.md` | Main skill entry point | Claude Code skill convention |
| settings.json hooks | `.claude/settings.json` | SessionStart automation | Official Claude Code hook system |
| Shell script | `.claude/hooks/` or `skill/hooks/` | Hook command handler | Only `type: "command"` supported by Claude Code |
| Reference files | `skill/references/*.md` | Progressive disclosure targets | Established project pattern |
| JSON templates | `skill/templates/*.json` | state.json, manifest.json schemas | Established project pattern |

### Supporting
| Component | Location | Purpose | When to Use |
|-----------|----------|---------|-------------|
| state.json | `.lifecycle/state.json` | Runtime state (stage, mode, feature) | Every session, every transition |
| manifest.json | `.lifecycle/manifest.json` | Artifact registry with gate rules | Gate checks, reconcile source |
| LIVING-STATE.md | `.lifecycle/LIVING-STATE.md` | Session context restoration | SessionStart hook reads this |
| history/ | `.lifecycle/history/*.json` | Transition log | Reconcile secondary source |

**No installation needed.** All components are Markdown, JSON, and shell scripts within the existing project structure.

## Architecture Patterns

### Pattern 1: SessionStart Hook via settings.json

**What:** Claude Code's `hooks.SessionStart` fires a shell script on every session start/resume/clear/compact. The script's stdout is injected as Claude's context automatically.

**When to use:** ARCH-03 -- replace manual Step 1.5 "Read Living State" with automatic loading.

**Recommended implementation:**

Create `.claude/settings.json` (project-scoped, not local):

```json
{
  "hooks": {
    "SessionStart": [
      {
        "matcher": "",
        "hooks": [
          {
            "type": "command",
            "command": "cat .lifecycle/LIVING-STATE.md 2>/dev/null && echo '---' && cat .lifecycle/state.json 2>/dev/null || echo 'No lifecycle state found. Run dev-lifecycle to initialize.'",
            "timeout": 5,
            "statusMessage": "Loading dev-lifecycle context..."
          }
        ]
      }
    ]
  }
}
```

**Key details (HIGH confidence, from official docs):**
- Matcher types for SessionStart: `startup`, `resume`, `clear`, `compact`
- Empty string matcher `""` matches ALL session types -- use this for always-load behavior
- stdout is automatically added as context Claude can see
- Keep hooks fast (< 5 seconds) -- they run on every session start
- Exit code 0 = success, exit 2 = blocking error
- `CLAUDE_ENV_FILE` available for setting environment variables

**Source:** [Claude Code Hooks Reference](https://code.claude.com/docs/en/hooks)

**Design decision (Claude's discretion):** Use inline `cat` command rather than a separate shell script file. Rationale:
1. The command is simple enough (two cat calls with fallback)
2. Avoids managing a separate hook script file
3. Reduces deployment complexity
4. If logic grows later, can migrate to a script file

**Alternative considered:** A dedicated shell script at `.claude/hooks/session-start.sh` that also runs `jq` to parse state.json and format a summary. This is overkill for Phase 10 -- the raw file contents are sufficient for Claude to understand context.

### Pattern 2: State Reconcile Algorithm

**What:** When `.lifecycle/state.json` is missing or corrupted, reconstruct current state from other `.lifecycle/` files.

**When to use:** ARCH-05 -- resilience against state loss (compaction bug, manual deletion, git conflicts).

**Algorithm:**

```
RECONCILE PROCEDURE:
1. Check .lifecycle/manifest.json exists
   - If missing: state is "fresh" -> initialize from templates
   - If exists: proceed to step 2

2. Scan manifest.json artifacts (highest completed stage wins):
   For stage in [9, 8, 7, 6, 5, 4, 3, 2, 1]:
     If artifacts.{stage_key}.outputs.length > 0 AND artifacts.{stage_key}.completed_at != null:
       last_completed = stage
       break

3. Determine current stage:
   - If last_completed found: current_stage = last_completed + 1
   - If last_completed is 9: cycle complete, reset to stage 1
   - If no completed stages: current_stage = 1

4. Infer mode from context:
   - Check if stages were skipped (status = "skipped" in history/)
   - Match skip pattern to known modes:
     - Stages 1,7,8,9 skipped -> hotfix
     - Stages 5,6,9 skipped -> feature
     - Stage 9 only skipped -> release
     - Nothing skipped -> milestone
   - Default: "feature" if unclear

5. Reconstruct state.json:
   - version: "1.0"
   - project.name: from manifest.json feature field or directory name
   - current.stage: from step 3
   - current.status: "not_started" (conservative)
   - current.mode: from step 4
   - progress.stages_completed: from manifest scan
   - session.resume_hint: "State recovered via reconcile. Verify current position."

6. Write recovered state.json
7. Log reconcile event to .lifecycle/history/
```

**Implementation location:** New reference file `skill/references/state-reconcile.md` -- follows progressive disclosure pattern.

**SKILL.md integration:** Add reconcile check to Session Start section, Step 1:

```
1. Read state: Load .lifecycle/state.json
   - If not exists AND .lifecycle/ directory exists: Run reconcile (Read: references/state-reconcile.md)
   - If not exists AND no .lifecycle/: Initialize from templates
```

### Pattern 3: Execution Mode Stage Skipping

**What:** When mode is set (hotfix/feature/release/milestone), automatically skip non-required stages.

**When to use:** FOUND-04 -- the skip rules are already defined in `stage-transitions.md` but not wired into the active transition flow.

**Current state:** The mode-to-skip mapping exists in stage-transitions.md:

| Mode | Skippable Stages |
|------|-----------------|
| hotfix | 1, 7, 8, 9 |
| feature | 5, 6, 9 |
| release | 9 |
| milestone | none |

**What needs to happen:**
1. SKILL.md Session Start should display current mode and note which stages will be skipped
2. Stage transition logic in stage-transitions.md needs explicit "auto-skip" procedure:
   - When transitioning past a skippable stage, set its status to `skipped` and advance to the next required stage
3. state.json template already has `current.mode` field -- this is the control point

### Pattern 4: SKILL.md Progressive Disclosure Verification

**What:** Verify SKILL.md stays under 500 lines, all detailed logic lives in references/, main file uses `Read:` directives.

**Current state (verified):**
- SKILL.md: **324 lines** (well under 500)
- references/: **15 files** (all 9 stage adapters + 6 supporting docs)
- templates/: **7 files** (state, manifest, spec, prototype, living-state, settings-changelog, decisions)
- All stages use `Read: $CLAUDE_SKILL_DIR/references/{stage}-stage.md` pattern

**What needs to happen:** Audit pass -- verify no inline logic that should be in references/. Document the progressive disclosure structure in SKILL.md itself or a reference.

### Anti-Patterns to Avoid

- **Over-engineering the reconcile:** Don't try to recover every field perfectly. Conservative defaults ("not_started" status, "feature" mode) are better than wrong confident guesses.
- **Heavy SessionStart hook:** Don't run complex logic (git log, file scanning) in the hook. Just cat the pre-generated LIVING-STATE.md. Keep it < 1 second.
- **Modifying SKILL.md structure:** Phase 10 is hardening, not restructuring. The 9-stage format and section layout are locked. Only add/modify minimal sections (reconcile check, mode display).
- **Breaking .claude/settings.local.json:** The project already has `.claude/settings.local.json` with permissions. The new `.claude/settings.json` (non-local) is separate and should not conflict.

## Don't Hand-Roll

| Problem | Don't Build | Use Instead | Why |
|---------|-------------|-------------|-----|
| Session context loading | Custom SKILL.md "read this first" directive | Claude Code `hooks.SessionStart` | Native, automatic, works even after compaction |
| State file watching | File system watcher or polling | One-time reconcile on state.json absence | Claude Code is request-response, not daemon |
| Complex mode selection UI | Interactive mode picker | Simple `state.json.current.mode` field set once | Modes are project-level, not session-level |
| JSON schema validation | Custom validator | Simple field existence checks | Complexity not justified for skill-internal JSON |

## Common Pitfalls

### Pitfall 1: settings.json Scope Confusion
**What goes wrong:** Hook defined in wrong settings file, doesn't fire.
**Why it happens:** Claude Code has 3 settings scopes: `~/.claude/settings.json` (global), `.claude/settings.json` (project), `.claude/settings.local.json` (local).
**How to avoid:** Use `.claude/settings.json` (project-scoped, committable) for the SessionStart hook. This ensures all developers using the skill get the hook. Do NOT put it in settings.local.json (that's for permissions only).
**Warning signs:** Hook doesn't fire on session start; Living State not loaded automatically.

### Pitfall 2: Reconcile Over-Confidence
**What goes wrong:** Reconcile guesses wrong stage, user proceeds with incorrect state.
**Why it happens:** Incomplete data in .lifecycle/ (e.g., manifest exists but history/ is empty).
**How to avoid:** Always set `resume_hint` to warn user about reconcile. Set status to `not_started` (not `in_progress`). Display reconcile result for user confirmation.
**Warning signs:** Stage number doesn't match user's expectation of progress.

### Pitfall 3: Hook Timeout on Large Projects
**What goes wrong:** `cat` of a very large LIVING-STATE.md or state.json times out.
**Why it happens:** LIVING-STATE.md is designed to be ~100 lines max, but could grow if generation logic is not enforced.
**How to avoid:** Keep LIVING-STATE.md at ~100 lines (already specified in template). Set hook timeout to 5 seconds (generous for cat).
**Warning signs:** Session start takes noticeably longer.

### Pitfall 4: settings.json Merge Conflict
**What goes wrong:** New `.claude/settings.json` with hooks conflicts with existing `.claude/settings.local.json`.
**Why it happens:** These are separate files -- they don't conflict. But user might confuse them.
**How to avoid:** Document clearly: `settings.json` = shared project config (hooks), `settings.local.json` = personal config (permissions). Both are loaded; they merge.

### Pitfall 5: Mode Not Persisted Across Sessions
**What goes wrong:** User sets mode to "hotfix" but next session defaults back to "feature".
**Why it happens:** Mode is in state.json which persists. This shouldn't happen if state.json is correctly managed.
**How to avoid:** Verify mode is read from state.json on session start, not defaulted. Reconcile should infer mode when possible.

## Code Examples

### SessionStart Hook Configuration
```json
// Source: Claude Code official hooks documentation
// File: .claude/settings.json
{
  "hooks": {
    "SessionStart": [
      {
        "matcher": "",
        "hooks": [
          {
            "type": "command",
            "command": "cat .lifecycle/LIVING-STATE.md 2>/dev/null && echo '---' && cat .lifecycle/state.json 2>/dev/null || echo 'No lifecycle state found.'",
            "timeout": 5,
            "statusMessage": "Loading dev-lifecycle context..."
          }
        ]
      }
    ]
  }
}
```

### Reconcile Entry Point in SKILL.md
```markdown
## Session Start

On session start (or after compaction):

1. **Read state:** Load `.lifecycle/state.json`
   - If not exists AND `.lifecycle/` directory exists: **Reconcile** (Read: `$CLAUDE_SKILL_DIR/references/state-reconcile.md`)
   - If not exists AND no `.lifecycle/`: Initialize from templates
```

### Execution Mode Skip Logic
```markdown
## Stage Transition with Mode Skip

When transitioning from completed Stage N:
1. Determine next_stage = N + 1
2. Check if next_stage is skippable in current mode (see mode table)
3. If skippable:
   - Set stage N+1 status to "skipped" in state.json
   - Record skip in .lifecycle/history/
   - Repeat from step 1 with N = N + 1
4. If not skippable: transition to next_stage normally
```

### Manifest-Based Stage Inference (Reconcile)
```
Scan order (highest first):
  9_promote -> 8_retrospect -> 7_document -> 6_deploy_test ->
  5_deploy -> 4_commit -> 3_test -> 2_do -> 1_plan

For each stage key in manifest.json:
  If artifacts[key].completed_at is not null:
    last_completed_stage = stage_number
    BREAK

current_stage = last_completed_stage + 1
If current_stage > 9: current_stage = 1 (new cycle)
```

## State of the Art

| Old Approach | Current Approach | When Changed | Impact |
|--------------|------------------|--------------|--------|
| SKILL.md Step 1.5 manual read | SessionStart hook auto-load | Phase 10 (now) | No user action needed for context restoration |
| No state recovery | Filesystem reconcile | Phase 10 (now) | Resilience against state.json loss |
| Mode stored but unused | Mode drives stage skipping | Phase 10 (now) | Hotfix/feature flows skip unnecessary stages |

## Validation Architecture

### Test Framework
| Property | Value |
|----------|-------|
| Framework | Manual verification (Claude Code skill, no automated test runner) |
| Config file | None -- skill is Markdown/JSON, not executable code |
| Quick run command | Manual: verify SKILL.md line count, check settings.json validity |
| Full suite command | Manual: full walkthrough of session start + reconcile + mode skip |

### Phase Requirements -> Test Map
| Req ID | Behavior | Test Type | Automated Command | File Exists? |
|--------|----------|-----------|-------------------|-------------|
| ARCH-01 | SKILL.md <= 500 lines, references/ contains all adapters | smoke | `wc -l skill/SKILL.md && ls skill/references/` | N/A (shell) |
| ARCH-02 | All stages use Read: directive pattern, no inline logic | manual | Inspect SKILL.md stage sections for `Read:` directives | N/A |
| ARCH-03 | SessionStart hook fires and loads LIVING-STATE.md | manual | Start new Claude Code session, verify context loaded | N/A |
| ARCH-05 | Reconcile recovers state from filesystem | manual | Delete state.json, start session, verify recovery | N/A |
| FOUND-04 | Mode-based stage skipping works | manual | Set mode to "hotfix", verify stages 1,7,8,9 are skipped | N/A |

### Sampling Rate
- **Per task commit:** `wc -l skill/SKILL.md` (must be <= 500)
- **Per wave merge:** Manual walkthrough of changed files
- **Phase gate:** All 5 requirements verified via manual checklist

### Wave 0 Gaps
None -- this phase produces Markdown and JSON files, not executable code. Validation is structural (line counts, file existence, JSON validity).

## Open Questions

1. **Should the SessionStart hook also load state.json?**
   - What we know: LIVING-STATE.md contains a summary of state.json data. Loading both provides redundancy.
   - What's unclear: Whether loading both adds noise or valuable detail.
   - Recommendation: Load both. state.json is small (~25 lines) and provides exact stage/status values that LIVING-STATE.md may summarize loosely.

2. **Should reconcile be automatic or user-confirmed?**
   - What we know: Conservative reconcile (default to "not_started") is safe but may annoy users who know their state.
   - What's unclear: How often state.json actually gets lost in practice.
   - Recommendation: Auto-reconcile with user notification. Display "State recovered via reconcile -- verify position" and let user correct if wrong.

3. **Should .claude/settings.json be created or documented?**
   - What we know: `.claude/settings.local.json` already exists with permissions. `.claude/settings.json` is a separate file for project-wide shared config.
   - What's unclear: Whether creating this file should be a task or just documented as an installation step.
   - Recommendation: Create the file as a task. It's a concrete deliverable for ARCH-03.

## Sources

### Primary (HIGH confidence)
- [Claude Code Hooks Reference](https://code.claude.com/docs/en/hooks) -- SessionStart hook configuration, matcher types, stdout injection, JSON structure
- [Claude Code Settings](https://code.claude.com/docs/en/settings) -- Settings file hierarchy and merge behavior
- Project files: `skill/SKILL.md` (324 lines), `skill/references/stage-transitions.md` (mode skip rules), `skill/templates/state.json` (reconcile target schema), `skill/templates/manifest.json` (gate rules for reconcile)

### Secondary (MEDIUM confidence)
- [Claude Code Session Hooks Blog](https://claudefa.st/blog/tools/hooks/session-lifecycle-hooks) -- Practical SessionStart patterns

### Tertiary (LOW confidence)
- None -- all findings verified against official docs or project source code.

## Metadata

**Confidence breakdown:**
- Standard stack: HIGH -- this is all internal project structure (Markdown, JSON, shell), no external dependencies
- Architecture: HIGH -- SessionStart hook mechanism verified against official Claude Code docs; reconcile is a straightforward filesystem scan
- Pitfalls: HIGH -- settings.json scoping is well-documented; reconcile edge cases are understood from the existing state.json/manifest.json schema

**Research date:** 2026-03-22
**Valid until:** 2026-04-22 (stable -- Claude Code hooks API is established)
