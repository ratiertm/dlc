---
name: dev-lifecycle
description: >-
  Full-cycle development orchestrator. Coordinates 9 stages from planning
  through deployment and documentation. Use when: "lifecycle", "dev lifecycle",
  "phase workflow", "start phase", "finish phase", "next stage", "stage transition",
  "stage gate", "artifact check", "phase complete",
  "개발 라이프사이클", "스테이지", "다음 단계", "전체 워크플로",
  "rework prevention", "재작업 방지".
---

# Dev Lifecycle Orchestrator

Full development lifecycle from planning to production, with built-in rework prevention. Each stage links to the next -- no context falls through the cracks.

## The 9-Stage Pipeline

```
  INNER LOOP (feature development)
  ┌──────────────────────────────────────────────┐
  │  1. PLAN ──> 2. DO ──> 3. TEST ──> 4. COMMIT │
  │     Plan       Build     Verify      Save     │
  └──────────────────────────────────────────────┘

  OUTER LOOP (release and reflection)
  ┌────────────────────────────────────────────────────────────────────┐
  │  5. DEPLOY ──> 6. DEPLOY TEST ──> 7. DOCUMENT ──> 8. RETROSPECT  │
  │     Ship         Smoke test         Record          Reflect       │
  │                                                                   │
  │  9. PROMOTE                                                       │
  │     Share                                                         │
  └────────────────────────────────────────────────────────────────────┘
```

## Session Start

On session start (or after compaction):

1. **Read state:** Load `.lifecycle/state.json`
   - If not exists AND `.lifecycle/` directory exists: **Reconcile** (Read: `$CLAUDE_SKILL_DIR/references/state-reconcile.md`)
   - If not exists AND no `.lifecycle/`: Initialize from `$CLAUDE_SKILL_DIR/templates/state.json`, create `.lifecycle/` directory
2. **Read Living State (if exists):** Load `.lifecycle/LIVING-STATE.md`
   - This single file restores full project context: current state, active settings, recent decisions, event timeline, key ADRs, and resume hint
   - If not exists: skip (first session or pre-Phase-8 project)
   - If exists: use its contents to inform all subsequent interactions in this session
   - Note: This is also loaded automatically by SessionStart hook (.claude/settings.json)
3. **Display position:**
   ```
   Dev Lifecycle: Stage {N} {NAME} -- {STATUS}
   Feature: {feature_name}
   Mode: {mode} (skipping stages: {skippable_list or "none"})
   Resume: {resume_hint}
   ```
4. **First run:** Detect project type
   - Read: `$CLAUDE_SKILL_DIR/references/project-detection.md`
   - Auto-detect from marker files, present to user for confirmation
   - Store confirmed type in `state.json`
5. **Check gates:** Verify manifest.json for current stage requirements

## Stage Overview

### Stage 1: PLAN

Purpose: Define scope, review lessons, generate E2E spec and clickable prototype, get user agreement.
Primary: dev-lifecycle (spec + prototype generation)
Supporting: GSD (broader project planning if needed), PDCA (`/pdca plan`), ADR (trade-off detection)
Outputs: E2E spec (.spec.md), clickable prototype (.prototype.html), pre-flight check

Read: `$CLAUDE_SKILL_DIR/references/plan-stage.md`

Pipeline:
1. Preflight check (retrospective lessons first, non-blocking warnings)
2. E2E Spec generation (fill spec-template.md, all 5 chain steps)
3. Prototype generation (from spec, dual spec linking)
4. Agreement gate (user clicks prototype in browser, hard block until approval)

### Stage 2: DO

Purpose: Implement against approved spec. Track progress, log deviations, detect ADR-worthy decisions.
Primary: dev-lifecycle (spec-driven implementation tracking)
Supporting: ADR (auto-detect trade-offs), WHY+SEE code comments, PDCA (quality support)
Outputs: Code changes, updated spec (step statuses + deviations), ADR documents

Read: `$CLAUDE_SKILL_DIR/references/do-stage.md`

Pipeline:
1. Stage init (gate check, spec load, status update)
2. Spec load and checklist build (read spec from disk, display progress)
3. Per-step implementation (implement each chain step, add SPEC comments)
4. Deviation handling (detect, present to user, record with approval)
5. ADR detection (deviation + tradeoff triggers, suggest ADR creation)
6. Completion (artifact registration, state update)

### Stage 3: TEST

Purpose: Verify implementation against E2E spec (per-step pass/fail), compare prototype manifest structure against code, generate user behavioral verification checklist. Dual gate: both Claude structural and user behavioral verification required before COMMIT.
Primary: dev-lifecycle (spec + prototype verification)
Supporting: GSD (`/gsd:verify-work`), PDCA (`/pdca analyze`)
Outputs: Verification report (verification.json + verification.md), user behavioral checklist

Read: `$CLAUDE_SKILL_DIR/references/test-stage.md`

Pipeline:
1. Stage init (gate check, spec + prototype load, deviation scan)
2. Spec step enumeration (parse all steps, display verification plan)
3. Per-step spec verification (check code against each step's criteria)
4. Prototype structural comparison (manifest vs implementation)
5. User behavioral checklist generation (spec steps -> user actions)
6. Completion (generate reports, dual gate enforcement)

### Stage 4: COMMIT

Purpose: Verify all spec steps passed, validate required artifacts exist, generate why-centric commit message with ADR references. Blocks on verification failures with specific gap list. Dual gate: both Claude structural and user behavioral verification must have passed in TEST stage.
Primary: dev-lifecycle (verification gate + commit orchestration)
Supporting: Git (direct commit execution), ADR (reference in commit msg)
Outputs: Commit hash, version tag (optional)

Read: `$CLAUDE_SKILL_DIR/references/commit-stage.md`

Pipeline:
1. Stage initialization (gate check 3_to_4, content-level verification.json check)
2. Verification gate display (gap list if blocked, override option)
3. Commit preparation (collect files, detect ADRs, generate why-centric message)
4. Git commit execution (stage files, exclude sensitive/runtime, capture hash)
5. Completion (register outputs in manifest, update state, announce)

### Stage 5: DEPLOY

Purpose: Ship to target environment using project-type-specific checklist. User executes deploy steps; Claude tracks progress.
Primary: dev-lifecycle (deploy orchestration)
Supporting: Project-specific tools
Outputs: Deploy log, deploy artifact

Read: `$CLAUDE_SKILL_DIR/references/deploy-stage.md`

Pipeline:
1. Stage init (gate 4_to_5, mode skip check)
2. Project type resolution
3. Deploy checklist (type-specific)
4. Deploy execution (user-guided)
5. Completion

### Stage 6: DEPLOY TEST

Purpose: Verify deployment with 3-category smoke tests: health, core flow, resources.
Primary: dev-lifecycle (smoke test orchestration)
Supporting: Project-specific tools
Outputs: Smoke test report

Read: `$CLAUDE_SKILL_DIR/references/deploy-test-stage.md`

Pipeline:
1. Stage init (gate 5_to_6, mode skip check)
2. Smoke test plan
3. Health check
4. Core flow verification
5. Resource check
6. Completion

### Stage 7: DOCUMENT

Purpose: Record project state with Mermaid architecture/sequence diagrams and update README/CLAUDE.md.
Primary: dev-lifecycle (documentation orchestration)
Supporting: Mermaid (inline), canvas-design (optional)
Outputs: Architecture doc, sequence diagrams

Read: `$CLAUDE_SKILL_DIR/references/document-stage.md`

Pipeline:
1. Stage init (gate 6_to_7, mode skip check)
2. Architecture diagram (Mermaid)
3. Sequence diagrams (Mermaid)
4. README/CLAUDE.md update
5. Completion

### Stage 8: RETROSPECT

Purpose: Reflect on the feature cycle. Delegate to gsd-retrospective, check ADR gaps, record work log, update Living State.
Primary: dev-lifecycle (retrospective orchestration)
Supporting: gsd-retrospective, ADR (gap check), work-log
Outputs: Retrospective document, missing ADRs (optional)

Read: `$CLAUDE_SKILL_DIR/references/retrospect-stage.md`

Pipeline:
1. Stage init (gate 7_to_8, mode skip check)
2. Retrospective generation (delegate to gsd-retrospective)
3. ADR gap check
4. Work log (delegate to work-log)
5. Living State update
6. Completion

### Stage 9: PROMOTE

Purpose: Optionally share the work via demo, announcement, or screenshots. Skip without penalty.
Primary: dev-lifecycle (optional promotion)
Supporting: None
Outputs: Demo, announcement (optional)

Read: `$CLAUDE_SKILL_DIR/references/promote-stage.md`

Pipeline:
1. Stage init (gate 8_to_9)
2. Skip check (default: skip)
3. Demo type selection
4. Demo creation guidance
5. Completion

## State Management

Dev-lifecycle tracks state in the project's `.lifecycle/` directory (separate from GSD's `.planning/`):

| File | Purpose |
|------|---------|
| `.lifecycle/state.json` | Current stage, status, feature, mode |
| `.lifecycle/manifest.json` | Artifact registry, stage connections |
| `.lifecycle/history/` | Stage transition log (timestamped JSON entries) |
| `.lifecycle/LIVING-STATE.md` | Context snapshot for session restoration |
| `.lifecycle/settings-changelog.md` | Settings change history with reasons |
| `.lifecycle/decisions.md` | Lightweight decision log |

On first invocation:
1. Create `.lifecycle/` directory
2. Copy `$CLAUDE_SKILL_DIR/templates/state.json` as starting state
3. Copy `$CLAUDE_SKILL_DIR/templates/manifest.json` as empty manifest
4. Detect project type and store in state.json

Read: `$CLAUDE_SKILL_DIR/references/stage-transitions.md` for gate logic and status values.

## Memory & Decision Trail

Dev-lifecycle automatically maintains memory files in `.lifecycle/` to prevent context loss across sessions.

| File | Purpose | Updated When |
|------|---------|-------------|
| `.lifecycle/LIVING-STATE.md` | Session-start context snapshot (read this ONE file to restore full context) | Every stage transition + session end |
| `.lifecycle/settings-changelog.md` | Append-only record of all settings/config changes with reasons | DO completion + COMMIT completion |
| `.lifecycle/decisions.md` | Lightweight one-line decision entries (below ADR threshold) | DO completion (when decisions made) |

Templates: `$CLAUDE_SKILL_DIR/templates/living-state.md`, `settings-changelog.md`, `decisions.md`

### WHY+SEE Code Comments

When an ADR is created during DO stage, related code receives WHY+SEE comments linking implementation to the decision record:

```
// SPEC: e2e-{feature}-{NNN} -- {step title}
// WHY: {one-line decision rationale}
// SEE: docs/decisions/{NNN}-{slug}.md
```

- `// SPEC:` always comes first (structural link to spec step)
- `// WHY:` + `// SEE:` follow (decision context from ADR)
- Full workflow: `$CLAUDE_SKILL_DIR/references/do-stage.md` Step 4 (ADR Detection)
- ADR creation: Delegated to ADR skill (`~/.claude/skills/adr/SKILL.md`)

Read: `$CLAUDE_SKILL_DIR/references/do-stage.md` for full ADR detection + WHY+SEE insertion logic.

## Stage Transitions

Each stage has a gate. The gate checks `manifest.json` for required artifacts before allowing advancement to the next stage.

- Gates use existence checks: "does Stage N have at least one registered output?"
- Failed gates set status to `blocked` and list missing artifacts
- Backward transitions (e.g., TEST -> DO on failure) skip gate checks
- All transitions are recorded in `.lifecycle/history/`

Read: `$CLAUDE_SKILL_DIR/references/stage-transitions.md`

## Execution Modes

| Mode | Required Stages | Skippable Stages | Use Case |
|------|----------------|-----------------|----------|
| hotfix | 2, 3, 4, 5, 6 | 1, 7, 8, 9 | Urgent bug fix |
| feature | 1, 2, 3, 4, 8 | 5, 6, 9 | Feature development |
| release | 1, 2, 3, 4, 5, 6, 7, 8 | 9 | Full release |
| milestone | All (1-9) | None | Complete lifecycle |

Note: Mode selection is stored in `state.json` for forward compatibility. Advanced mode-based behavior will be enhanced in later phases.

## Skill Orchestration

dev-lifecycle orchestrates existing skills. It never executes stage work directly.

The orchestration pattern:
1. Detect current stage from `state.json`
2. Identify primary skill for that stage
3. Suggest the skill invocation command to the user
4. Track output artifacts in `manifest.json`
5. Gate transition to the next stage

Read: `$CLAUDE_SKILL_DIR/references/skill-invocation.md`
Read: `$CLAUDE_SKILL_DIR/references/role-matrix.md`

Users can invoke GSD, PDCA, ADR, or any skill directly at any time. dev-lifecycle tracks outputs when active but never blocks direct skill usage.

## Project Type Detection

Auto-detects project type from marker files (package.json, pubspec.yaml, Cargo.toml, go.mod, etc.), presents to user for confirmation.

Supports multi-type projects (e.g., React frontend + Python backend). Detected type determines stage-specific templates for deploy, test, and build steps.

Read: `$CLAUDE_SKILL_DIR/references/project-detection.md`

## Phase Transition Detection

| Signal | Transition | Action |
|--------|-----------|--------|
| "phase start" / "start phase" | Explicit start | Stage 1 (PLAN) |
| `/gsd:execute-phase` | GSD execution | Stage 1 check, then Stage 2 |
| "done" / "complete" | Implicit completion | Stage 3 (TEST) |
| "commit" | Commit request | Stage 4 (COMMIT) |
| "deploy" | Deploy request | Stage 5 (DEPLOY) |
| "document" / "diagram" | Doc request | Stage 7 (DOCUMENT) |
| "retro" / "retrospective" | Retro request | Stage 8 (RETROSPECT) |
| `/gsd:complete-milestone` | Milestone complete | Stage 3 through 8 full run |

## Edge Cases

- **First phase:** No retrospective/ADR exists yet. Skip Stage 1 pre-flight retrospective items. Stage 8 creates the first retrospective for future phases.
- **Mid-session join:** Start from current stage. Do not retroactively run earlier stages.
- **User refuses stages:** Minimum is Stage 8 (retrospective). A 5-line retro is better than none.
- **Multiple phases behind on retros:** Offer consolidated vs individual retrospectives.

## Output Language

Match the user's language. Technical terms (GSD, ADR, PDCA, Git) stay in English.

<!-- Architecture Audit (Phase 10)
ARCH-01: SKILL.md 327 lines (limit: 500) -- PASS
ARCH-02: All 9 stages use Read directives, no inline logic -- PASS
References: 16/15 files present (includes state-reconcile.md added in 10-01) -- PASS
Templates: 7/7 files present -- PASS
Orchestration: "never executes stage work directly" confirmed (line 283) -- PASS
Role Matrix: All 9 stages list dev-lifecycle as Primary -- PASS
Supporting Skills: Referenced by name only, no modification instructions -- PASS
Skill Invocation: Commands reference existing skill entry points -- PASS
Audit date: 2026-03-22
-->
