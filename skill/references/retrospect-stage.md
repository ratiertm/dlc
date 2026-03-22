# RETROSPECT Stage Adapter

This reference defines the complete RETROSPECT Stage workflow -- Stage 8 of the dev-lifecycle pipeline. When Claude reads this file, it follows the instructions below to orchestrate retrospective generation, ADR gap checking, and work logging by delegating to their respective skills.

**CRITICAL:** This adapter orchestrates existing skills. It does NOT contain retrospective templates, git log analysis, or work-log formatting. Those belong to their respective skills (gsd-retrospective, adr, work-log).

## When This Runs

- User says "retro", "retrospective", "reflect", or similar
- `state.json` `current.stage = 8` (or transitioning from 7 to 8)
- SKILL.md Stage 8 section directs here via `Read: $CLAUDE_SKILL_DIR/references/retrospect-stage.md`

## Prerequisites

- `.lifecycle/` directory initialized (from PLAN stage)
- `state.json` and `manifest.json` exist
- DOCUMENT stage must have outputs -- gate 7_to_8 must pass (`artifacts.7_document.outputs.length > 0`)

## Step 0: Stage Initialization

Before any retrospective work, verify the pipeline state and check mode-based skipping.

**Actions:**

1. Read `.lifecycle/state.json` and verify:
   - `current.feature` has a value
   - `current.stage` = `8` (or transition from 7 to 8)

2. Check gate condition (7_to_8):
   - Read `.lifecycle/manifest.json`
   - Verify: `artifacts.7_document.outputs.length > 0`
   - If gate fails: set `current.status = "blocked"`, display missing artifacts, STOP

3. Check mode-based skip:
   - Read `current.mode` from `state.json`
   - If mode is `hotfix`: offer skip -- "Hotfix mode allows skipping RETROSPECT. Skip? (yes/no)"
   - If user chooses skip: register `{ "type": "skipped", "path": "N/A", "status": "skipped" }` in `artifacts.8_retrospect.outputs`, set `artifacts.8_retrospect.completed_at`, set `current.status = "skipped"`, announce skip, STOP

4. Update `state.json`:
   - `current.stage` = `8`
   - `current.stage_name` = `"RETROSPECT"`
   - `current.status` = `"in_progress"`
   - `progress.current_stage_started_at` = current ISO 8601 timestamp

## Step 1: Retrospective Generation

**Purpose:** Generate a structured retrospective by delegating to the gsd-retrospective skill.

**Actions:**

1. Invoke the gsd-retrospective skill per its interface:
   - Read `~/.claude/skills/gsd-retrospective/SKILL.md` for invocation instructions
   - Provide context: feature name, stage history from `.lifecycle/history/`, spec step results from verification.json
2. The gsd-retrospective skill handles:
   - Git log analysis for the feature
   - What went well / what to improve identification
   - Metrics collection (duration, rework count, etc.)
3. Output destination: `docs/retrospectives/{feature-name}-retro.md`
4. Do NOT reimplement any retrospective logic -- the skill owns this entirely

## Step 2: ADR Gap Check

**Purpose:** Identify decisions made during the feature lifecycle that were not formally recorded as ADRs.

**Actions:**

1. Scan git log for the feature commits looking for decision indicators:
   - Commit messages containing "decided", "chose", "switched", "instead of"
   - Changes to config files that indicate architectural choices
   - Read `.lifecycle/decisions.md` for recorded decisions
2. Compare against existing ADRs in `docs/decisions/`
3. For each potential gap, suggest ADR creation:
   - Display: "Potential unrecorded decision: {summary}. Create ADR? (yes/no)"
   - If user approves: invoke ADR skill per `~/.claude/skills/adr/SKILL.md` interface
   - If user declines: note as "acknowledged, no ADR needed"
4. Do NOT create ADRs directly -- suggest and let user approve each one

## Step 3: Work Log

**Purpose:** Record the feature work in the project work log.

**Actions:**

1. Invoke work-log skill per its interface:
   - Read `~/.claude/skills/work-log/SKILL.md` for invocation instructions
   - Provide: feature name, start/end timestamps, key outcomes, stage durations
2. The work-log skill handles:
   - Formatting the log entry
   - Writing to the configured Obsidian vault or project log location
3. Do NOT reimplement work-log formatting -- the skill owns this entirely

## Step 4: Living State Update

**Purpose:** Regenerate the Living State document with lessons from the retrospective.

**Actions:**

1. Follow the regeneration procedure from stage-transitions.md Step 6:
   - Read current `state.json` for stage/status/feature/mode
   - Read last 20 entries from `.lifecycle/settings-changelog.md` (if exists)
   - Read last 10 entries from `.lifecycle/decisions.md` (if exists)
   - Read `.lifecycle/history/` recent entries for Event Timeline
   - Read `manifest.json` for stage completion timestamps
   - List ADRs from `docs/decisions/` (if directory exists)
   - Read `state.json` `session.resume_hint` for Resume Hint section
2. Write complete `.lifecycle/LIVING-STATE.md` using template structure from `$CLAUDE_SKILL_DIR/templates/living-state.md`
3. Include any new insights or lessons from the retrospective output

## Step 5: Completion

**Purpose:** Register outputs, update pipeline state, and announce completion.

**Actions:**

1. Register outputs in `manifest.json` under `artifacts.8_retrospect.outputs`:

```json
[
  {
    "type": "retrospective",
    "path": "docs/retrospectives/{feature-name}-retro.md",
    "created_at": "{ISO-8601}",
    "status": "complete"
  }
]
```

If ADRs were created, add each:
```json
{
  "type": "adr",
  "path": "docs/decisions/ADR-{NNN}-{title}.md",
  "created_at": "{ISO-8601}",
  "status": "complete"
}
```

2. Set `artifacts.8_retrospect.completed_at` = current ISO 8601 timestamp

3. Update `state.json`:
   - `current.status` = `"completed"`
   - `session.last_active` = current ISO 8601 timestamp
   - `session.resume_hint` = `"RETROSPECT complete. Ready for PROMOTE stage (optional)."`

4. Write history entry to `.lifecycle/history/`:

```json
{
  "timestamp": "{ISO-8601}",
  "from_stage": 7,
  "to_stage": 8,
  "gate_result": "passed",
  "missing_artifacts": [],
  "mode": "{current mode}"
}
```

5. Announce completion:

```
RETROSPECT Stage -- {feature-name} -- COMPLETE
===============================================
Retrospective: docs/retrospectives/{feature-name}-retro.md
ADRs created: {N} (or "none")
Work log: recorded

Ready for PROMOTE stage (optional -- skip is fine).
```

## Anti-Patterns

These are the most common mistakes. Do NOT do any of these:

| # | Anti-Pattern | Why It Is Wrong |
|---|-------------|-----------------|
| 1 | **Reimplementing retrospective generation** | The gsd-retrospective skill owns retro logic (templates, git analysis, metrics). This adapter invokes the skill -- it does not contain retro templates or analysis code. |
| 2 | **Auto-creating ADRs without user approval** | ADRs are formal architectural records. Each one should be deliberately created with user consent. Auto-creation leads to low-quality ADRs that pollute the decision record. |
| 3 | **Forgetting Living State update** | The Living State document is the project's memory. After a retrospective, it must be regenerated to capture new insights and updated stage completion data. |
| 4 | **Reimplementing work-log instead of invoking the skill** | The work-log skill owns formatting and storage location logic. This adapter provides context (feature name, timestamps) and delegates. |
| 5 | **Skipping ADR gap check** | Unrecorded decisions are lost context. Even if no gaps are found, the check itself is valuable and fast. |

## Relationship to Other Skills

- **dev-lifecycle:** Primary skill for Stage 8. This adapter orchestrates the RETROSPECT Stage workflow by delegating to specialized skills.
- **gsd-retrospective:** Invoked in Step 1 for retrospective generation. Owns retro templates, git log analysis, and metrics formatting.
- **ADR:** Invoked in Step 2 when user approves ADR creation for identified gaps. Owns ADR template and numbering.
- **work-log:** Invoked in Step 3 for work log recording. Owns log formatting and Obsidian vault integration.
- **GSD:** Not directly involved in Stage 8. GSD handles project planning, not feature retrospectives.

## File Locations

| File | Path | Purpose |
|------|------|---------|
| This adapter | `$CLAUDE_SKILL_DIR/references/retrospect-stage.md` | RETROSPECT Stage workflow instructions |
| Document adapter | `$CLAUDE_SKILL_DIR/references/document-stage.md` | Previous stage (gate 7_to_8 source) |
| Promote adapter | `$CLAUDE_SKILL_DIR/references/promote-stage.md` | Next stage (optional) |
| Stage transitions | `$CLAUDE_SKILL_DIR/references/stage-transitions.md` | Gate conditions (7_to_8, 8_to_9) |
| State template | `$CLAUDE_SKILL_DIR/templates/state.json` | State schema with stage/status fields |
| Manifest template | `$CLAUDE_SKILL_DIR/templates/manifest.json` | Artifact registry with gate rules |
| Living State template | `$CLAUDE_SKILL_DIR/templates/living-state.md` | Template for LIVING-STATE.md regeneration |
| Retro output | `docs/retrospectives/{feature}-retro.md` | Generated retrospective document |

**Runtime location:** `$CLAUDE_SKILL_DIR` = `~/.claude/skills/dev-lifecycle`
**Version-controlled location:** `skill/` directory in the project repository
