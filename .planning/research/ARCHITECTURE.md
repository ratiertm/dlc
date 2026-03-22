# Architecture Research

**Domain:** Claude Code skill-based development lifecycle orchestrator
**Researched:** 2026-03-22
**Confidence:** HIGH (based on direct analysis of existing skill implementations)

## System Overview

```
┌─────────────────────────────────────────────────────────────────────┐
│                     USER (Claude Code session)                      │
│                                                                     │
│  "start phase" / "deploy" / "retro" / slash commands                │
├─────────────────────────────────────────────────────────────────────┤
│                     ORCHESTRATOR LAYER                               │
│                     (dev-lifecycle SKILL.md)                         │
│                                                                     │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐              │
│  │ Stage Router │  │ State Engine │  │ Artifact Bus │              │
│  │ (detection + │  │ (lifecycle   │  │ (output→input│              │
│  │  transition) │  │  tracker)    │  │  connector)  │              │
│  └──────┬───────┘  └──────┬───────┘  └──────┬───────┘              │
│         │                 │                  │                      │
├─────────┴─────────────────┴──────────────────┴──────────────────────┤
│                     SKILL DELEGATION LAYER                           │
│                     (adapter per wrapped skill)                      │
│                                                                     │
│  ┌────────┐ ┌────────┐ ┌────────┐ ┌────────┐ ┌────────┐           │
│  │  GSD   │ │  PDCA  │ │  ADR   │ │ Retro  │ │Work-Log│           │
│  │Adapter │ │Adapter │ │Adapter │ │Adapter │ │Adapter │           │
│  └────┬───┘ └────┬───┘ └────┬───┘ └────┬───┘ └────┬───┘           │
│       │          │          │          │          │                 │
├───────┴──────────┴──────────┴──────────┴──────────┴─────────────────┤
│                     EXISTING SKILLS (UNTOUCHED)                      │
│                                                                     │
│  /gsd:*         /pdca *        /adr       /gsd-retrospective        │
│  .planning/     docs/01-plan/  docs/      docs/retrospectives/      │
│  STATE.md       .pdca-status   decisions/ work-log in Obsidian      │
│  ROADMAP.md     .json                                               │
├─────────────────────────────────────────────────────────────────────┤
│                     ARTIFACT STORAGE                                 │
│                                                                     │
│  .lifecycle/                                                        │
│  ├── state.json          # Orchestrator stage state                 │
│  ├── manifest.json       # Artifact registry per stage              │
│  ├── history/            # Stage transition log                     │
│  └── prototypes/         # User Interaction Prototypes (HTML+JS)    │
└─────────────────────────────────────────────────────────────────────┘
```

### Component Responsibilities

| Component | Responsibility | Implementation |
|-----------|----------------|----------------|
| Stage Router | Detect user intent, map to stage, enforce ordering, handle skip/shortcut modes | Pattern matching on user input + current state; encoded in SKILL.md instruction logic |
| State Engine | Track current stage, record transitions, enable session resumption | JSON file (`.lifecycle/state.json`) read at session start, written after each transition |
| Artifact Bus | Pass output of stage N as input to stage N+1; validate artifact existence before transition | Manifest file mapping stage -> artifact paths; validation checks before allowing advancement |
| Skill Adapters | Translate orchestrator intent into skill-specific invocations without modifying the skill | Instruction blocks in SKILL.md that compose the right slash command + context |
| Prototype Engine | Generate clickable HTML+JS mockups during PLAN for user validation | HTML file generation in `.lifecycle/prototypes/`; linked to feature specs |

## Recommended Project Structure

```
~/.claude/skills/dev-lifecycle/
├── SKILL.md                    # Main orchestrator skill definition
└── (no other files — everything is instruction-driven)

Per-project artifacts:
.lifecycle/                     # Orchestrator's own state (separate from .planning/)
├── state.json                  # Current stage, timestamps, mode
├── manifest.json               # Artifact registry: stage -> [file paths]
├── history/                    # Transition log entries
│   └── YYYY-MM-DD-HH-MM.json  # Each transition recorded
├── prototypes/                 # User Interaction Prototypes
│   └── feature-name.html      # Clickable mockup
└── specs/                      # End-to-end feature specs
    └── feature-name.spec.md   # Screen→API→DB→Response→Error chain
```

### Structure Rationale

- **`.lifecycle/` separate from `.planning/`:** GSD owns `.planning/`. The orchestrator must not pollute GSD's namespace. Separate directory avoids coupling and makes it clear what belongs to what system.
- **`state.json` not `STATE.md`:** Machine-readable state enables programmatic checks (is artifact X present?). Human-readable state lives in GSD's STATE.md. The orchestrator reads GSD's STATE.md but writes its own state.json.
- **`manifest.json`:** The central registry solving "Stage 7 needs artifacts from Stage 2, 3, and 5." Each stage registers what it produced; downstream stages query what's available.
- **`history/`:** Immutable log. Never edited, only appended. Enables RETROSPECT (Stage 8) to auto-generate timeline.

## Architectural Patterns

### Pattern 1: Adapter Pattern for Skill Wrapping

**What:** Each wrapped skill (GSD, PDCA, ADR, retro, work-log) gets a logical adapter — a section in SKILL.md that knows how to invoke the skill and interpret its outputs — without modifying the skill's source.

**When to use:** Always. This is the core constraint: "GSD/PDCA 본체 수정 — 기존 시스템 위에서 오케스트레이션만 담당."

**Trade-offs:**
- PRO: Existing skills evolve independently. Upgrading GSD or PDCA never breaks the orchestrator.
- PRO: Each adapter is a localized concern — easy to add new skills.
- CON: Adapters must be updated when skill interfaces change (new artifact paths, renamed commands).
- CON: No compile-time guarantee that adapter matches skill — must rely on runtime validation.

**Example:**
```markdown
## Stage 2 Adapter: DO

When entering Stage 2:
1. Check: `.lifecycle/manifest.json` has Stage 1 artifacts (plan exists)
2. Invoke: User triggers `/pdca do {feature}` or `/gsd:execute-phase`
3. Monitor: Watch for trade-off signals → trigger `/adr` if detected
4. Capture: After completion, register artifacts:
   - Code changes (from git diff)
   - ADR documents created (from docs/decisions/)
   - PDCA status (from docs/.pdca-status.json)
5. Write: Update `.lifecycle/manifest.json` with Stage 2 outputs
6. Transition: Suggest Stage 3 (TEST)
```

### Pattern 2: Manifest-Based Artifact Flow

**What:** A JSON manifest tracks what each stage produced. Downstream stages declare what they need. The orchestrator validates presence before allowing transition.

**When to use:** For every stage transition. This solves the core problem: "Stage 간 산출물 연결 — 각 Stage의 출력이 다음 Stage의 입력으로 자동 전달되는 구조."

**Trade-offs:**
- PRO: Explicit dependency chain — no implicit "I hope that file exists."
- PRO: Manifest enables skip-mode validation ("hotfix skips Stage 1, but Stage 3 still needs test criteria — where from?").
- CON: Manifest must be updated religiously. If a skill creates an artifact and the adapter doesn't register it, it's invisible.

**Example:**
```json
{
  "feature": "user-auth",
  "mode": "feature",
  "current_stage": 3,
  "stages": {
    "1_PLAN": {
      "status": "complete",
      "completed_at": "2026-03-22T10:30:00",
      "artifacts": {
        "plan": ".planning/phases/01-user-auth/PLAN.md",
        "prototype": ".lifecycle/prototypes/user-auth.html",
        "spec": ".lifecycle/specs/user-auth.spec.md",
        "pre_flight": ".lifecycle/history/preflight-001.json"
      }
    },
    "2_DO": {
      "status": "complete",
      "completed_at": "2026-03-22T14:00:00",
      "artifacts": {
        "code_commits": ["abc1234", "def5678"],
        "adrs": ["docs/decisions/005-jwt-over-session.md"],
        "pdca_status": "docs/.pdca-status.json"
      }
    },
    "3_TEST": {
      "status": "in_progress",
      "requires": ["1_PLAN.plan", "2_DO.code_commits"]
    }
  }
}
```

### Pattern 3: File-Based State Machine

**What:** Stage transitions are governed by a state machine persisted as JSON. Each transition has preconditions (required artifacts), actions (skill invocations), and postconditions (expected outputs). State survives session boundaries.

**When to use:** Every session start (read state), every stage transition (validate + write).

**Trade-offs:**
- PRO: Session-resilient — new Claude Code session reads state.json and knows exactly where things are.
- PRO: Supports multiple execution modes (full, hotfix, feature, release) by defining different valid transition paths.
- CON: State file can desync from reality if user does things outside the orchestrator (manually commits, deploys).
- MITIGATION: Add a `reconcile` command that scans git history and file system to reconstruct state.

### Pattern 4: Instruction-Driven Architecture (No External Runtime)

**What:** The entire orchestrator is a SKILL.md file — pure instructions that Claude Code follows. No server, no daemon, no CLI tool. The "code" is the natural language instructions.

**When to use:** This is the foundational constraint: "Claude Code 스킬(SKILL.md)로 배포 — 별도 서버나 인프라 없음."

**Trade-offs:**
- PRO: Zero infrastructure. Install = copy one file.
- PRO: Works everywhere Claude Code works.
- CON: No persistent process — everything must be reconstructable from files.
- CON: Complex logic in natural language is harder to debug than code.
- MITIGATION: Keep orchestrator logic as simple state-machine rules. Complex logic belongs in the wrapped skills.

## Data Flow

### Stage Pipeline Flow

```
USER REQUEST
    │
    ▼
STAGE ROUTER ──── reads ──── .lifecycle/state.json
    │                              │
    ├── "What stage am I in?"      │
    ├── "What does user want?"     │
    ▼                              │
PRECONDITION CHECK ◄──────── manifest.json
    │                              │
    ├── "Do required artifacts exist?"
    │                              │
    ▼ (if yes)                     │
SKILL ADAPTER ──── invokes ──── Wrapped Skill (GSD/PDCA/ADR/etc.)
    │                              │
    ├── Skill produces artifacts   │
    │                              │
    ▼                              │
ARTIFACT REGISTRATION ────► manifest.json (updated)
    │                              │
    ▼                              │
STATE TRANSITION ──────────► state.json (updated)
    │                              │
    ▼                              │
HISTORY ENTRY ─────────────► history/YYYY-MM-DD-HH-MM.json
    │
    ▼
SUGGEST NEXT STAGE (or complete)
```

### Cross-Stage Artifact Dependencies

```
Stage 1 (PLAN)
├── PLAN.md ──────────────────────────────► Stage 3 (TEST: verification criteria)
├── prototype.html ───────────────────────► Stage 2 (DO: implementation reference)
├── spec.md ──────────────────────────────► Stage 2 (DO: completeness target)
│                                          Stage 3 (TEST: e2e validation)
│                                          Stage 6 (DEPLOY TEST: smoke test list)
└── pre-flight.json ──────────────────────► Stage 8 (RETROSPECT: what was planned)

Stage 2 (DO)
├── code commits ─────────────────────────► Stage 3 (TEST: what to verify)
│                                          Stage 4 (COMMIT: what to commit)
├── ADR docs ─────────────────────────────► Stage 7 (DOCUMENT: architecture docs)
│                                          Stage 8 (RETROSPECT: decisions review)
└── PDCA status ──────────────────────────► Stage 3 (TEST: gap analysis)

Stage 3 (TEST)
├── VERIFICATION.md ──────────────────────► Stage 4 (COMMIT: confidence to commit)
└── gap list ─────────────────────────────► Stage 2 (DO: rework loop)

Stage 4 (COMMIT)
└── commit hashes + tags ─────────────────► Stage 5 (DEPLOY: what to deploy)

Stage 5 (DEPLOY)
└── deployment info ──────────────────────► Stage 6 (DEPLOY TEST: where to test)

Stage 6 (DEPLOY TEST)
└── smoke test results ───────────────────► Stage 7 (DOCUMENT: deployment status)

Stage 7 (DOCUMENT)
├── architecture diagrams ────────────────► Stage 9 (PROMOTE: demo assets)
└── updated CLAUDE.md ────────────────────► Stage 1 (PLAN: next cycle context)

Stage 8 (RETROSPECT)
├── retrospective doc ────────────────────► Stage 1 (PLAN: lessons for next)
└── work-log entry ───────────────────────► external (Obsidian vault)
```

### State Management Across Sessions

```
SESSION START
    │
    ▼
READ .lifecycle/state.json
    │
    ├── state exists? ──── YES ──► Resume from recorded stage
    │                               Show: "You're in Stage X (DO). Last: 2h ago."
    │                               Check: manifest for partial artifacts
    │
    └── state missing? ──── NO ──► Fresh start
                                   Check: .planning/ exists? (GSD already in use)
                                   If yes: Reconcile — infer stage from existing artifacts
                                   If no: Initialize from Stage 1
```

### Key Data Flows

1. **Plan-to-Implementation flow:** PLAN stage generates feature spec + prototype -> DO stage uses spec as completeness checklist and prototype as UI reference -> TEST stage validates against spec (not just "does it work" but "does it match what was agreed").

2. **Decision tracking flow:** DO stage detects trade-offs -> ADR skill creates decision records -> DOCUMENT stage includes ADR list in architecture docs -> RETROSPECT stage reviews decisions for lessons -> PLAN (next cycle) reads lessons as pre-flight check.

3. **Memory persistence flow:** Each stage's artifacts are registered in manifest.json -> state.json tracks current position -> history/ logs transitions with timestamps -> On session start, state.json + manifest.json reconstruct full context without user re-explaining.

## Scaling Considerations

| Scale | Architecture Adjustments |
|-------|--------------------------|
| 1 project, 1 user | Single `.lifecycle/` directory. Simple state.json. No concurrency concerns. |
| 5-10 projects, 1 user | Each project has its own `.lifecycle/`. SKILL.md is shared (installed once in `~/.claude/skills/`). No changes needed. |
| Complex project, many features | Multiple features in parallel: state.json tracks per-feature stage. Manifest tracks per-feature artifacts. Consider `state.json` -> `states/{feature}.json` split. |

### Scaling Priorities

1. **First bottleneck: SKILL.md size.** As adapters for more skills are added, the SKILL.md grows. Claude Code skills are instruction files — too long and Claude's attention degrades. MITIGATION: Keep SKILL.md under 500 lines. Move detailed adapter logic into referenced files (e.g., `@.lifecycle/adapters/gsd.md`).

2. **Second bottleneck: Manifest complexity.** For projects with 20+ features, manifest.json grows unwieldy. MITIGATION: Archive completed features. Only active features in manifest. Completed features move to `manifest-archive/`.

## Anti-Patterns

### Anti-Pattern 1: Merging State with GSD's STATE.md

**What people do:** Store orchestrator stage state inside `.planning/STATE.md` alongside GSD state.
**Why it's wrong:** Violates the "don't modify existing skills" constraint. GSD's STATE.md has a defined schema. Adding orchestrator fields creates coupling — when GSD updates its state format, orchestrator data gets corrupted or lost.
**Do this instead:** Orchestrator READS `.planning/STATE.md` for GSD context but WRITES its own `.lifecycle/state.json`. Two systems, two state stores, one-way dependency (orchestrator depends on GSD, never reverse).

### Anti-Pattern 2: Intercepting Skill Commands

**What people do:** Override `/gsd:execute-phase` to inject orchestrator logic before/after GSD runs.
**Why it's wrong:** Claude Code skills don't support middleware or hooks in that way. Attempting to intercept creates fragile ordering dependencies and confusing user experience (which skill is running?).
**Do this instead:** Orchestrator suggests the right command to run and validates before/after. The user or orchestrator explicitly calls the skill. Orchestrator is a coordinator, not a proxy.

### Anti-Pattern 3: Encoding All Logic in One SKILL.md

**What people do:** Put the entire orchestrator — all 9 stage adapters, state machine, validation logic, artifact mapping — in a single SKILL.md file.
**Why it's wrong:** SKILL.md becomes 2000+ lines. Claude's attention to instructions degrades with length. Critical rules at line 1800 get ignored.
**Do this instead:** SKILL.md contains: stage overview, router logic, state machine rules. Each stage's detailed adapter instructions live in separate referenced files. SKILL.md stays under 500 lines as an orchestration hub.

### Anti-Pattern 4: Requiring Stage Completion for All Modes

**What people do:** Enforce that every project must go through all 9 stages.
**Why it's wrong:** Hotfixes don't need PLAN. Bug fixes don't need PROMOTE. Forcing all stages creates friction that makes users abandon the system.
**Do this instead:** Define execution modes (full/feature/hotfix/release) with valid stage subsets. Default to suggesting all stages but allow skipping with explicit acknowledgment recorded in history.

## Integration Points

### Wrapped Skills (Read-Only Integration)

| Skill | What Orchestrator Reads | What Orchestrator Does NOT Touch |
|-------|------------------------|----------------------------------|
| GSD | `.planning/STATE.md`, `.planning/ROADMAP.md`, phase plans, VERIFICATION.md, SUMMARY.md | Never writes to `.planning/` — only reads |
| PDCA | `docs/.pdca-status.json`, plan/design/analysis docs | Never writes to `docs/01-plan/` etc. |
| ADR | `docs/decisions/*.md` | Never creates/modifies ADR files directly — invokes `/adr` skill |
| gsd-retrospective | `docs/retrospectives/*.md` | Never creates retro directly — invokes `/gsd-retrospective` |
| work-log | Obsidian vault entries | Never writes to vault — invokes work-log skill |
| canvas-design | Architecture diagrams in `docs/` | Never creates diagrams — invokes canvas-design skill |

### Internal Boundaries

| Boundary | Communication | Notes |
|----------|---------------|-------|
| Stage Router <-> State Engine | Direct function call (in-skill logic) | Router reads state, decides action, writes state |
| Stage Router <-> Skill Adapters | Instruction delegation | Router determines which adapter to activate |
| Skill Adapters <-> Wrapped Skills | Slash command invocation | Adapter suggests/triggers slash command, reads outputs |
| Artifact Bus <-> All Components | Manifest read/write | Central registry, all components register/query |
| State Engine <-> File System | JSON read/write | state.json + history/ |

## Build Order (Dependency-Driven)

Components should be built in this order because each depends on the previous:

```
Phase 1: Foundation
├── State Engine (state.json schema + read/write)
├── Manifest schema (artifact registry definition)
└── Execution mode definitions (full/feature/hotfix/release)

Phase 2: Core Routing
├── Stage Router (intent detection + stage mapping)
├── Stage transition rules (preconditions per stage)
└── Session resumption logic (read state -> present context)

Phase 3: Skill Adapters (Stages 1-4, inner loop)
├── PLAN adapter (GSD + PDCA + pre-flight check)
├── DO adapter (PDCA + ADR monitoring)
├── TEST adapter (GSD verify + PDCA analyze)
└── COMMIT adapter (Git conventions)

Phase 4: User Interaction Prototypes
├── Prototype generator (HTML+JS mockup creation)
├── E2E spec generator (feature spec template)
└── Integration into PLAN adapter

Phase 5: Skill Adapters (Stages 5-9, outer loop)
├── DEPLOY adapter (generic, not muse-specific)
├── DEPLOY TEST adapter (smoke test framework)
├── DOCUMENT adapter (canvas-design + Mermaid)
├── RETROSPECT adapter (gsd-retrospective + ADR gap check + work-log)
└── PROMOTE adapter (optional, manual trigger only)

Phase 6: Polish
├── Reconcile command (reconstruct state from filesystem)
├── Skip-mode validation
└── History visualization
```

**Why this order:**
- State Engine first because everything reads/writes state.
- Router second because it needs state to function.
- Inner loop (1-4) before outer loop (5-9) because inner loop covers 80% of daily use. Get feedback early.
- Prototypes in Phase 4 because they enhance the PLAN adapter (Phase 3) but aren't blocking.
- Outer loop last because deploy/document/promote are less frequent.

## Sources

- Direct analysis of existing skill files:
  - `~/.claude/skills/dev-lifecycle/SKILL.md` (current muse-hardcoded version)
  - `~/.claude/skills/adr/SKILL.md`
  - `~/.claude/skills/gsd-retrospective/SKILL.md`
  - `~/.claude/skills/work-log/SKILL.md`
  - `~/.claude/skills/canvas-design/SKILL.md`
  - `~/.claude/plugins/cache/bkit-marketplace/bkit/1.5.3/skills/pdca/SKILL.md`
- GSD internals:
  - `~/.claude/get-shit-done/templates/state.md` (STATE.md schema)
  - `~/.claude/get-shit-done/references/planning-config.md` (config system)
  - `~/.claude/get-shit-done/workflows/execute-phase.md` (orchestration pattern reference)
- Project context: `.planning/PROJECT.md`

---
*Architecture research for: dev-lifecycle orchestrator*
*Researched: 2026-03-22*
