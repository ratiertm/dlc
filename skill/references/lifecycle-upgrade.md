# Lifecycle Upgrade Procedure

Migration algorithm for upgrading `.lifecycle/` project directories when the skill version advances. Claude follows these instructions during session start when a version mismatch is detected.

## When Triggered

During Session Start (SKILL.md step 1b), after loading state.json:

1. Read `$CLAUDE_SKILL_DIR/VERSION` (e.g., "2.0.0")
2. Read `.lifecycle/.dlc-version` (or assume "1.0.0" if file is missing)
3. If skill version > lifecycle version: execute this migration procedure
4. If versions match: skip migration entirely

**Important:** Only runs when `.lifecycle/` directory exists. Fresh installs (no `.lifecycle/`) skip migration -- the normal initialization flow creates files at current-version schema.

## Pre-Migration Backup

Before any migration writes, create a safety backup.

1. If `.lifecycle/.upgrade-backup/` already exists (leftover from a prior failed migration): remove it entirely
2. Create `.lifecycle/.upgrade-backup/` directory
3. Copy these files into the backup directory (skip any that do not exist):
   - `state.json`
   - `manifest.json`
   - `config.yaml`
   - `settings-changelog.md`
   - `decisions.md`
   - `LIVING-STATE.md`
4. If backup creation fails for any reason: **abort migration**, display error, continue session with old schema

## Migration Registry

Ordered, idempotent migration blocks. Each block has a marker file that prevents re-execution. When upgrading across multiple versions (e.g., v1.0 to v3.0), blocks run sequentially in order.

### v1.0 -> v2.0

**Marker:** `.lifecycle/.v2-migrated`

**Pre-check:** If `.lifecycle/.v2-migrated` exists, SKIP this entire block.

**Steps** (each is idempotent -- safe to re-run):

#### Step 1: state.json schema update

- Read `.lifecycle/state.json`
- If `current.completeness` field is missing: add `"completeness": null` to the `current` object
- If `version` field is `"1.0"`: change to `"2.0"`
- **Preserve ALL existing fields** -- `stage`, `stage_name`, `status`, `feature`, `mode`, `progress`, `session` must remain unchanged
- Write the updated state.json back

#### Step 2: config.yaml creation

- If `.lifecycle/config.yaml` does NOT exist: copy from `$CLAUDE_SKILL_DIR/templates/config.yaml`
- If it already exists: leave unchanged

#### Step 3: settings-changelog.md creation

- If `.lifecycle/settings-changelog.md` does NOT exist: copy from `$CLAUDE_SKILL_DIR/templates/settings-changelog.md`
- If it already exists: leave unchanged

#### Step 4: decisions.md creation

- If `.lifecycle/decisions.md` does NOT exist: copy from `$CLAUDE_SKILL_DIR/templates/decisions.md`
- If it already exists: leave unchanged

#### Step 5: Version tracking

- Write `"2.0.0"` to `.lifecycle/.dlc-version`
- Create `.lifecycle/.v2-migrated` (empty marker file)

## Rollback Procedure

If ANY step in the migration block fails:

1. Restore all files from `.lifecycle/.upgrade-backup/` (overwrite current versions)
2. Delete `.lifecycle/.upgrade-backup/` directory
3. Do NOT create the `.v2-migrated` marker
4. Display to user:
   ```
   Migration to v2.0 failed. Rolled back to previous state.
   Error: {description of what failed}
   ```
5. Continue session with old schema (best effort)

## Post-Migration Cleanup

On successful completion of all migration blocks:

1. Delete `.lifecycle/.upgrade-backup/` directory
2. Display changelog summary (see next section)

## Changelog Display

After successful migration, show the user what changed:

```
dev-lifecycle upgraded: v{old} -> v{new}

What's new:
- [Read skill/CHANGELOG.md, extract Added/Changed bullets between old and new version headers]

Continuing session...
```

**Example** (v1.0.0 -> v2.0.0):
```
dev-lifecycle upgraded: v1.0.0 -> v2.0.0

What's new:
- Configuration system: lifecycle-config get/set/list with 3-layer resolution
- Mini-verify loops: each spec step verified immediately in DO stage
- Completeness scoring: N/10 quality score at every stage completion
- Safe upgrade: automatic schema migration with rollback safety

Continuing session...
```

## Future Migration Blocks

To add a new version migration:

1. Add a new section after the last migration block: `### v2.0 -> v3.0`
2. Define a unique marker file (e.g., `.lifecycle/.v3-migrated`)
3. List idempotent migration steps
4. Blocks run sequentially: a user upgrading from v1.0 to v3.0 runs v1.0->v2.0 first, then v2.0->v3.0
5. Each block's pre-check (marker file) ensures it only runs once

## Edge Cases

| Scenario | Behavior |
|----------|----------|
| No `.lifecycle/` directory | Skip migration entirely (fresh install) |
| `.dlc-version` missing | Assume v1.0.0 (pre-upgrade install) |
| `.v2-migrated` exists but `.dlc-version` missing | Write `.dlc-version` = "2.0.0", skip migration steps |
| User mid-stage (e.g., DO in progress) | Migration preserves `current.stage`, `status`, `feature` -- no disruption |
| `.upgrade-backup/` already exists from prior failed migration | Remove it first, then proceed with fresh backup |
| `state.json` missing entirely | Let state-reconcile.md handle recovery first, then run migration on the reconciled file |
| `manifest.json` missing | Not affected by v1.0->v2.0 migration (schema unchanged); skip backup of missing file |

## Interaction with State Reconcile

If state.json is missing at session start, the state-reconcile procedure (see `references/state-reconcile.md`) runs first to reconstruct it. Migration then runs on the reconciled file. Order:

1. Session Start detects missing state.json
2. State reconcile creates state.json from manifest.json
3. Version check detects mismatch
4. Migration updates the reconciled state.json to current schema
