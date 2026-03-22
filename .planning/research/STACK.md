# Stack Research

**Domain:** Claude Code skill-based development lifecycle orchestrator
**Researched:** 2026-03-22
**Confidence:** HIGH

## Recommended Stack

This project builds Claude Code skills (SKILL.md files), not a traditional software application. The "stack" is the set of patterns, file formats, and structural conventions that make Claude Code skills effective orchestrators. No npm packages or frameworks are needed -- everything runs as markdown instructions interpreted by Claude Code, with optional shell scripts and a Node.js CLI utility.

### Core Technologies

| Technology | Version | Purpose | Why Recommended |
|------------|---------|---------|-----------------|
| Claude Code Skills (SKILL.md) | Agent Skills 1.0 spec | Primary orchestration mechanism | Official, cross-platform standard (works in Claude Code, Codex, Copilot, Cursor). YAML frontmatter + markdown body = zero infrastructure. Auto-invocation via description matching eliminates need for explicit triggers. Confidence: HIGH (verified via official docs) |
| Claude Code Hooks | Settings-based | Deterministic lifecycle automation | Hooks fire every time (not probabilistic like skills). Essential for enforcing stage gates, auto-formatting, context re-injection after compaction. 20+ lifecycle events available. Confidence: HIGH (verified via official docs) |
| Claude Code Subagents | .claude/agents/ | Parallel task execution and delegation | Wave-based execution pattern proven by GSD. Each subagent gets fresh context (200k). Supports model routing (`haiku` for exploration, `sonnet` for execution, `opus` for planning). Confidence: HIGH (verified via official docs) |
| Node.js CLI (gsd-tools.cjs) | Node 18+ | State management, config parsing, git operations | GSD pattern: centralizes 50+ repetitive bash patterns into one CLI. Single-file CJS for zero-dependency distribution. Handles: state load/save, phase operations, frontmatter CRUD, roadmap parsing, progress tracking. Confidence: HIGH (verified via existing GSD codebase) |
| Markdown with YAML Frontmatter | CommonMark + YAML | Structured artifact format | Every planning artifact (STATE.md, ROADMAP.md, PLAN.md, SUMMARY.md) uses this pattern. Frontmatter = machine-readable metadata, body = human-readable content. Claude Code natively parses both. Confidence: HIGH |
| JSON Config | .planning/config.json | Project-specific settings | GSD pattern: mode, granularity, parallelization, model_profile, workflow toggles. Read by CLI, consumed by all workflows. Simple, no schema validation needed -- Claude interprets flexibly. Confidence: HIGH |

### Supporting Libraries

| Library | Version | Purpose | When to Use |
|---------|---------|---------|-------------|
| jq | System | JSON parsing in hook scripts | When hooks need to extract fields from stdin JSON (tool_input.file_path, command, etc.). Required for PreToolUse/PostToolUse hooks. |
| Git | System | Version control + state persistence | Commits = checkpoints. GSD pattern: atomic commits per task, planning docs tracked in git, branch strategies (none/phase/milestone). |
| Bash/Zsh | System | Hook command execution, dynamic context injection | Shell scripts for file validation, formatting, notifications. `!`backtick`` syntax in SKILL.md for pre-processing. |

### Development Tools

| Tool | Purpose | Notes |
|------|---------|-------|
| `/hooks` browser | View/debug configured hooks | Read-only in Claude Code. Edit settings.json directly to modify. |
| `/agents` command | Manage subagent configurations | Interactive creation, but manual file editing preferred for version control. |
| `$CLAUDE_SKILL_DIR` | Reference skill-bundled files | Use in SKILL.md to reference scripts/templates regardless of cwd. |
| `/context` command | Check skill loading budget | Skills share 2% of context window (16K chars fallback). Monitor when skills multiply. |

## Architecture Patterns (Stack-Level)

### Pattern 1: Progressive Disclosure Architecture

**The most important pattern for skill development.**

Skills should NOT load all instructions at once. Instead:

1. **Frontmatter** (always loaded): name, description -- minimal, for routing decisions only
2. **SKILL.md body** (loaded on invocation): core instructions, navigation to supporting files
3. **Supporting files** (loaded on demand): reference.md, templates/, scripts/, examples/

```
my-skill/
  SKILL.md           # < 500 lines. Overview + navigation
  reference.md       # Detailed API docs, loaded when needed
  templates/         # Output templates
  scripts/           # Executable helpers
  examples/          # Expected output samples
```

**Why:** Claude Code's skill description budget is 2% of context window. Bloated descriptions crowd out other skills. Full content loads only on invocation.

### Pattern 2: Orchestrator-Executor Separation

The orchestrator skill stays lean (~10-15% context). It coordinates, never executes directly.

```
Orchestrator (SKILL.md)
  - Reads STATE.md for current position
  - Determines which stage to run
  - Spawns subagent with stage-specific instructions
  - Collects results, updates state
  - Routes to next stage

Executor (subagent)
  - Gets fresh 200k context
  - Reads plan files
  - Executes tasks
  - Creates artifacts
  - Returns structured result
```

GSD proves this: execute-phase.md orchestrates, execute-plan.md executes. The orchestrator never touches code directly.

### Pattern 3: Markdown-as-Memory

Every artifact produced during development serves as external memory for future sessions:

| Artifact | Memory Function |
|----------|----------------|
| STATE.md | Where am I? What's next? |
| ROADMAP.md | What's the full plan? What's done? |
| PROJECT.md | Why are we building this? What are the constraints? |
| SUMMARY.md (per plan) | What was built? What decisions were made? |
| ADR docs | Why was X chosen over Y? |
| Retrospective docs | What went wrong? What to remember next time? |

**Key insight from PROJECT.md:** "Stage artifacts themselves serve as structured memory -- no separate 'remember this' mechanism needed."

### Pattern 4: State Machine via Filesystem

Stage transitions are tracked through file existence and frontmatter status fields, not in-memory state:

```
.planning/
  STATE.md              # Current position (phase, plan, status)
  ROADMAP.md            # Phase definitions + progress table
  config.json           # Project settings
  phases/
    01-plan/
      01-01-PLAN.md     # Plan exists = planned
      01-01-SUMMARY.md  # Summary exists = executed
      01-VERIFICATION.md # Verification exists = verified
```

**Why filesystem, not database:** Claude Code sessions are ephemeral. Context compaction loses in-memory state. Files persist across sessions, compactions, and crashes.

## Invocation Control Matrix

Critical for the dev-lifecycle orchestrator -- some skills should auto-trigger, others should be user-only:

| Skill | `disable-model-invocation` | `user-invocable` | Rationale |
|-------|---------------------------|-------------------|-----------|
| dev-lifecycle (main) | false | true | Auto-detects phase transitions AND user can invoke directly |
| Stage executors | true | false | Only the orchestrator should trigger these |
| ADR | false | true | Auto-detects decision moments AND user can invoke |
| gsd-retrospective | true | true | User-triggered only (no false positives) |
| work-log | true | true | User-triggered only |

## Hook Strategy for Dev-Lifecycle

| Hook Event | Matcher | Purpose | Confidence |
|------------|---------|---------|------------|
| `SessionStart` | `startup` | Re-inject STATE.md context, surface pre-flight checks | HIGH |
| `SessionStart` | `compact` | Re-inject critical state after compaction (prevents context loss) | HIGH |
| `PreToolUse` | `Edit\|Write` | Validate against stage gates (no code edits during PLAN stage) | MEDIUM |
| `PostToolUse` | `Edit\|Write` | Track modified files for COMMIT stage artifact | HIGH |
| `Stop` | (none) | Check if current stage deliverables are complete before allowing stop | MEDIUM |
| `SubagentStop` | (any) | Collect executor results, update STATE.md | HIGH |

## Alternatives Considered

| Recommended | Alternative | When to Use Alternative |
|-------------|-------------|-------------------------|
| SKILL.md (Agent Skills standard) | .claude/commands/ (legacy) | Never for new work. Commands still work but lack: supporting files, invocation control, subagent execution. Skills supersede commands entirely. |
| Node.js CLI (gsd-tools.cjs) | Pure bash scripts | When operations are trivial (< 10 lines). For anything involving JSON parsing, file manipulation, or state management, the CLI is more reliable. Bash is fragile with JSON. |
| Subagents for execution | Inline execution in skill | Never for complex tasks. Inline execution bleeds into orchestrator context. Always fork to subagent for tasks that produce output or modify files. |
| `context: fork` on skills | Manual Task() spawning | When the skill IS the task definition. Use `context: fork` + `agent: [type]` for self-contained skills. Use Task() (now Agent tool) when orchestrator needs to compose the task dynamically. |
| Filesystem state (STATE.md) | In-memory state / environment vars | Never for cross-session state. Env vars die with the session. In-memory state dies with compaction. Files survive everything. |
| Progressive disclosure | Monolithic SKILL.md | Never. A 2000-line SKILL.md wastes context budget. Split into SKILL.md (< 500 lines) + supporting files. |

## What NOT to Use

| Avoid | Why | Use Instead |
|-------|-----|-------------|
| MCP servers for state management | Overkill for file-based state. Adds infrastructure dependency. Skills should be zero-infrastructure. | Filesystem state (STATE.md, config.json) read/written by CLI tool |
| Database (SQLite, etc.) for planning artifacts | Claude Code reads/writes markdown natively. SQL adds complexity with zero benefit for this use case. | Markdown files with YAML frontmatter |
| Complex dependency injection | Skills are prompt instructions, not OOP code. DI patterns don't apply. | Progressive disclosure: reference files when needed |
| Separate processes/servers | Constraint: "no separate server or infrastructure." Skills run inside Claude Code. | Shell scripts for pre-processing, CLI tool for complex operations |
| Hard-coded project paths in SKILL.md | The current SKILL.md has muse-specific paths (server URLs, APK build commands). This prevents reuse. | Template variables, project-specific config in .planning/config.json |
| Overly broad skill descriptions | If description matches too many user inputs, skill triggers constantly, wasting context. | Specific trigger phrases + `disable-model-invocation: true` for side-effect skills |
| `user-invocable: false` for main orchestrator | Would prevent users from typing /dev-lifecycle directly. | Keep default (both user and model can invoke) |
| Amending commits in hooks | PostToolUse hooks run after tool completion. Git amend in hooks can lose work. | Always create NEW commits. GSD enforces this pattern. |

## Stack Patterns by Variant

**If building a simple skill (reference/conventions):**
- Single SKILL.md file, no supporting files
- No subagents needed
- Use `user-invocable: false` if it's background knowledge
- Example: api-conventions, coding-style

**If building an orchestrator skill (this project):**
- SKILL.md + reference files + templates + CLI tool
- Multiple subagents for stage execution
- Hooks for deterministic enforcement
- Filesystem state management
- Progressive disclosure is mandatory
- Example: dev-lifecycle, GSD

**If building a generator skill (creates artifacts):**
- SKILL.md + templates/ + scripts/
- Use `context: fork` to isolate generation
- Bundle scripts in skill directory, reference via `$CLAUDE_SKILL_DIR`
- Example: canvas-design, codebase-visualizer

## Version Compatibility

| Component | Compatible With | Notes |
|-----------|-----------------|-------|
| Agent Skills spec 1.0 | Claude Code, OpenAI Codex, GitHub Copilot, Cursor, VS Code, Gemini CLI | Cross-platform standard. Core features (SKILL.md, frontmatter, supporting files) work everywhere. Claude Code extensions (hooks, subagents, `context: fork`) are Claude-specific. |
| gsd-tools.cjs (Node.js) | Node 18+ | Single-file CJS format. No package.json, no dependencies. Runs on any system with Node installed. |
| Hooks system | Claude Code 1.0+ | Settings-based configuration. 20+ lifecycle events. Prompt-based and agent-based hooks available for judgment calls. |
| Subagents | Claude Code 1.0+ | Up to 10 simultaneous subagents (2026). `.claude/agents/` for project-level, `~/.claude/agents/` for user-level. |

## Key Constraints from PROJECT.md

These constraints directly shape stack choices:

1. **Skill format only** -- No servers, no infrastructure. Everything runs as SKILL.md + supporting files inside Claude Code.
2. **Orchestration over modification** -- Don't modify GSD/PDCA. Build on top of them. The dev-lifecycle skill calls GSD/PDCA skills, not replaces them.
3. **Universal applicability** -- Must work for web, mobile, API, desktop. No framework-specific code in the orchestrator. Project-specific details go in config.json.
4. **Hardcoding removal** -- Current SKILL.md has muse-specific paths, commands, and flows. All must become configurable or templated.

## Sources

- [Claude Code Skills Documentation](https://code.claude.com/docs/en/skills) -- SKILL.md format, frontmatter reference, progressive disclosure, invocation control. Verified 2026-03-22. Confidence: HIGH
- [Claude Code Hooks Guide](https://code.claude.com/docs/en/hooks-guide) -- Hook events, matchers, input/output, prompt-based and agent-based hooks. Verified 2026-03-22. Confidence: HIGH
- [Claude Code Subagents Documentation](https://code.claude.com/docs/en/sub-agents) -- Subagent configuration, frontmatter fields, tool restrictions, memory, isolation. Verified 2026-03-22. Confidence: HIGH
- [Agent Skills Open Standard](https://agentskills.io/home) -- Cross-platform specification. Adopted by Anthropic, Microsoft, OpenAI, Atlassian, Figma, Cursor, GitHub. Confidence: HIGH
- [Anthropic Skills Repository](https://github.com/anthropics/skills) -- Official skill specification and SDK. Confidence: HIGH
- GSD codebase (`~/.claude/get-shit-done/`) -- Proven orchestration patterns: CLI tool, workflows, state management, subagent spawning. Directly examined. Confidence: HIGH
- Existing skills (`~/.claude/skills/`) -- dev-lifecycle, adr, gsd-retrospective, work-log, canvas-design. Directly examined. Confidence: HIGH

---
*Stack research for: Claude Code skill-based development lifecycle orchestrator*
*Researched: 2026-03-22*
