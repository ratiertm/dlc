# Skill Role Matrix

This document maps all 9 stages of the dev-lifecycle pipeline to their primary and supporting skills. It defines the boundary between dev-lifecycle (orchestrator) and the skills it coordinates.

## Stage-Skill Mapping

| Stage | Stage Name | Primary Skill | Supporting Skills | dev-lifecycle Role |
|-------|------------|---------------|-------------------|-------------------|
| 1 | PLAN | GSD (`/gsd:plan-phase`, `/gsd:discuss-phase`) | PDCA (`/pdca plan`), ADR (trade-off detection) | Trigger pre-flight check, invoke GSD, register plan artifacts |
| 2 | DO | PDCA (`/pdca do`) | ADR (auto-detect decisions), WHY+SEE comments | Track implementation progress, update state.json |
| 3 | TEST | GSD (`/gsd:verify-work`) | PDCA (`/pdca analyze`) | Run verification, compare against manifest, update status |
| 4 | COMMIT | Git (direct) | ADR (reference in commit msg) | Verify artifacts exist before allowing commit |
| 5 | DEPLOY | Project-specific template | None | Provide deploy template, track deploy artifact |
| 6 | DEPLOY TEST | Project-specific template | None | Provide smoke test template, gate next stage |
| 7 | DOCUMENT | canvas-design (optional) | Mermaid (inline) | Trigger documentation generation |
| 8 | RETROSPECT | gsd-retrospective | ADR (gap check), work-log | Invoke retro skill, update CLAUDE.md |
| 9 | PROMOTE | Manual / optional | None | Provide promote checklist |

## Key Principle

dev-lifecycle NEVER executes stage work directly. It:
1. Detects current stage from state.json
2. Invokes the appropriate primary skill
3. Collects outputs into manifest.json
4. Gates transition to the next stage
5. Updates state.json

## User Freedom

Users can invoke GSD, PDCA, ADR skills directly at any time. dev-lifecycle orchestrates when overlap occurs but never blocks direct skill usage.

When a user calls `/gsd:plan-phase` directly (without going through dev-lifecycle), the planning work still happens correctly. If dev-lifecycle is active, it observes the output artifacts and registers them in the manifest. If dev-lifecycle is not active, nothing breaks -- the skills are fully independent.

## Skill Overlap Resolution

When multiple skills could handle a task, dev-lifecycle follows this priority:

| Situation | Resolution |
|-----------|-----------|
| GSD `/gsd:plan-phase` vs PDCA `/pdca plan` | GSD is primary for Stage 1. PDCA supplements with cycle tracking. |
| GSD `/gsd:verify-work` vs PDCA `/pdca analyze` | GSD is primary for Stage 3. PDCA provides gap analysis. |
| ADR auto-detection during Stage 2 | ADR runs in parallel with PDCA. Both outputs are registered. |
| Manual skill invocation during any stage | Always allowed. dev-lifecycle tracks the output if active. |

## Stage Output Types

Each stage produces specific artifact types that get registered in manifest.json:

| Stage | Expected Output Types |
|-------|----------------------|
| 1 PLAN | plan, spec (optional) |
| 2 DO | code, prototype (optional) |
| 3 TEST | verification, test-report |
| 4 COMMIT | commit-hash, tag (optional) |
| 5 DEPLOY | deploy-log, deploy-artifact |
| 6 DEPLOY TEST | smoke-test-report |
| 7 DOCUMENT | architecture-doc, sequence-doc |
| 8 RETROSPECT | retrospective, adr (optional) |
| 9 PROMOTE | demo, announcement (optional) |
