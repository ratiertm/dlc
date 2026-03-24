# Phase 13: Lifecycle Upgrade - Research

**Researched:** 2026-03-25
**Domain:** Schema migration, version management, rollback safety for Claude Code skill files
**Confidence:** HIGH

## Summary

Phase 13 implements a safe upgrade mechanism for dev-lifecycle. The core challenge is that `.lifecycle/` directories in user projects contain state files (state.json, config.yaml, manifest.json, etc.) whose schemas evolve across versions. When the skill itself is updated via `git pull` (symlink install) or re-copy (vendored), existing project `.lifecycle/` files may be incompatible with the new skill version.

The gstack project provides a proven reference pattern: migration markers prevent re-running, `.bak` directories enable rollback, CHANGELOG.md provides user-facing summaries, and a VERSION file tracks the current skill version. Our implementation adapts this pattern to dev-lifecycle's specific concerns -- primarily the `.lifecycle/` directory schema rather than the skill installation itself.

**Primary recommendation:** Create a `skill/references/lifecycle-upgrade.md` reference file containing the full migration algorithm, a `skill/CHANGELOG.md` for version history, a `skill/VERSION` file, and add a brief Upgrade section to SKILL.md that triggers migration check on session start when version mismatch is detected.

<phase_requirements>
## Phase Requirements

| ID | Description | Research Support |
|----|-------------|-----------------|
| UPGR-01 | User can upgrade dev-lifecycle via git pull with automatic schema migration of .lifecycle/ files | Version detection + migration registry pattern; session-start hook triggers migration when skill VERSION > .lifecycle/ version |
| UPGR-02 | Migration markers (.lifecycle/.v2-migrated etc.) prevent re-running completed migrations | Marker file pattern from gstack (.codex-desc-healed); idempotent marker check before each migration step |
| UPGR-03 | Failed migration can be rolled back to pre-upgrade state | Pre-migration backup to .lifecycle/.upgrade-backup/; restore on failure |
| UPGR-04 | Upgrade shows changelog summary of what changed | skill/CHANGELOG.md + version diff display pattern from gstack |
</phase_requirements>

## Standard Stack

This phase does not introduce external libraries. Everything is file-based logic within the Claude Code skill system (markdown references read by Claude, not executable code).

### Core Files to Create/Modify

| File | Purpose | Why |
|------|---------|-----|
| `skill/VERSION` | Single source of truth for current skill version | gstack pattern; simple `cat` to read |
| `skill/CHANGELOG.md` | Human-readable version history | gstack pattern; Claude reads and summarizes for user |
| `skill/references/lifecycle-upgrade.md` | Full migration algorithm and registry | Progressive disclosure -- keeps SKILL.md slim |
| `skill/SKILL.md` (modify) | Add version check to Session Start + brief Upgrade section | Entry point for migration trigger |

### Files to Modify in .lifecycle/ (per-project)

| File | Migration Needed | v1.0 -> v2.0 Changes |
|------|-----------------|---------------------|
| `state.json` | YES | Add `current.completeness` field (Phase 12); version field "1.0" -> "2.0" |
| `config.yaml` | CREATE if missing | Did not exist in v1.0; copy from template |
| `manifest.json` | NO | Schema unchanged |
| `settings-changelog.md` | CREATE if missing | Did not exist in v1.0; copy from template |
| `decisions.md` | CREATE if missing | Did not exist in v1.0; copy from template |
| `LIVING-STATE.md` | NO | Regenerated on next stage transition |
| `.lifecycle/.dlc-version` | CREATE | Tracks which version the .lifecycle/ was last migrated to |

## Architecture Patterns

### Recommended Structure

```
skill/
  VERSION                          # "2.0.0" (single line)
  CHANGELOG.md                     # Version history
  references/
    lifecycle-upgrade.md           # Full migration algorithm (NEW)
  templates/
    (existing templates unchanged)

.lifecycle/                        # Per-project (modified during migration)
  .dlc-version                     # "2.0.0" -- tracks migrated version
  .v2-migrated                     # Migration marker (UPGR-02)
  .upgrade-backup/                 # Temporary backup during migration (UPGR-03)
    state.json.bak
    manifest.json.bak
    ...
```

### Pattern 1: Version-Triggered Migration on Session Start

**What:** During Session Start (SKILL.md step 1), after loading state.json, compare skill VERSION against `.lifecycle/.dlc-version`. If skill version is newer, trigger migration.

**When to use:** Every session start where `.lifecycle/` exists.

**Flow:**
```
Session Start
  -> Load state.json (existing step 1)
  -> Read skill/VERSION
  -> Read .lifecycle/.dlc-version (or assume "1.0.0" if missing)
  -> If skill_version > lifecycle_version:
       -> Read: $CLAUDE_SKILL_DIR/references/lifecycle-upgrade.md
       -> Execute migration
       -> Show changelog summary
  -> Continue normal session
```

### Pattern 2: Migration Registry (Ordered, Idempotent)

**What:** A registry of migration steps, each with a marker file. Steps run in order, skip if marker exists.

**Example migration registry (in lifecycle-upgrade.md):**
```
Migration Registry:
  v2.0:
    marker: .lifecycle/.v2-migrated
    steps:
      1. Add current.completeness to state.json (if missing)
      2. Update state.json version field to "2.0"
      3. Create config.yaml from template (if not exists)
      4. Create settings-changelog.md from template (if not exists)
      5. Create decisions.md from template (if not exists)
      6. Write .lifecycle/.dlc-version = "2.0.0"
      7. Touch .lifecycle/.v2-migrated
```

Each migration block is idempotent: individual steps check "if not exists" or "if field missing" before acting. The marker file at the end prevents the entire block from re-running.

### Pattern 3: Backup-Before-Migrate (Rollback Safety)

**What:** Before any migration writes, copy all `.lifecycle/` files to `.lifecycle/.upgrade-backup/`. On failure, restore. On success, remove backup.

**Flow:**
```
1. Create .lifecycle/.upgrade-backup/
2. Copy state.json, manifest.json, config.yaml (if exists) to backup
3. Run migration steps
4. If any step fails:
   a. Restore all files from .upgrade-backup/
   b. Remove .upgrade-backup/
   c. Display error: "Migration failed. Rolled back to previous state."
   d. Continue session with old schema (best effort)
5. If all succeed:
   a. Remove .upgrade-backup/
   b. Display: "Migrated .lifecycle/ to v2.0"
```

### Pattern 4: Changelog Summary (gstack pattern)

**What:** After migration, read skill/CHANGELOG.md, find entries between old and new version, summarize 3-5 bullets of user-facing changes.

**Format:**
```
dev-lifecycle v2.0.0 -- upgraded from v1.0.0!

What's new:
- Configuration system: manage settings via lifecycle-config get/set/list
- Mini-verify: each spec step is verified immediately during DO stage
- Completeness scoring: every stage reports quality as N/10
- Rollback safety: failed migrations restore previous state

Continuing session...
```

### Anti-Patterns to Avoid

- **Executable migration scripts:** dev-lifecycle is a Claude Code skill (markdown only). Migrations are instructions Claude follows, not bash scripts. Do not create shell scripts for migration.
- **Breaking existing sessions:** Migration must preserve all in-progress state. A user mid-feature (e.g., stage 3 TEST) should continue seamlessly after upgrade.
- **Version in state.json as sole tracker:** state.json version "1.0" is the schema version of the state file, not the skill version. Use a separate `.dlc-version` file (like gstack's VERSION file pattern) to track what version migrated the .lifecycle/ directory.
- **Destructive migration:** Never delete user data. Adding fields is safe; removing/renaming fields requires copying old value to new location first.

## Don't Hand-Roll

| Problem | Don't Build | Use Instead | Why |
|---------|-------------|-------------|-----|
| Version comparison | Semantic version parser | Simple string comparison or ordered version list | Only comparing known versions (1.0.0, 2.0.0); no need for semver parsing |
| Config file creation | Custom YAML writer | Copy template file | Templates already exist in skill/templates/ |
| Schema evolution | JSON schema validation | Additive field checks ("if field missing, add default") | State files are simple JSON; additive-only changes are trivially safe |
| Changelog parsing | Complex markdown parser | Claude reads CHANGELOG.md and summarizes (it's a Claude skill) | Claude is already the runtime; let it read and summarize naturally |

**Key insight:** This is a Claude Code skill, not executable software. "Migration" means Claude reading instructions and editing JSON/YAML files per those instructions. The complexity is in getting the instructions right, not in building migration infrastructure.

## Common Pitfalls

### Pitfall 1: Migration Runs During Active Stage Work
**What goes wrong:** User is mid-DO stage, upgrades skill, starts new session. Migration modifies state.json while stage work is conceptually "in progress."
**Why it happens:** Migration doesn't check current stage status.
**How to avoid:** Migration must preserve `current.stage`, `current.status`, `current.feature`, and all `progress` fields. Only add new fields; never reset existing values.
**Warning signs:** state.json `current.status` changes to "not_started" after migration when it should be "in_progress."

### Pitfall 2: Local Install vs Symlink Divergence
**What goes wrong:** setup.sh offers two install modes: global symlink (git pull updates immediately) and local copy (stale until re-copied). Migration assumes new skill version but .lifecycle/ was created by old skill version.
**Why it happens:** Local copy users don't get automatic updates.
**How to avoid:** Version check compares skill/VERSION (from the installed skill, wherever it lives) against .lifecycle/.dlc-version. Works regardless of install method.
**Warning signs:** Migration reference file not found because skill is old version but .lifecycle/ has new marker.

### Pitfall 3: Missing .lifecycle/ Files Treated as Error
**What goes wrong:** v1.0 projects don't have config.yaml, settings-changelog.md, or decisions.md. Migration tries to "update" them and fails.
**Why it happens:** Migration assumes all files exist.
**How to avoid:** Each migration step uses "create if not exists" pattern, not "update existing." Template copy is the safe default for missing files.
**Warning signs:** File not found errors during migration.

### Pitfall 4: SKILL.md Line Budget Exceeded
**What goes wrong:** Adding upgrade logic to SKILL.md pushes it over 500 lines.
**Why it happens:** Trying to inline migration logic instead of using references/.
**How to avoid:** SKILL.md gets only 3-5 lines in Session Start (version check + Read directive) and 5-10 lines for a brief Upgrade section. All logic lives in `skill/references/lifecycle-upgrade.md`.
**Warning signs:** SKILL.md exceeds 370 lines (current: 355, budget for phase 13: ~15 lines max).

### Pitfall 5: Backup Directory Left Behind
**What goes wrong:** `.upgrade-backup/` persists after successful migration, confusing future state reconcile.
**Why it happens:** Cleanup step forgotten or error path doesn't clean up.
**How to avoid:** Always remove `.upgrade-backup/` in both success and failure paths. State reconcile should also ignore this directory.

## Code Examples

### state.json v1.0 -> v2.0 Migration

Before (v1.0):
```json
{
  "version": "1.0",
  "project": { "name": "", "type": "auto", "detected_type": null, "confirmed": false },
  "current": {
    "stage": 2,
    "stage_name": "DO",
    "status": "in_progress",
    "feature": "user-auth",
    "mode": "feature"
  },
  "progress": {
    "stages_completed": [1],
    "current_stage_started_at": "2026-03-20T10:00:00Z",
    "last_transition_at": "2026-03-20T09:55:00Z"
  },
  "session": {
    "last_active": "2026-03-20T11:00:00Z",
    "resume_hint": "Implementing step 3 of spec"
  }
}
```

After (v2.0 migration applied):
```json
{
  "version": "2.0",
  "project": { "name": "", "type": "auto", "detected_type": null, "confirmed": false },
  "current": {
    "stage": 2,
    "stage_name": "DO",
    "status": "in_progress",
    "feature": "user-auth",
    "mode": "feature",
    "completeness": null
  },
  "progress": {
    "stages_completed": [1],
    "current_stage_started_at": "2026-03-20T10:00:00Z",
    "last_transition_at": "2026-03-20T09:55:00Z"
  },
  "session": {
    "last_active": "2026-03-20T11:00:00Z",
    "resume_hint": "Implementing step 3 of spec"
  }
}
```

Changes: `version` "1.0" -> "2.0", `current.completeness` added (null until first stage completion with new skill).

### .dlc-version File

```
2.0.0
```

Single line, no trailing content. Read with `cat .lifecycle/.dlc-version`.

### CHANGELOG.md Format

```markdown
# Dev Lifecycle Changelog

## [2.0.0] - 2026-03-25

### Added
- **Configuration system** (Phase 11): lifecycle-config get/set/list with 3-layer resolution
- **Mini-verify loops** (Phase 12): each spec step verified immediately in DO stage
- **Completeness scoring** (Phase 12): N/10 quality score at every stage completion
- **Safe upgrade** (Phase 13): automatic schema migration with rollback safety
- **config.yaml**: file-based settings with environment variable overrides
- **settings-changelog.md**: automatic change tracking with reasons
- **decisions.md**: lightweight decision log below ADR threshold

### Changed
- state.json version "1.0" -> "2.0" (new `current.completeness` field)
- Mode resolution now uses config layers (env > config.yaml > state.json)
- DO stage pipeline expanded with mini-verify sub-steps (2d.1/2d.2)

## [1.0.0] - 2026-03-22

### Added
- 9-stage pipeline (PLAN through PROMOTE)
- E2E spec format with 5-step chain
- Clickable HTML prototypes
- State management (state.json, manifest.json)
- Living State Document for session restoration
- 4 execution modes (hotfix, feature, release, milestone)
- Stage transition gates with artifact verification
- Memory & Decision Trail (settings-changelog, decisions, WHY+SEE comments)
- State reconcile for recovery from state.json loss
```

### Session Start Version Check (SKILL.md addition)

```markdown
## Session Start

On session start (or after compaction):

1. **Read state:** Load `.lifecycle/state.json`
   - (existing logic unchanged)
1b. **Version check:** Compare `$CLAUDE_SKILL_DIR/VERSION` against `.lifecycle/.dlc-version`
   - If `.dlc-version` missing or older: Read `$CLAUDE_SKILL_DIR/references/lifecycle-upgrade.md` and run migration
   - If versions match: continue normally
2. **Read Living State (if exists):** ...
```

## State of the Art

| Old Approach | Current Approach | When Changed | Impact |
|--------------|------------------|--------------|--------|
| No versioning | VERSION file + .dlc-version tracking | Phase 13 | Enables safe migration between any versions |
| Manual file creation | Template copy on first use | Phase 11 (config.yaml) | Consistent initialization pattern reused by migration |
| No upgrade path | Migration registry + markers | Phase 13 | Users on v1.0 can safely upgrade to v2.0 |

## Validation Architecture

### Test Framework

| Property | Value |
|----------|-------|
| Framework | Manual verification (Claude Code skill -- no executable test framework) |
| Config file | N/A |
| Quick run command | Create a mock v1.0 .lifecycle/ directory and run session start |
| Full suite command | Test all migration paths: fresh install, v1.0->v2.0, already-migrated, mid-stage migration |

### Phase Requirements -> Test Map

| Req ID | Behavior | Test Type | Automated Command | File Exists? |
|--------|----------|-----------|-------------------|-------------|
| UPGR-01 | git pull + auto-migration | manual | Create v1.0 .lifecycle/, update skill, start session, verify v2.0 fields | N/A |
| UPGR-02 | Marker prevents re-run | manual | Run migration, verify .v2-migrated exists, start new session, verify migration skipped | N/A |
| UPGR-03 | Rollback on failure | manual | Simulate corrupt state during migration, verify backup restored | N/A |
| UPGR-04 | Changelog summary shown | manual | Upgrade from v1.0, verify changelog bullets displayed | N/A |

### Sampling Rate
- **Per task commit:** Read modified reference files, verify internal consistency
- **Per wave merge:** Walk through full v1.0 -> v2.0 migration scenario mentally
- **Phase gate:** Manual end-to-end test with mock .lifecycle/ directory

### Wave 0 Gaps
None -- this is a Claude Code skill with markdown-based "tests" (manual verification scenarios). No test framework infrastructure needed.

## Open Questions

1. **Future version migrations (v2.0 -> v3.0)**
   - What we know: The migration registry pattern supports adding new version blocks
   - What's unclear: Whether we need a migration runner that chains v1.0->v2.0->v3.0 sequentially for users who skip versions
   - Recommendation: Design the registry to support sequential chaining but only implement v1.0->v2.0 for now. Each migration block runs independently with its own marker.

2. **Template version sync**
   - What we know: Templates in `skill/templates/` should match the current version
   - What's unclear: Whether templates need their own version markers
   - Recommendation: Templates are always "current version." Migration creates files from templates for missing files, which is inherently current-version. No template versioning needed.

## Sources

### Primary (HIGH confidence)
- gstack-upgrade/SKILL.md -- complete upgrade flow pattern (version detection, backup, restore, changelog display)
- gstack bin/gstack-update-check -- migration marker pattern (.codex-desc-healed)
- skill/SKILL.md (355 lines) -- current session start flow, line budget constraints
- skill/templates/state.json -- current v1.0 schema
- skill/templates/config.yaml -- v2.0 addition (Phase 11)
- skill/references/state-reconcile.md -- existing recovery pattern (informs migration safety)
- skill/references/lifecycle-config.md -- config resolution algorithm (migration must preserve)

### Secondary (MEDIUM confidence)
- .planning/ROADMAP.md -- Phase 13 dependencies and success criteria
- .planning/STATE.md -- SKILL.md line budget (355/500, ~145 remaining)
- setup.sh -- install mechanisms (symlink vs copy) that affect upgrade delivery

## Metadata

**Confidence breakdown:**
- Standard stack: HIGH -- no external dependencies; file-based patterns well understood from gstack
- Architecture: HIGH -- gstack-upgrade provides proven reference; adapted to dev-lifecycle's markdown-only constraint
- Pitfalls: HIGH -- identified from analyzing current codebase (state.json schema, install modes, line budget)

**Research date:** 2026-03-25
**Valid until:** 2026-04-25 (stable domain -- skill file patterns don't change rapidly)
