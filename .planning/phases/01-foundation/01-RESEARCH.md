# Phase 1: Foundation - Research

**Researched:** 2026-03-22
**Domain:** Claude Code skill architecture, JSON state management, artifact manifest design, skill orchestration
**Confidence:** HIGH

## Summary

Phase 1 builds the generic skeleton of the dev-lifecycle orchestrator skill. The work is entirely within the Claude Code skill ecosystem -- no npm packages, no servers, no build tools. The deliverables are: (1) a rewritten SKILL.md stripped of muse hardcoding, (2) a `.lifecycle/` directory structure with `state.json` and `manifest.json`, (3) templates for stage artifacts, and (4) a role matrix defining which existing skill handles what at each stage.

The current SKILL.md (281 lines) contains hardcoded rsync paths, flutter build commands, muse-specific PIN authentication flows, and server-specific deployment commands. All of these must be replaced with generic, project-type-agnostic patterns. The skill structure follows the Agent Skills 1.0 standard with progressive disclosure: SKILL.md under 500 lines, detailed logic in `references/` files split by function.

**Primary recommendation:** Build from the inside out -- state.json schema first (it is the backbone everything reads/writes), then manifest.json (it connects stages), then SKILL.md rewrite (it orchestrates), then role matrix (it documents boundaries).

<user_constraints>
## User Constraints (from CONTEXT.md)

### Locked Decisions
- Skill lives in `~/.claude/skills/dev-lifecycle/` (global) with `.claude/skills/` (local) override support
- `references/` split by function (not by stage) -- e.g., preflight.md, e2e-spec.md, prototype.md
- `templates/` folder contains all stage artifact templates (spec, prototype, verification report, deploy checklist, etc.)
- `.lifecycle/` namespace separate from `.planning/` -- dev-lifecycle state does NOT live in GSD's directory
- `state.json` + `manifest.json` as separate files (not combined)
- Auto-detect + invoke existing skills (GSD, PDCA, ADR, retrospective, work-log)
- Users can use GSD/PDCA/dev-lifecycle in parallel -- no single entry point forced
- dev-lifecycle orchestrates when overlap occurs (e.g., GSD plan vs PDCA plan)
- All project types supported equally (web, mobile, API, desktop, CLI)
- Auto-detect project type (package.json, pubspec.yaml, etc.) then user confirms

### Claude's Discretion
- GSD STATE.md relationship -- read-only sync vs independent operation (research recommends read-only)
- `references/` file list and responsibility scope per file
- `state.json` schema field design (detailed schema proposed below)

### Deferred Ideas (OUT OF SCOPE)
- None -- discussion stayed within phase scope
</user_constraints>

<phase_requirements>
## Phase Requirements

| ID | Description | Research Support |
|----|-------------|-----------------|
| FOUND-01 | muse hardcoding removal -- generic skill for all project types | Current SKILL.md analysis identifies 12+ muse-specific items to remove. Project type detection pattern documented. SKILL.md rewrite guidance provided. |
| FOUND-02 | state.json for current stage, progress, active feature tracking (session-persistent) | Full state.json schema designed. Read/write patterns documented. Session resumption flow defined. |
| FOUND-03 | Stage Artifact Manifest -- register each stage's output, link as required input to next stage | manifest.json schema designed with per-stage artifact registration and dependency declaration. Gate logic documented. |
| FOUND-05 | GSD/PDCA/custom skill role matrix -- which skill handles what at each stage | Complete role matrix with all 9 stages, 6 skills, and clear primary/supporting/optional designations. |
</phase_requirements>

## Standard Stack

### Core

| Technology | Version | Purpose | Why Standard |
|------------|---------|---------|--------------|
| Claude Code Skills (SKILL.md) | Agent Skills 1.0 | Primary orchestration mechanism | Cross-platform standard (Claude Code, Codex, Copilot, Cursor). YAML frontmatter + markdown body. Zero infrastructure. |
| JSON files (state.json, manifest.json) | JSON | Persistent state across sessions | Claude Code natively reads/writes JSON. Survives compaction, session end, crashes. Simpler than YAML for programmatic access. |
| Markdown with YAML frontmatter | CommonMark + YAML | Artifact format (templates, references, role matrix) | Every planning artifact in the ecosystem uses this pattern. Claude Code parses natively. |

### Supporting

| Technology | Version | Purpose | When to Use |
|------------|---------|---------|-------------|
| `$CLAUDE_SKILL_DIR` | Claude Code built-in | Reference skill-bundled files regardless of cwd | Always -- use in SKILL.md to point to references/ and templates/ |
| Git | System | Version control for .lifecycle/ artifacts | Commit state changes at stage transitions |
| Bash/Zsh | System | Project type auto-detection scripts | During initial setup and project type detection |

### Alternatives Considered

| Instead of | Could Use | Tradeoff |
|------------|-----------|----------|
| Separate state.json + manifest.json | Single combined state file | Separate files: cleaner concerns, but two file reads. User decision locks this as separate. |
| `.lifecycle/` namespace | `.planning/` reuse | `.lifecycle/` prevents collision with GSD. User decision locks this. |
| JSON for state | YAML for state | JSON is simpler for programmatic read/write. YAML is more readable but parsing is heavier. JSON wins for machine state. |

## Architecture Patterns

### Recommended Project Structure

```
~/.claude/skills/dev-lifecycle/
  SKILL.md                    # < 500 lines. Core orchestration + navigation
  references/
    preflight.md              # Stage 1 pre-flight check logic
    stage-transitions.md      # Stage gate rules + transition conditions
    project-detection.md      # Auto-detect project type patterns
    role-matrix.md            # Which skill handles what (FOUND-05 artifact)
    skill-invocation.md       # How to call GSD/PDCA/ADR/etc.
  templates/
    state.json                # Initial state template
    manifest.json             # Initial manifest template
    stage-artifacts/          # Per-stage output templates
      plan-output.md
      do-checklist.md
      test-report.md
      commit-checklist.md
      deploy-checklist.md
      deploy-test-report.md
      document-checklist.md
      retrospective-template.md
      promote-checklist.md
```

```
{project}/
  .lifecycle/                 # Dev-lifecycle state (separate from .planning/)
    state.json                # Current stage, progress, active feature
    manifest.json             # Artifact registry per stage
    history/                  # Stage transition log
      YYYY-MM-DD-HH-MM.json
    features/                 # Per-feature artifacts (later phases)
      {feature-name}/
        spec.md
        prototype.html
        verification.json
```

### Pattern 1: State Machine via JSON

**What:** `state.json` is the single source of truth for where the project is in the lifecycle pipeline. Every skill interaction reads it first, acts, then updates it.

**When to use:** Every session start, every stage transition, every progress update.

**Schema:**

```json
{
  "version": "1.0",
  "project": {
    "name": "",
    "type": "auto",
    "detected_type": null,
    "confirmed": false
  },
  "current": {
    "stage": 1,
    "stage_name": "PLAN",
    "status": "not_started",
    "feature": null,
    "mode": "feature"
  },
  "progress": {
    "stages_completed": [],
    "current_stage_started_at": null,
    "last_transition_at": null
  },
  "session": {
    "last_active": null,
    "resume_hint": ""
  }
}
```

**Status values:** `not_started` | `in_progress` | `completed` | `blocked` | `skipped`

**Mode values:** `feature` | `hotfix` | `release` | `milestone` (FOUND-04 is Phase 10, but the field should exist now for forward compatibility)

### Pattern 2: Artifact Manifest as Stage Connector

**What:** `manifest.json` registers every artifact produced at each stage and declares what the next stage requires. This is the mechanism that connects stage outputs to stage inputs (FOUND-03).

**When to use:** After every artifact creation. Before every stage transition (gate check).

**Schema:**

```json
{
  "version": "1.0",
  "feature": "feature-name",
  "artifacts": {
    "1_plan": {
      "required_inputs": [],
      "outputs": [
        {
          "type": "plan",
          "path": ".planning/phases/01-feature/01-PLAN.md",
          "created_at": "2026-03-22T10:00:00Z",
          "status": "complete"
        }
      ],
      "completed_at": "2026-03-22T10:30:00Z"
    },
    "2_do": {
      "required_inputs": ["1_plan.outputs[type=plan]"],
      "outputs": [],
      "completed_at": null
    }
  },
  "gate_rules": {
    "1_to_2": ["1_plan.outputs.length > 0"],
    "2_to_3": ["2_do.outputs.length > 0"],
    "3_to_4": ["3_test.outputs[type=verification].status == 'passed'"]
  }
}
```

### Pattern 3: Progressive Disclosure for SKILL.md

**What:** SKILL.md stays under 500 lines. It contains: pipeline overview, stage summaries (2-3 lines each), state management commands, and pointers to `references/` files for detailed logic.

**When to use:** Always. The 2% context window budget for skill descriptions makes this mandatory.

**Example SKILL.md structure:**

```markdown
---
name: dev-lifecycle
description: [auto-invocation keywords]
---

# Dev Lifecycle Orchestrator

## Quick Reference
[Pipeline diagram - 10 lines]

## Session Start
1. Read .lifecycle/state.json
2. Display current position
3. Show next action

## Stage Overview
[Stage 1-9, 2-3 lines each = ~50 lines]

## Stage Transitions
Read: $CLAUDE_SKILL_DIR/references/stage-transitions.md

## Project Detection
Read: $CLAUDE_SKILL_DIR/references/project-detection.md

## Skill Invocation
Read: $CLAUDE_SKILL_DIR/references/skill-invocation.md
```

### Pattern 4: Project Type Auto-Detection

**What:** On first invocation in a new project, scan for project markers to determine type. Present detection result to user for confirmation.

**Detection markers:**

| File | Project Type |
|------|-------------|
| `package.json` | Node.js / Web |
| `pubspec.yaml` | Flutter / Dart |
| `Cargo.toml` | Rust |
| `go.mod` | Go |
| `requirements.txt` / `pyproject.toml` | Python |
| `*.xcodeproj` / `*.xcworkspace` | iOS |
| `build.gradle` / `build.gradle.kts` | Android / JVM |
| `Makefile` only | C/C++ |
| `*.sln` / `*.csproj` | .NET |

**Flow:**
```
1. Scan project root for markers
2. Determine type (may be multi: Flutter + Python backend)
3. Present to user: "Detected: Flutter app with Python backend. Correct?"
4. Store in state.json: project.detected_type, project.confirmed
5. Use type for stage-specific templates (deploy, test, etc.)
```

### Anti-Patterns to Avoid

- **Monolithic SKILL.md:** Do NOT put all stage logic inline. Use references/ for anything over 10 lines of stage-specific instruction.
- **Hardcoded paths:** No `/home/opc/`, no `muse-backend/`, no specific server IPs. Everything must be configurable or detected.
- **Storing state in SKILL.md:** State goes in `.lifecycle/state.json`, never in the skill file itself.
- **Modifying GSD/PDCA skills:** dev-lifecycle calls them via their existing interfaces. Never edit their SKILL.md files.
- **Single entry point enforcement:** Users must be free to invoke `/gsd:plan-phase` directly without going through dev-lifecycle first.

## Don't Hand-Roll

| Problem | Don't Build | Use Instead | Why |
|---------|-------------|-------------|-----|
| Phase planning | Custom planning logic | GSD (`/gsd:plan-phase`, `/gsd:discuss-phase`) | GSD has proven planning workflows with research, plans, verification |
| PDCA cycles | Custom plan-do-check-act | PDCA skill (`/pdca plan`, `/pdca do`) | Existing skill with status tracking |
| Decision records | Custom decision format | ADR skill (`/adr`) | Established format with WHY+SEE code comment integration |
| Retrospectives | Custom retro format | gsd-retrospective skill | Git-aware, structured output, CLAUDE.md integration |
| Work logging | Custom log format | work-log skill | Obsidian vault integration already built |
| JSON reading/writing | Complex bash JSON manipulation | Native Claude Code JSON handling or simple node scripts | Bash is fragile with JSON. Claude reads/writes JSON natively. |

**Key insight:** Phase 1 builds orchestration infrastructure, not stage execution logic. The execution logic already exists in GSD, PDCA, ADR, retrospective, and work-log skills. Dev-lifecycle's job is to know WHEN to call WHICH skill, and to track WHAT was produced.

## Common Pitfalls

### Pitfall 1: Trying to Replace GSD/PDCA Instead of Orchestrating

**What goes wrong:** Building custom planning/execution logic in dev-lifecycle instead of delegating to existing skills.
**Why it happens:** It feels simpler to write the logic directly than to figure out invocation interfaces.
**How to avoid:** The role matrix (FOUND-05) must clearly mark each stage's PRIMARY skill. Dev-lifecycle only orchestrates -- it never executes stage work directly.
**Warning signs:** SKILL.md growing past 500 lines. Stage instructions containing implementation details instead of skill invocation pointers.

### Pitfall 2: Over-Engineering state.json Schema

**What goes wrong:** Designing a complex state schema with nested objects, arrays of history, computed fields, that becomes hard to read/write correctly.
**Why it happens:** Anticipating future needs (FOUND-04 modes, parallel features, etc.) and trying to solve them now.
**How to avoid:** Start with the minimal schema that satisfies FOUND-02 (stage, status, feature, timestamps). Add fields only when a later phase needs them. The `version` field enables future schema migration.
**Warning signs:** state.json exceeding 30 lines for a single-feature project. Fields that no current requirement reads.

### Pitfall 3: manifest.json Gate Rules That Are Too Strict

**What goes wrong:** Gate rules that block stage transitions for artifacts that are not always needed (e.g., requiring a prototype.html for a CLI tool).
**Why it happens:** Designing for the happy path (web app with UI) without considering other project types.
**How to avoid:** Gate rules should check for "required" artifacts only. Mark artifacts as `required` vs `optional` per project type. The role matrix should indicate which artifacts are mandatory per stage.
**Warning signs:** Users unable to proceed because an artifact type does not apply to their project.

### Pitfall 4: Losing .lifecycle/ State on Git Operations

**What goes wrong:** `.lifecycle/state.json` gets reset or lost during git checkout, branch switch, or merge.
**Why it happens:** `.lifecycle/` is either gitignored (loses state on clone) or tracked (causes merge conflicts on state changes).
**How to avoid:** Track `.lifecycle/` in git but treat state.json as a "last-writer-wins" file. Add `.lifecycle/state.json` to `.gitattributes` with merge strategy `ours` to prevent merge conflicts. History/ files are append-only and safe to merge.
**Warning signs:** State showing wrong stage after branch operations.

### Pitfall 5: Muse-Specific Patterns Surviving in Generic Forms

**What goes wrong:** Removing the word "muse" but keeping the pattern. E.g., replacing `rsync to muse-backend` with `rsync to {server}` -- still assumes rsync-based deployment.
**Why it happens:** Pattern blindness. The developer sees the hardcoded value and replaces it with a variable, without questioning whether the pattern itself is universal.
**How to avoid:** For each muse-specific item, ask: "Would a React web app need this? A Go CLI tool? A Python API?" If no, the entire pattern must be removed or made optional, not just parameterized.
**Warning signs:** Templates that assume a specific deployment model, test approach, or build system.

## Code Examples

### state.json -- Initial State for New Project

```json
{
  "version": "1.0",
  "project": {
    "name": "my-project",
    "type": "auto",
    "detected_type": "node-web",
    "confirmed": true
  },
  "current": {
    "stage": 1,
    "stage_name": "PLAN",
    "status": "not_started",
    "feature": null,
    "mode": "feature"
  },
  "progress": {
    "stages_completed": [],
    "current_stage_started_at": null,
    "last_transition_at": null
  },
  "session": {
    "last_active": "2026-03-22T10:00:00Z",
    "resume_hint": "Ready to start Stage 1 PLAN. No previous work."
  }
}
```

### manifest.json -- After Stage 1 PLAN Completes

```json
{
  "version": "1.0",
  "feature": "user-auth",
  "artifacts": {
    "1_plan": {
      "required_inputs": [],
      "outputs": [
        {
          "type": "plan",
          "path": ".planning/phases/01-user-auth/01-PLAN.md",
          "created_at": "2026-03-22T10:30:00Z",
          "status": "complete"
        }
      ],
      "completed_at": "2026-03-22T11:00:00Z"
    },
    "2_do": {
      "required_inputs": ["1_plan"],
      "outputs": [],
      "completed_at": null
    }
  }
}
```

### SKILL.md Description Field -- Auto-Invocation Keywords

```yaml
description: >-
  Full-cycle development orchestrator. Coordinates 9 stages from planning
  through deployment and documentation. Use when: "lifecycle", "dev lifecycle",
  "phase workflow", "start phase", "finish phase", "next stage", "stage transition",
  "개발 라이프사이클", "스테이지", "다음 단계", "phase complete", "전체 워크플로",
  "rework prevention", "재작업 방지", "stage gate", "artifact check".
```

### Muse Hardcoding Removal Inventory

Items to remove from current SKILL.md (line references to current 281-line file):

| Line(s) | Hardcoded Item | Replacement |
|---------|---------------|-------------|
| 100-116 | rsync to `opc@server:/home/opc/muse-backend/` | Generic deploy template pointer |
| 106 | `ssh opc@server "sudo systemctl restart muse"` | Generic restart template |
| 108 | `flutter build apk --release --dart-define=API_URL=http://server:8000` | Project-type-specific build template |
| 110 | `ssh opc@server "cd /home/opc/muse-backend && .venv/bin/alembic upgrade head"` | Generic migration template |
| 122 | `curl http://server:8000/docs` | Generic health check template |
| 123 | PIN authentication test | Generic auth test template |
| 124 | Character selection + chat flow | Remove -- project-specific |
| 125 | `ssh opc@server "free -h"` | Generic resource check template |
| 126 | APK install test | Remove -- project-specific |
| 139-141 | `docs/muse-architecture.html`, `docs/muse-architecture.png` | Generic documentation template |
| 155-160 | Mermaid diagrams: PIN login, God Agent, etc. | Remove -- project-specific content |
| 196-213 | Maestro + ADB + FFmpeg pipeline | Keep as optional promote template, remove muse scenarios |

### Role Matrix (FOUND-05 Artifact)

```markdown
# Skill Role Matrix

| Stage | Primary Skill | Supporting Skills | dev-lifecycle Role |
|-------|--------------|-------------------|-------------------|
| 1. PLAN | GSD (`/gsd:plan-phase`) | PDCA (`/pdca plan`), ADR (trade-off detection) | Trigger pre-flight check, invoke GSD, register plan artifacts |
| 2. DO | PDCA (`/pdca do`) | ADR (auto-detect decisions), WHY+SEE comments | Track implementation progress, update state.json |
| 3. TEST | GSD (`/gsd:verify-work`) | PDCA (`/pdca analyze`) | Run verification, compare against manifest, update status |
| 4. COMMIT | Git (direct) | ADR (reference in commit msg) | Verify artifacts exist before allowing commit |
| 5. DEPLOY | Project-specific | None | Provide deploy template, track deploy artifact |
| 6. DEPLOY TEST | Project-specific | None | Provide smoke test template, gate next stage |
| 7. DOCUMENT | canvas-design (optional) | Mermaid (inline) | Trigger documentation generation |
| 8. RETROSPECT | gsd-retrospective | ADR (gap check), work-log | Invoke retro skill, update CLAUDE.md |
| 9. PROMOTE | Manual / optional | None | Provide promote checklist |

**Key principle:** dev-lifecycle NEVER executes stage work directly. It:
1. Detects current stage from state.json
2. Invokes the appropriate primary skill
3. Collects outputs into manifest.json
4. Gates transition to the next stage
5. Updates state.json
```

### Session Resumption Pattern

```
On session start (or after compaction):
1. Read .lifecycle/state.json
2. Display:
   "Dev Lifecycle: Stage {N} {NAME} -- {STATUS}"
   "Feature: {feature_name}"
   "Last active: {timestamp}"
   "Resume hint: {hint}"
3. If status == "in_progress":
   "Continuing Stage {N}. {resume_hint}"
4. If status == "completed":
   "Stage {N} complete. Ready for Stage {N+1} ({next_name})."
   Check manifest gates for Stage {N+1}.
```

## State of the Art

| Old Approach (current SKILL.md) | New Approach (Phase 1 target) | Impact |
|--------------------------------|-------------------------------|--------|
| Hardcoded muse paths and commands | Generic templates + project type detection | Universal project support |
| No persistent state | state.json tracks stage and progress | Session continuity |
| No artifact tracking | manifest.json registers outputs per stage | Stage-to-stage data flow |
| Implicit skill delegation ("use GSD") | Explicit role matrix with primary/supporting designations | No ambiguity about which skill does what |
| Monolithic 281-line SKILL.md | SKILL.md < 500 lines + references/ + templates/ | Progressive disclosure, context budget safe |
| Stage 5 assumes rsync+systemd+flutter | Stage 5 uses project-type-specific templates | All deployment models supported |

## Validation Architecture

### Test Framework

| Property | Value |
|----------|-------|
| Framework | Manual validation (Claude Code skill -- no automated test runner) |
| Config file | None -- skill validation is behavioral |
| Quick run command | `cat .lifecycle/state.json` (verify state schema) |
| Full suite command | Manual: invoke skill in test project, verify all 4 requirements |

### Phase Requirements to Test Map

| Req ID | Behavior | Test Type | Automated Command | File Exists? |
|--------|----------|-----------|-------------------|-------------|
| FOUND-01 | No muse-specific references in any skill file | smoke | `grep -ri "muse\|opc@\|muse-backend\|flutter build apk\|PIN" ~/.claude/skills/dev-lifecycle/` (expect 0 results) | N/A |
| FOUND-02 | state.json created on first invocation, persists stage/feature/progress | manual | `cat {project}/.lifecycle/state.json` (verify schema fields) | Wave 0: create state.json template |
| FOUND-03 | manifest.json registers artifacts, gates prevent skipping stages with missing artifacts | manual | `cat {project}/.lifecycle/manifest.json` (verify artifact entries after stage completion) | Wave 0: create manifest.json template |
| FOUND-05 | Role matrix document exists with all 9 stages mapped to skills | smoke | `cat ~/.claude/skills/dev-lifecycle/references/role-matrix.md` (verify content) | Wave 0: create role-matrix.md |

### Sampling Rate

- **Per task commit:** `grep -ri "muse\|opc@" ~/.claude/skills/dev-lifecycle/` (zero matches = pass)
- **Per wave merge:** Manual invocation in a test project directory
- **Phase gate:** All 4 requirements verified before `/gsd:verify-work`

### Wave 0 Gaps

- [ ] `~/.claude/skills/dev-lifecycle/templates/state.json` -- initial state template
- [ ] `~/.claude/skills/dev-lifecycle/templates/manifest.json` -- initial manifest template
- [ ] `~/.claude/skills/dev-lifecycle/references/role-matrix.md` -- FOUND-05 deliverable
- [ ] `~/.claude/skills/dev-lifecycle/references/stage-transitions.md` -- gate logic
- [ ] `~/.claude/skills/dev-lifecycle/references/project-detection.md` -- auto-detect patterns
- [ ] `~/.claude/skills/dev-lifecycle/references/skill-invocation.md` -- how to call each skill

## Open Questions

1. **GSD STATE.md sync strategy**
   - What we know: dev-lifecycle has its own state.json in `.lifecycle/`. GSD has STATE.md in `.planning/`.
   - What's unclear: Should dev-lifecycle read GSD's STATE.md to know GSD phase/plan progress? Or operate fully independently?
   - Recommendation: Read-only. dev-lifecycle reads `.planning/STATE.md` at session start to know GSD's current position, but never writes to it. GSD owns its own state. This prevents conflicts while enabling awareness.

2. **references/ file granularity**
   - What we know: Split by function, not by stage (user decision).
   - What's unclear: Exact file count and responsibility boundaries.
   - Recommendation: Start with 4 files (role-matrix.md, stage-transitions.md, project-detection.md, skill-invocation.md). Add more only when a file exceeds ~200 lines.

3. **manifest.json complexity level**
   - What we know: Must track artifacts and connect stages.
   - What's unclear: How complex the gate rules should be for Phase 1.
   - Recommendation: Simple existence checks only ("does Stage N have at least one output?"). Complex verification (spec step status, prototype coverage) belongs in later phases when E2E spec is implemented.

## Sources

### Primary (HIGH confidence)

- Direct analysis of `~/.claude/skills/dev-lifecycle/SKILL.md` -- current 281-line muse-hardcoded skill (line-by-line inventory above)
- Direct analysis of `~/.claude/skills/adr/SKILL.md`, `gsd-retrospective/SKILL.md`, `work-log/SKILL.md` -- existing skill interfaces for role matrix
- Direct analysis of `~/.claude/get-shit-done/` -- GSD patterns for state management, CLI tools, templates, references
- `.planning/research/STACK.md` -- verified skill architecture patterns, progressive disclosure, hooks strategy
- `.planning/research/ARCHITECTURE.md` -- `.lifecycle/` structure, spec+prototype patterns, verification flows
- `.planning/REQUIREMENTS.md` -- FOUND-01 through FOUND-05 definitions
- `.planning/phases/01-foundation/01-CONTEXT.md` -- locked decisions, Claude's discretion areas

### Secondary (MEDIUM confidence)

- Agent Skills 1.0 standard patterns (from STACK.md research, verified against official docs)
- GSD template patterns (state.md, config.json) as inspiration for state.json/manifest.json design

### Tertiary (LOW confidence)

- None -- all findings verified against existing codebase and prior research documents.

## Metadata

**Confidence breakdown:**
- Standard stack: HIGH -- all technologies are existing Claude Code features already in use
- Architecture: HIGH -- patterns derived from working GSD implementation and prior architecture research
- Pitfalls: HIGH -- derived from direct analysis of current SKILL.md hardcoding patterns
- State schema: MEDIUM -- schema design is Claude's discretion, will need validation during implementation

**Research date:** 2026-03-22
**Valid until:** 2026-04-22 (stable domain -- Claude Code skill format unlikely to change within 30 days)
