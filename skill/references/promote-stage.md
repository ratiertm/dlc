# PROMOTE Stage Adapter

This reference defines the complete PROMOTE Stage workflow -- Stage 9 of the dev-lifecycle pipeline. When Claude reads this file, it follows the instructions below to optionally guide the user through creating a demo or announcement for their completed feature. Skip is the default path.

## When This Runs

- User says "promote", "demo", "announce", "share", or similar
- `state.json` `current.stage = 9` (or transitioning from 8 to 9)
- SKILL.md Stage 9 section directs here via `Read: $CLAUDE_SKILL_DIR/references/promote-stage.md`

## Prerequisites

- `.lifecycle/` directory initialized (from PLAN stage)
- `state.json` and `manifest.json` exist
- RETROSPECT stage must have outputs -- gate 8_to_9 must pass (`artifacts.8_retrospect.outputs.length > 0`)

## Step 0: Stage Initialization

Before any promotion work, verify the pipeline state and check mode-based skipping.

**Actions:**

1. Read `.lifecycle/state.json` and verify:
   - `current.feature` has a value
   - `current.stage` = `9` (or transition from 8 to 9)

2. Check gate condition (8_to_9):
   - Read `.lifecycle/manifest.json`
   - Verify: `artifacts.8_retrospect.outputs.length > 0`
   - If gate fails: set `current.status = "blocked"`, display missing artifacts, STOP

3. Check mode-based skip:
   - Read `current.mode` from `state.json`
   - If mode is `hotfix`, `feature`, or `release`: this stage is skippable
   - Only `milestone` mode requires PROMOTE
   - For all skippable modes: proceed to Step 1 (Skip Check) immediately

4. Update `state.json`:
   - `current.stage` = `9`
   - `current.stage_name` = `"PROMOTE"`
   - `current.status` = `"in_progress"`
   - `progress.current_stage_started_at` = current ISO 8601 timestamp

## Step 1: Skip Check

**Purpose:** Ask the user whether they want to create promotional content. Skip is the default.

**Actions:**

1. Display prompt:
   ```
   PROMOTE Stage -- {feature-name}
   ================================
   Would you like to create a demo or announcement?
   This stage is optional -- skip is perfectly fine.

   Options:
     1. Skip (default -- press Enter)
     2. Create demo/announcement
   ```

2. If user chooses skip (or presses Enter / says "skip" / says "no"):
   - Register skipped artifact in `manifest.json` under `artifacts.9_promote.outputs`:
     ```json
     [{ "type": "skipped", "path": "N/A", "status": "skipped" }]
     ```
   - Set `artifacts.9_promote.completed_at` = current ISO 8601 timestamp
   - Set `current.status` = `"skipped"` in `state.json`
   - Write history entry to `.lifecycle/history/`
   - Regenerate `.lifecycle/LIVING-STATE.md`
   - Announce pipeline completion (see completion announcement below)
   - STOP

3. If user wants to proceed: continue to Step 2

## Step 2: Demo Type Selection

**Purpose:** Let the user choose what type of promotional content to create.

**Actions:**

1. Display options based on project type:

   ```
   What type of demo would you like to create?

     1. Screen recording (web apps, desktop tools)
     2. Mobile recording via Maestro + ADB + FFmpeg (mobile apps)
     3. Screenshots with annotations
     4. Written announcement (changelog, blog post, Slack message)
   ```

2. Record user's selection
3. Continue to Step 3 with selected type

## Step 3: Demo Creation Guidance

**Purpose:** Provide tool-specific instructions for the selected demo type. Claude guides -- Claude does NOT automate recording.

**Actions:**

### Screen Recording (Option 1)
- Suggest tools: OBS Studio, QuickTime (macOS), built-in screen recorder
- Guide user through: open app, start recording, walk through key flow, stop recording
- Suggest output location: `docs/demos/{feature-name}-demo.mp4` or upload to hosting

### Mobile Recording (Option 2)
- For Android: `adb shell screenrecord /sdcard/demo.mp4` or Maestro flow recording
- For iOS: QuickTime via USB or Xcode screen capture
- Guide user through recording the key mobile flow

### Screenshots (Option 3)
- Guide user through capturing key screens/states
- Suggest annotation tools if needed
- Output: `docs/demos/{feature-name}-screenshots/`

### Written Announcement (Option 4)
- Generate a draft based on:
  - Feature spec (what was built)
  - Retrospective (what was learned)
  - Deploy info (where it is live)
- Format options: changelog entry, blog post draft, Slack/Teams message
- Present draft for user editing

## Step 4: Completion

**Purpose:** Register outputs, finalize pipeline state, and announce full pipeline completion.

**Actions:**

1. Register outputs in `manifest.json` under `artifacts.9_promote.outputs`:

```json
[
  {
    "type": "demo",
    "path": "{path-to-demo-artifact}",
    "created_at": "{ISO-8601}",
    "status": "complete"
  }
]
```

Or for written announcement:
```json
[
  {
    "type": "announcement",
    "path": "{path-to-announcement}",
    "created_at": "{ISO-8601}",
    "status": "complete"
  }
]
```

2. Set `artifacts.9_promote.completed_at` = current ISO 8601 timestamp

3. Update `state.json`:
   - `current.status` = `"completed"`
   - `session.last_active` = current ISO 8601 timestamp
   - `session.resume_hint` = `"Pipeline complete. Ready for next feature."`

4. Write history entry to `.lifecycle/history/`

5. Regenerate `.lifecycle/LIVING-STATE.md`

6. Announce full pipeline completion:

```
PROMOTE Stage -- {feature-name} -- COMPLETE
============================================
{Demo/Announcement}: {path or "skipped"}

PIPELINE COMPLETE
=================
Feature: {feature-name}
Mode: {mode}
Stages completed: {list}
Stages skipped: {list or "none"}

Ready for next feature. Run PLAN stage to begin a new cycle.
```

## Anti-Patterns

These are the most common mistakes. Do NOT do any of these:

| # | Anti-Pattern | Why It Is Wrong |
|---|-------------|-----------------|
| 1 | **Treating PROMOTE as required** | PROMOTE is optional for all modes except milestone. Skip is the default path. Never guilt or pressure the user into creating promotional content. |
| 2 | **No skip path in flow** | The skip path must be the first option presented, clearly marked as default. Without it, users feel forced to create demos for every feature. |
| 3 | **Trying to automate screen/mobile recording** | Claude cannot control screen recording tools. This stage provides guidance and tracks completion -- the user operates the recording tools. |
| 4 | **Blocking pipeline completion on demo creation** | The pipeline is complete once RETROSPECT finishes. PROMOTE adds optional polish. A skipped PROMOTE with a registered "skipped" artifact is a fully valid pipeline completion. |

## Relationship to Other Skills

- **dev-lifecycle:** Primary skill for Stage 9. This adapter orchestrates the PROMOTE Stage workflow and manages the optional demo/announcement flow.
- **gsd-retrospective:** Provides retrospective output that informs written announcements (what was learned, what went well).
- **GSD:** Not directly involved in Stage 9.

## File Locations

| File | Path | Purpose |
|------|------|---------|
| This adapter | `$CLAUDE_SKILL_DIR/references/promote-stage.md` | PROMOTE Stage workflow instructions |
| Retrospect adapter | `$CLAUDE_SKILL_DIR/references/retrospect-stage.md` | Previous stage (gate 8_to_9 source) |
| Stage transitions | `$CLAUDE_SKILL_DIR/references/stage-transitions.md` | Gate conditions (8_to_9) |
| State template | `$CLAUDE_SKILL_DIR/templates/state.json` | State schema with stage/status fields |
| Manifest template | `$CLAUDE_SKILL_DIR/templates/manifest.json` | Artifact registry with gate rules |

**Runtime location:** `$CLAUDE_SKILL_DIR` = `~/.claude/skills/dev-lifecycle`
**Version-controlled location:** `skill/` directory in the project repository
