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
   - If not exists: initialize from `$CLAUDE_SKILL_DIR/templates/state.json`, create `.lifecycle/` directory
2. **Display position:**
   ```
   Dev Lifecycle: Stage {N} {NAME} -- {STATUS}
   Feature: {feature_name}
   Resume: {resume_hint}
   ```
3. **First run:** Detect project type
   - Read: `$CLAUDE_SKILL_DIR/references/project-detection.md`
   - Auto-detect from marker files, present to user for confirmation
   - Store confirmed type in `state.json`
4. **Check gates:** Verify manifest.json for current stage requirements

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

Purpose: Implement the plan. Track progress and decisions.
Primary skill: PDCA (`/pdca do`)
Supporting: ADR (auto-detect trade-offs), WHY+SEE code comments
Outputs: Code changes, ADR documents, progress updates

During implementation:
- Detect trade-offs ("chose A over B") and suggest ADR creation
- Insert WHY+SEE comments at decision points in code
- Flag large direction changes for PLAN update

### Stage 3: TEST

Purpose: Verify implementation against plan criteria.
Primary skill: GSD (`/gsd:verify-work`)
Supporting: PDCA (`/pdca analyze`)
Outputs: Verification report, gap analysis

Verification items:
- Plan-vs-implementation completeness
- Functional correctness (project-type-specific checks)
- Error log review
- Visual/behavioral confirmation where applicable

On failure: generate gap list, return to Stage 2.

### Stage 4: COMMIT

Purpose: Save verified work with meaningful commit history.
Tool: Git (direct)
Supporting: ADR (reference in commit message)
Outputs: Commit hash, version tag (optional)

Rules:
- Commit message focuses on "why", not "what"
- Version tag format: `v{major}.{minor}.{patch}`
- Exclude sensitive files (`.env`, credentials)
- Reference related ADRs: `(ADR-NNN)` in commit message

### Stage 5: DEPLOY

Purpose: Ship to target environment.
Tool: Project-type-specific (see templates/)
Outputs: Deploy log, deploy artifact

Note: Deploy steps vary by project type. See `$CLAUDE_SKILL_DIR/references/project-detection.md` for supported types. Stage-specific checklists will be provided based on detected project type.

### Stage 6: DEPLOY TEST

Purpose: Verify deployment in target environment.
Tool: Project-type-specific smoke tests
Outputs: Smoke test report

Verification targets:
- Service health check (endpoint responds)
- Authentication flow works
- Core user flow completes end-to-end
- Resource utilization is within bounds
- Device/browser-specific checks where applicable

On failure: check logs, rollback or hotfix.

### Stage 7: DOCUMENT

Purpose: Record project state visually and textually after successful deploy.
Primary skill: canvas-design (optional)
Supporting: Mermaid diagrams (inline)
Outputs: Architecture doc, sequence diagrams, README/CLAUDE.md updates

Documentation targets:
- Architecture diagram (system overview, component relationships)
- Sequence diagrams (key user flows)
- CLAUDE.md / README.md updates (ADR list, retro links, deploy info)

### Stage 8: RETROSPECT

Purpose: Reflect on the phase. Capture lessons before they fade.
Primary skill: gsd-retrospective
Supporting: ADR (gap check), work-log
Outputs: Retrospective document, missing ADRs, CLAUDE.md update

Steps:
1. Generate retrospective (what went well/wrong/surprising)
2. Link key decisions to ADRs
3. Identify technical debt
4. Record lessons for next phase
5. Check for unrecorded decisions, suggest ADR creation
6. Update CLAUDE.md with retro link and key lessons

### Stage 9: PROMOTE

Purpose: Share the work. Demo, announce, celebrate.
Tool: Manual or automated (optional)
Outputs: Demo recording, announcement (optional)

This stage is optional. Useful for: portfolio, team demos, stakeholder updates, app store screenshots.

## State Management

Dev-lifecycle tracks state in the project's `.lifecycle/` directory (separate from GSD's `.planning/`):

| File | Purpose |
|------|---------|
| `.lifecycle/state.json` | Current stage, status, feature, mode |
| `.lifecycle/manifest.json` | Artifact registry, stage connections |
| `.lifecycle/history/` | Stage transition log (timestamped JSON entries) |

On first invocation:
1. Create `.lifecycle/` directory
2. Copy `$CLAUDE_SKILL_DIR/templates/state.json` as starting state
3. Copy `$CLAUDE_SKILL_DIR/templates/manifest.json` as empty manifest
4. Detect project type and store in state.json

Read: `$CLAUDE_SKILL_DIR/references/stage-transitions.md` for gate logic and status values.

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
