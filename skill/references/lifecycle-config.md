# Lifecycle Configuration Reference

Config operations reference for dev-lifecycle. Handles `lifecycle-config get|set|list` commands with 3-layer resolution and automatic change tracking.

## Settings Table

| Setting | Type | Default | Valid Values | Description |
|---------|------|---------|-------------|-------------|
| `mode` | string | `feature` | `hotfix\|feature\|release\|milestone` | Execution mode determining which stages are required |
| `skip_stages` | list[int] | `[]` | Stage numbers 1-9 | Explicit stage skip overrides (additive with mode defaults) |
| `proactive` | boolean | `true` | `true\|false` | Suggest companion skills at stage transitions |
| `auto_skip` | boolean | `true` | `true\|false` | Auto-skip skippable stages without prompting |
| `verification_strictness` | string | `standard` | `strict\|standard\|relaxed` | How strictly to enforce spec verification |

## Resolution Algorithm

3-layer precedence. First match wins:

1. **Environment variable:** `LIFECYCLE_{SETTING_UPPER}` (e.g., `LIFECYCLE_MODE`, `LIFECYCLE_SKIP_STAGES`)
   - Check: `echo "${LIFECYCLE_MODE:-}"`
   - If non-empty, use this value
2. **Config file:** `.lifecycle/config.yaml`
   - Read the file, find the key
   - If file doesn't exist, skip to layer 3
3. **Built-in defaults:** Hardcoded defaults from Settings Table above

To resolve setting `{key}`:
1. Run `echo "${LIFECYCLE_{KEY_UPPER}:-}"` -- if non-empty, use this value
2. If empty, read `.lifecycle/config.yaml`, find `{key}:` line, extract value
3. If not found or file missing, use default from Settings Table

## Operations

### `lifecycle-config get {key}`

1. Validate key is one of the 5 known settings (reject unknown keys)
2. Resolve value using Resolution Algorithm
3. Display: `{key}: {value} (source: {environment|config.yaml|default})`

### `lifecycle-config set {key} {value}`

1. Validate key is known, value is valid for that key's type/constraints (see Validation Rules)
2. Read current resolved value (for changelog old -> new)
3. If `.lifecycle/config.yaml` doesn't exist, copy from `$CLAUDE_SKILL_DIR/templates/config.yaml`
4. Write new value to `.lifecycle/config.yaml`
5. Ask user for reason (one-liner)
6. **Mandatory changelog:** Append to `.lifecycle/settings-changelog.md`:
   `| {date} | .lifecycle/config.yaml | {key}: {old} -> {new} | {reason} |`
   If settings-changelog.md doesn't exist, create from `$CLAUDE_SKILL_DIR/templates/settings-changelog.md` first
7. If key is `mode`: also update `state.json` `current.mode` for backward compatibility
8. Confirm: `{key} set to {value} (was: {old})`

Step 6 is MANDATORY on every set operation. This fulfills CONF-03.

### `lifecycle-config list`

1. For each of the 5 settings, resolve value using Resolution Algorithm
2. Display table format:

```
Dev Lifecycle Configuration:

  mode:                      feature     (default)
  skip_stages:               []          (config.yaml)
  proactive:                 true        (default)
  auto_skip:                 true        (default)
  verification_strictness:   standard    (default)

Config file: .lifecycle/config.yaml (exists|not found)
Env overrides: {list or "none detected"}
```

## Validation Rules

| Setting | Rule |
|---------|------|
| `mode` | Must be one of: `hotfix`, `feature`, `release`, `milestone` |
| `skip_stages` | List of integers 1-9 |
| `proactive` | Must be `true` or `false` |
| `auto_skip` | Must be `true` or `false` |
| `verification_strictness` | Must be one of: `strict`, `standard`, `relaxed` |

- Unknown keys: reject on `set`, ignore on read
- Invalid values: reject on `set` with explanation of valid values

## Edge Cases

- **Config file doesn't exist:** Resolution falls through to defaults. `list` shows all defaults. `get` returns default. First `set` copies template then writes.
- **skip_stages interaction with mode:** Union (additive). `skip_stages` adds to mode's skippable stages, never removes required stages. Example: mode=feature skips [5,6,9]; skip_stages=[1] results in skipping [1,5,6,9].
- **auto_skip scope:** Applies to all skippable stages (both mode-based and explicit skip_stages). When `false`, user is prompted for each skippable stage.
- **Env var for skip_stages:** Comma-separated integers, e.g., `LIFECYCLE_SKIP_STAGES="5,6,9"`
- **Env var for booleans:** String `"true"` or `"false"` (e.g., `LIFECYCLE_PROACTIVE="false"`)
- **verification_strictness levels:**
  - `strict`: All spec steps must pass, no overrides allowed
  - `standard`: All must pass, but user can override with justification
  - `relaxed`: Warnings only, no blocking

## Integration Notes

- **Config is source of truth for mode.** `state.json` reflects it (updated on `set`). On session start, resolve mode from config, write to `state.json` for backward compatibility.
- **settings-changelog.md** tracks both project config changes and lifecycle config changes (same format, same rules).
- **Template copy on first set:** Config file is created from `$CLAUDE_SKILL_DIR/templates/config.yaml` only when a `set` operation requires it. The system works with zero configuration via defaults.
