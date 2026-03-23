# Phase 11: Configuration - Research

**Researched:** 2026-03-24
**Domain:** File-based configuration system for Claude Code skill (no server, no CLI binary)
**Confidence:** HIGH

## Summary

Phase 11 introduces a unified configuration system for dev-lifecycle, inspired by gstack's `gstack-config get/set` pattern but adapted for the Claude Code skill constraint: no shell scripts, no CLI binaries -- instructions only in SKILL.md and references/. The config system must resolve settings through three layers (env var > config file > defaults) and automatically log changes to the existing settings-changelog.

The key design challenge is that gstack uses a bash script (`bin/gstack-config`) for get/set operations, but dev-lifecycle operates purely through Claude's instruction-following. Configuration must be implemented as "Claude reads/writes YAML files according to instructions in references/" -- the same pattern used for state.json, manifest.json, and all other .lifecycle/ files.

**Primary recommendation:** Implement config as a new reference file (`references/lifecycle-config.md`) that instructs Claude how to read/write `.lifecycle/config.yaml` with layered resolution, plus update SKILL.md with a compact "Configuration" section (~20 lines) that points to the reference. Reuse the existing settings-changelog.md for change tracking (CONF-03).

<phase_requirements>
## Phase Requirements

| ID | Description | Research Support |
|----|-------------|-----------------|
| CONF-01 | User can get/set lifecycle settings via lifecycle-config pattern (mode, skip_stages, proactive, auto_skip, verification_strictness) | Config YAML schema, get/set instruction pattern, setting definitions |
| CONF-02 | Settings are layered -- environment variable > .lifecycle/config.yaml > defaults | Resolution algorithm with 3-layer precedence |
| CONF-03 | Config changes are automatically recorded in settings-changelog with reason | Existing settings-changelog template and DO/COMMIT stage integration |
</phase_requirements>

## Standard Stack

This phase does not introduce external libraries. Everything is file-based within the Claude Code skill framework.

### Core Components

| Component | Format | Purpose | Why This Way |
|-----------|--------|---------|-------------|
| `.lifecycle/config.yaml` | YAML | User-facing config file | YAML is human-readable, matches gstack pattern, already used in settings-changelog detection patterns |
| `references/lifecycle-config.md` | Markdown | Instructions for Claude to handle config operations | Same pattern as all other stage references (progressive disclosure) |
| `templates/config.yaml` | YAML | Default config with all settings and comments | Provides self-documenting defaults, copied on first use |
| `.lifecycle/settings-changelog.md` | Markdown table | Change tracking (already exists) | CONF-03 reuses existing mechanism from v1.0 MEMO-01 |

### Alternatives Considered

| Instead of | Could Use | Tradeoff |
|------------|-----------|----------|
| YAML config file | JSON config file | YAML is more human-readable with comments; JSON is what state.json uses. YAML wins because config is user-edited, state.json is machine-edited |
| Inline SKILL.md logic | Reference file | Reference keeps SKILL.md under 500 lines (currently 339, budget ~160). Config logic is ~80-100 lines of instructions |
| Separate config-changelog | Existing settings-changelog | Reuse existing file -- no need for a second changelog. Just ensure lifecycle config changes are logged there too |

## Architecture Patterns

### Recommended File Structure

```
skill/
  references/
    lifecycle-config.md    # NEW: config get/set/list instructions
  templates/
    config.yaml            # NEW: default config with comments

.lifecycle/                # (per-project, created at runtime)
  config.yaml              # User's actual config (created from template)
  settings-changelog.md    # Change log (already exists from v1.0)
```

### Pattern 1: Config Resolution Algorithm (3-Layer Precedence)

**What:** When Claude needs a config value, it resolves through three layers in order.
**When to use:** Every time a config-dependent behavior is triggered (mode check, skip decision, proactive suggestion, etc.)

```
Resolution order (first match wins):
1. Environment variable: LIFECYCLE_{SETTING_NAME} (uppercase, underscored)
   - e.g., LIFECYCLE_MODE, LIFECYCLE_PROACTIVE, LIFECYCLE_VERIFICATION_STRICTNESS
   - Check: echo $LIFECYCLE_{KEY} in bash
2. Config file: .lifecycle/config.yaml
   - Read the YAML file, find the key
3. Built-in defaults (hardcoded in reference instructions):
   - mode: feature
   - skip_stages: [] (empty)
   - proactive: true
   - auto_skip: true
   - verification_strictness: standard
```

**Example resolution instruction for Claude:**
```markdown
To resolve setting `{key}`:
1. Run: `echo "${LIFECYCLE_{KEY_UPPER}:-}"` -- if non-empty, use this value
2. If empty: read `.lifecycle/config.yaml`, find `{key}:` line, extract value
3. If not found: use default from table below
```

### Pattern 2: Config Get/Set/List via Natural Language

**What:** User says "lifecycle-config get mode" or "set proactive false" and Claude executes.
**When to use:** User explicitly requests config operations.

```markdown
## Phase Transition Detection (addition to SKILL.md)

| Signal | Action |
|--------|--------|
| "lifecycle-config get {key}" | Read setting value via resolution algorithm |
| "lifecycle-config set {key} {value}" | Write to .lifecycle/config.yaml, log to changelog |
| "lifecycle-config list" | Display all settings with current resolved values and source |
| "config", "settings" | Show current config summary |
```

### Pattern 3: Config Change Logging (CONF-03)

**What:** Every `set` operation appends to settings-changelog.md with reason.
**When to use:** Automatically on every config write.

```markdown
When setting a config value:
1. Write value to .lifecycle/config.yaml
2. Ask user for reason (one-liner, e.g., "switching to milestone mode for release")
3. Append to .lifecycle/settings-changelog.md:
   | {date} | .lifecycle/config.yaml | {key}: {old} -> {new} | {reason} |
```

### Pattern 4: Config File Template

**What:** Self-documenting YAML with all settings, comments, and defaults.

```yaml
# Dev Lifecycle Configuration
# Edit directly or use: lifecycle-config set <key> <value>
# Env vars override: LIFECYCLE_<KEY> (e.g., LIFECYCLE_MODE=hotfix)

# Execution mode: hotfix | feature | release | milestone
mode: feature

# Stages to skip (comma-separated stage numbers, overrides mode defaults)
# Example: skip_stages: [5, 6, 9]
skip_stages: []

# Suggest companion skills at stage transitions
proactive: true

# Auto-skip skippable stages without asking (vs. offering skip choice)
auto_skip: true

# Verification strictness: strict | standard | relaxed
# strict: all spec steps must pass, no overrides
# standard: all must pass, override with justification
# relaxed: warnings only, no blocking
verification_strictness: standard
```

### Anti-Patterns to Avoid

- **Storing config in state.json:** state.json is for runtime state (current stage, progress). Config is for user preferences. Keep them separate. Currently `mode` is in state.json -- the config system should be the source of truth, with state.json reading from config on session start.
- **Binary/script approach:** dev-lifecycle has no bin/ directory and must not create one. All operations are Claude instruction-following.
- **Over-validating on read:** Don't block if config.yaml has unknown keys. Be permissive on read, strict on write.
- **Requiring config.yaml to exist:** The defaults layer means the system works with zero configuration. Config file is created only on first `set` or explicit init.

## Don't Hand-Roll

| Problem | Don't Build | Use Instead | Why |
|---------|-------------|-------------|-----|
| YAML parsing | Custom parser logic | Claude's native ability to read YAML | Claude can read YAML natively; no parsing code needed in a skill |
| Change tracking | New changelog system | Existing `.lifecycle/settings-changelog.md` | Already built in v1.0, same format, same rules |
| Settings persistence | New persistence mechanism | `.lifecycle/config.yaml` flat file | Flat YAML file is sufficient; no DB, no complex formats |
| Env var checking | Complex detection logic | Simple `echo $VAR` bash command | One bash command per check; Claude handles the conditional |

**Key insight:** This is a Claude Code skill, not a program. "Implementation" means writing clear instructions that Claude follows, not writing executable code. The config system's complexity lives in the reference document's clarity, not in code.

## Common Pitfalls

### Pitfall 1: Mode Duplication Between state.json and config.yaml
**What goes wrong:** `mode` currently lives in `state.json` (`current.mode`). Adding it to config.yaml creates two sources of truth.
**Why it happens:** state.json was designed before config existed.
**How to avoid:** Config.yaml is the source of truth for `mode`. On session start, read mode from config resolution (env > config > default). Write it into state.json for backward compatibility. Document this clearly: "config.yaml defines mode; state.json reflects it."
**Warning signs:** state.json mode and config.yaml mode diverge.

### Pitfall 2: SKILL.md Line Budget Exceeded
**What goes wrong:** Adding config instructions directly to SKILL.md pushes it over 500 lines.
**Why it happens:** Config has many settings and resolution logic.
**How to avoid:** SKILL.md gets only ~15-20 lines: a "Configuration" section with the settings table and `Read: $CLAUDE_SKILL_DIR/references/lifecycle-config.md` directive. All resolution logic, get/set procedures, and edge cases go in the reference file.
**Warning signs:** SKILL.md exceeding ~360 lines after config additions (currently 339).

### Pitfall 3: Forgetting to Log Changes
**What goes wrong:** Config is changed but settings-changelog not updated, breaking CONF-03.
**Why it happens:** The set procedure doesn't enforce logging.
**How to avoid:** The reference instructions must make logging a mandatory step in the set procedure -- not optional, not "if you remember." Step sequence: validate -> write -> log -> confirm.
**Warning signs:** settings-changelog.md missing entries for known config changes.

### Pitfall 4: Environment Variable Naming Conflicts
**What goes wrong:** `LIFECYCLE_MODE` could conflict with other tools.
**Why it happens:** Generic prefix.
**How to avoid:** Use `LIFECYCLE_` prefix consistently. Document all env var names explicitly. This is unlikely to conflict in practice since it's specific enough.
**Warning signs:** Unexpected config values from unrelated env vars.

### Pitfall 5: Config File Not Created on First Use
**What goes wrong:** User runs `lifecycle-config get mode` before any `set`, and there's no config.yaml.
**Why it happens:** Config file doesn't exist until first write.
**How to avoid:** The resolution algorithm handles this gracefully -- layer 2 (config file) is skipped if file doesn't exist, falls through to layer 3 (defaults). The `list` command should show resolved values with source indicators (env/file/default) even when no config file exists.

## Code Examples

### Example 1: Config Template (templates/config.yaml)

```yaml
# Dev Lifecycle Configuration
# Manage with: lifecycle-config get|set|list
# Env overrides: LIFECYCLE_<KEY> (e.g., LIFECYCLE_MODE=hotfix)

# Execution mode: hotfix | feature | release | milestone
mode: feature

# Stages to always skip (list of stage numbers)
# Overrides mode defaults when non-empty
skip_stages: []

# Suggest companion skills at stage transitions
proactive: true

# Auto-skip skippable stages without prompting
auto_skip: true

# Verification strictness: strict | standard | relaxed
verification_strictness: standard
```

### Example 2: lifecycle-config list Output Format

```
Dev Lifecycle Configuration:

  mode:                      feature     (default)
  skip_stages:               []          (default)
  proactive:                 true        (config.yaml)
  auto_skip:                 true        (default)
  verification_strictness:   standard    (default)

Config file: .lifecycle/config.yaml (exists)
Env overrides: none detected
```

### Example 3: lifecycle-config set Flow

```
User: lifecycle-config set mode milestone
Claude:
  1. Validates: "milestone" is valid for mode (hotfix|feature|release|milestone)
  2. Reads current value: mode = feature (default)
  3. Writes to .lifecycle/config.yaml: mode: milestone
     - If config.yaml doesn't exist, copies from template first
  4. Asks: "Reason for change? (one line)"
  User: "Starting v2.0 release cycle"
  5. Appends to .lifecycle/settings-changelog.md:
     | 2026-03-24 14:30 | .lifecycle/config.yaml | mode: feature -> milestone | Starting v2.0 release cycle |
  6. Updates state.json: current.mode = "milestone"
  7. Confirms: "mode set to milestone (was: feature)"
```

### Example 4: Resolution with Env Override

```
User: LIFECYCLE_MODE=hotfix lifecycle-config get mode
Claude:
  1. Check env: echo $LIFECYCLE_MODE -> "hotfix"
  2. Returns: "hotfix (from environment variable LIFECYCLE_MODE)"
  Note: config.yaml still says "feature" but env takes precedence
```

## State of the Art

| Old Approach (v1.0) | New Approach (v2.0) | Impact |
|---------------------|---------------------|--------|
| `mode` hardcoded in state.json at init | `mode` resolved from config layers | Users can change mode without editing state.json |
| No config file | `.lifecycle/config.yaml` with layered resolution | Unified settings management |
| Settings changelog only tracks project config files (.env, tsconfig, etc.) | Settings changelog also tracks lifecycle config changes | Complete audit trail for all setting changes |
| No env var overrides | `LIFECYCLE_*` env vars override config | CI/CD and temporary overrides without file changes |

## Settings Definition Table

Each setting needs clear definition for the reference document:

| Setting | Type | Default | Valid Values | Used By | Description |
|---------|------|---------|-------------|---------|-------------|
| `mode` | string | `feature` | `hotfix`, `feature`, `release`, `milestone` | Stage transitions, skip logic | Execution mode determining which stages are required |
| `skip_stages` | list[int] | `[]` | Stage numbers 1-9 | Stage transitions | Explicit stage skip overrides (supplements mode defaults) |
| `proactive` | boolean | `true` | `true`, `false` | Ecosystem suggestions (Phase 15) | Whether to suggest companion skills at transitions |
| `auto_skip` | boolean | `true` | `true`, `false` | Stage transitions | Whether to auto-skip without asking or prompt user |
| `verification_strictness` | string | `standard` | `strict`, `standard`, `relaxed` | TEST stage, COMMIT gate | How strictly to enforce spec verification |

## Integration Points

### Existing Files That Need Updates

| File | Change | Reason |
|------|--------|--------|
| `SKILL.md` | Add ~15-20 line "Configuration" section | Entry point for config, phase transition detection |
| `references/stage-transitions.md` | Read mode from config resolution instead of state.json directly | CONF-02 layered precedence |
| `templates/state.json` | No change needed | state.json continues to reflect mode, but source of truth moves to config |
| `references/do-stage.md` | Settings changelog now includes lifecycle config changes | CONF-03 integration |
| `references/commit-stage.md` | Same -- lifecycle config detection in changelog check | CONF-03 integration |

### New Files

| File | Purpose | Approximate Size |
|------|---------|-----------------|
| `references/lifecycle-config.md` | Full config reference (resolution, get/set/list, validation, edge cases) | ~100-120 lines |
| `templates/config.yaml` | Default config template with comments | ~20 lines |

### SKILL.md Budget Impact

- Current: 339 lines
- Addition: ~15-20 lines (Configuration section + phase transition detection entry)
- Projected: ~355-360 lines
- Budget remaining: ~140-145 lines (well within 500 limit)

## Open Questions

1. **skip_stages interaction with mode**
   - What we know: Mode defines default skippable stages. `skip_stages` config adds explicit overrides.
   - What's unclear: Does `skip_stages` replace mode defaults or supplement them? If mode=feature skips [5,6,9] and skip_stages=[1], is the result [1,5,6,9] (union) or [1] (replace)?
   - Recommendation: Union (additive). `skip_stages` adds to mode defaults, never removes required stages. Document clearly.

2. **auto_skip vs mode skip behavior**
   - What we know: Currently stages are offered for skip ("Feature mode allows skipping DEPLOY. Skip?"). auto_skip=true would skip without asking.
   - What's unclear: Should auto_skip apply only to mode-based skips, or also to skip_stages?
   - Recommendation: auto_skip applies to all skippable stages (both mode-based and explicit). When false, user is prompted for each.

3. **verification_strictness levels**
   - What we know: Three levels mentioned (strict/standard/relaxed).
   - What's unclear: Exact behavior differences. Phase 12 (Iteration) will add more verification semantics.
   - Recommendation: Define basic behavior now (strict=block, standard=block+override, relaxed=warn). Phase 12 can refine.

## Validation Architecture

### Test Framework

| Property | Value |
|----------|-------|
| Framework | Manual validation (Claude Code skill -- no executable test framework) |
| Config file | N/A -- skill operates through instruction-following |
| Quick run command | Manual: create test project, run lifecycle-config commands |
| Full suite command | Manual: test all 5 settings x 3 layers + changelog logging |

### Phase Requirements -> Test Map

| Req ID | Behavior | Test Type | Automated Command | File Exists? |
|--------|----------|-----------|-------------------|-------------|
| CONF-01 | get/set/list for all 5 settings | manual | Create .lifecycle/, run lifecycle-config set/get for each setting | N/A |
| CONF-02 | 3-layer resolution (env > file > default) | manual | Set env var, set config file, verify precedence | N/A |
| CONF-03 | Changes logged to settings-changelog | manual | Run lifecycle-config set, verify changelog entry | N/A |

### Sampling Rate
- **Per task:** Read the created/modified files, verify instruction clarity
- **Phase gate:** Manual walkthrough of all 3 requirements in a test project

### Wave 0 Gaps
None -- this is a documentation/instruction skill, not executable code. Validation is through manual testing of the skill instructions.

## Sources

### Primary (HIGH confidence)
- gstack `bin/gstack-config` -- actual bash implementation of get/set/list pattern (read directly from `~/.claude/skills/gstack/bin/gstack-config`)
- gstack `SKILL.md` -- how config is used in session preamble and proactive behavior (read directly)
- dev-lifecycle `SKILL.md` -- current 339-line skill, architecture constraints (read directly)
- dev-lifecycle `templates/state.json` -- current state schema with mode field (read directly)
- dev-lifecycle `templates/settings-changelog.md` -- existing change tracking format (read directly)
- dev-lifecycle `references/stage-transitions.md` -- current mode resolution from state.json (read directly)

### Secondary (MEDIUM confidence)
- REQUIREMENTS.md CONF-01/02/03 definitions -- requirements are locked but implementation details are interpretive

## Metadata

**Confidence breakdown:**
- Standard stack: HIGH -- no external dependencies, all file-based within existing patterns
- Architecture: HIGH -- follows established reference/template pattern from v1.0
- Pitfalls: HIGH -- identified from direct analysis of current codebase (mode duplication, line budget)
- Settings definition: MEDIUM -- exact behavior of verification_strictness and skip_stages interaction needs user confirmation

**Research date:** 2026-03-24
**Valid until:** 2026-04-24 (stable domain, no external dependency drift)
