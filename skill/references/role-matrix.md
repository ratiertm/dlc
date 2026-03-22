# Skill Role Matrix

This document maps all 9 stages of the dev-lifecycle pipeline to their primary and supporting skills. It defines the boundary between dev-lifecycle (orchestrator) and the skills it coordinates.

## Stage-Skill Mapping

| Stage | Stage Name | Primary Skill | Supporting Skills | dev-lifecycle Role |
|-------|------------|---------------|-------------------|-------------------|
| 1 | PLAN | dev-lifecycle (spec + prototype generation) | GSD (broader planning), PDCA (`/pdca plan`), ADR (trade-off detection) | Run preflight, generate spec + prototype, enforce agreement gate |
| 2 | DO | dev-lifecycle (spec-driven implementation) | PDCA (quality support), ADR (auto-detect decisions), WHY+SEE comments | Run spec checklist, track per-step status, log deviations, detect ADR moments |
| 3 | TEST | dev-lifecycle (spec + prototype verification) | GSD (`/gsd:verify-work`), PDCA (`/pdca analyze`) | Run per-step spec verification, prototype structural comparison, generate behavioral checklist, enforce dual verification gate |
| 4 | COMMIT | dev-lifecycle (verification gate + commit orchestration) | Git (direct), ADR (reference in commit msg) | Enforce verification gate, generate why-centric commit, register outputs |
| 5 | DEPLOY | dev-lifecycle (deploy orchestration) | Project-specific tools | Provide type-specific deploy checklist, track deploy progress, register artifacts |
| 6 | DEPLOY TEST | dev-lifecycle (smoke test orchestration) | Project-specific tools | 3-category smoke test (health, core flow, resources), gate next stage |
| 7 | DOCUMENT | dev-lifecycle (documentation orchestration) | Mermaid (inline), canvas-design (optional) | Generate Mermaid architecture + sequence diagrams, update README/CLAUDE.md |
| 8 | RETROSPECT | dev-lifecycle (retrospective orchestration) | gsd-retrospective, ADR (gap check), work-log | Orchestrate retrospective, ADR gap check, work log, Living State update |
| 9 | PROMOTE | dev-lifecycle (optional promotion) | None | Skip-or-proceed flow, demo creation guidance |

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
| dev-lifecycle vs GSD in Stage 1 | dev-lifecycle is primary for spec + prototype generation. GSD is available for broader project planning (roadmap, phases). These are complementary, not competing. |
| dev-lifecycle vs GSD in Stage 3 | dev-lifecycle is primary for spec-driven and prototype structural verification. GSD is available for broader project-level verification. PDCA provides gap analysis. These are complementary. |
| dev-lifecycle vs PDCA in Stage 2 | dev-lifecycle is primary for spec-driven implementation tracking. PDCA is available for quality cycles if user invokes directly. These are complementary. |
| ADR auto-detection during Stage 2 | ADR runs in parallel with implementation. Both outputs are registered. |
| dev-lifecycle vs Git in Stage 4 | dev-lifecycle is primary for COMMIT orchestration (verification gate, message generation, artifact validation). Git is the direct tool for commit execution. These are complementary. |
| Manual skill invocation during any stage | Always allowed. dev-lifecycle tracks the output if active. |
| dev-lifecycle vs project tools in Stage 5 | dev-lifecycle is primary orchestrator providing the deploy checklist. Project-specific tools (Vercel CLI, Docker, etc.) are the execution tools. User runs deploy commands, Claude tracks progress. |
| dev-lifecycle vs gsd-retrospective in Stage 8 | dev-lifecycle is primary orchestrator. gsd-retrospective is the delegated skill that generates the retrospective document. dev-lifecycle coordinates the full flow (retro + ADR gap + work-log + Living State). |
| dev-lifecycle vs canvas-design in Stage 7 | dev-lifecycle uses Mermaid by default for all diagrams. canvas-design is optional enhancement if user requests visual design tools. |

## Stage Output Types

Each stage produces specific artifact types that get registered in manifest.json:

| Stage | Expected Output Types |
|-------|----------------------|
| 1 PLAN | spec (required), prototype (required) |
| 2 DO | code, spec-updated (required), adr (optional) |
| 3 TEST | verification (required), test-report (required) |
| 4 COMMIT | commit-hash (required), tag (optional) |
| 5 DEPLOY | deploy-log, deploy-artifact |
| 6 DEPLOY TEST | smoke-test-report |
| 7 DOCUMENT | architecture-doc, sequence-doc |
| 8 RETROSPECT | retrospective, adr (optional) |
| 9 PROMOTE | demo, announcement (optional) |
