# Feature Research

**Domain:** AI Development Lifecycle Orchestration (Claude Code Skill)
**Researched:** 2026-03-22
**Confidence:** HIGH

## Feature Landscape

### Table Stakes (Users Expect These)

Features users assume exist. Missing these = the orchestrator adds no value over raw GSD/PDCA.

| Feature | Why Expected | Complexity | Notes |
|---------|--------------|------------|-------|
| **9-Stage Pipeline Orchestration** | Core purpose. Without it, this is just another skill collection. PLAN > DO > TEST > COMMIT > DEPLOY > DEPLOY TEST > DOCUMENT > RETROSPECT > PROMOTE must flow as a connected sequence. | MEDIUM | Already exists in current SKILL.md but hardcoded to muse. Must become project-agnostic. |
| **Stage Artifact Chaining** | Each stage's output must feed the next stage's input automatically. If PLAN output doesn't flow to DO, the pipeline is manual copy-paste with extra steps. | MEDIUM | The "artifact = memory" principle from PROJECT.md. PLAN.md feeds DO, VERIFICATION.md feeds COMMIT decisions, etc. |
| **Phase Transition Detection** | User says "done" or "deploy" and the orchestrator knows which stage to activate. Without this, users must remember the stage number. | LOW | Signal table already defined in current SKILL.md. Needs generalization beyond muse-specific signals. |
| **Minimum Execution Modes** | Not every change needs all 9 stages. Hotfix = DO > TEST > COMMIT > DEPLOY > DEPLOY TEST. Feature = PLAN > DO > TEST > COMMIT > RETROSPECT. Full release = all 9. | LOW | Already designed. Key: the orchestrator picks the right mode, not the user. |
| **Retrospective Integration** | Lessons from previous phases must surface before the next phase starts. Without this, the same mistakes repeat across sessions. | MEDIUM | Current gsd-retrospective skill handles generation. Orchestrator must read retrospectives in Stage 1 preflight. |
| **ADR Auto-Detection** | When user makes a tradeoff decision during DO, the orchestrator should trigger ADR creation without being asked. | LOW | ADR skill already has proactive detection. Orchestrator wires it into Stage 2 automatically. |
| **Project-Agnostic Configuration** | Must work with any project type (web, mobile, API, desktop, CLI). No hardcoded paths, deploy commands, or framework assumptions. | MEDIUM | Current SKILL.md has muse-specific rsync/flutter/systemd commands. These must become configurable templates or auto-detected. |
| **Session Context Restoration** | Starting a new Claude Code session must restore full project state. User should not need to re-explain what phase they are in, what was decided, or what remains. | MEDIUM | CLAUDE.md + STATE.md pattern. GSD already has STATE.md. Orchestrator must maintain a "current lifecycle state" file that includes active stage, pending artifacts, and recent decisions. |

### Differentiators (Competitive Advantage)

Features that set this apart from raw GSD/PDCA usage. These solve the three core problems from PROJECT.md.

| Feature | Value Proposition | Complexity | Notes |
|---------|-------------------|------------|-------|
| **User Interaction Prototype (HTML+JS Mockups)** | Solves Problem #1: "Claude says done but feature doesn't work." Clickable HTML+JS mockups generated in PLAN stage let user click through the flow BEFORE code is written. Agreement on what "done" means happens before DO, not after. | HIGH | Generate standalone HTML files with embedded JS that simulate the feature flow. Not pixel-perfect design -- functional flow validation. User clicks Button A, sees Panel B appear, fills Form C, gets Response D. Each interaction point becomes a verification checkpoint. |
| **End-to-End Feature Specification** | Solves Problem #1 from the opposite direction. Instead of "implement login," specify: Screen (login form) > Connection (POST /auth) > Processing (validate credentials) > Response (JWT + redirect) > Error (invalid password message). Each link in the chain becomes a testable assertion. | HIGH | This is the "function exists != feature works" solution. The spec format must be structured enough for the TEST stage to verify each link independently. When any link fails, the exact break point is identified. |
| **Settings Change Tracking** | Solves Problem #2: memory loss. Every config/settings change auto-records WHAT changed, WHEN, WHY (context from conversation), and WHO requested it. Not an ADR (too heavy) -- a lightweight changelog that accumulates automatically. | MEDIUM | Distinct from ADR: ADRs capture architectural decisions between alternatives. Settings tracking captures operational changes (port number changed, API key rotated, feature flag toggled). Format: append-only log file. |
| **Decision History (Lightweight)** | Solves Problem #2: direction changes vanish. Between full ADRs (heavyweight) and nothing (current state for minor decisions), this captures micro-decisions: "switched from approach A to B because X." Lighter than ADR, heavier than a commit message. | MEDIUM | Append-only file. One-liner per decision: `[date] [stage] [decision] [reason]`. Searchable. No template overhead. ADR skill handles the big decisions; this handles the rest. |
| **Living State Document** | A single document that represents the project's current truth. Not a changelog, not a history -- the CURRENT state. What stage are we in? What's deployed? What decisions are active? What's the next action? Read this one file and you know everything. | MEDIUM | Distinct from CLAUDE.md (which is instructions) and STATE.md (which is GSD-specific). This is the orchestrator's "project snapshot" that gets overwritten (not appended) on every stage transition. |
| **Cross-Stage Verification Chain** | TEST stage doesn't just check "does the function exist" -- it verifies the end-to-end spec from PLAN. If PLAN said "button click sends POST to /api/chat," TEST specifically verifies that chain. Gaps are traced back to the exact spec line that failed. | HIGH | Requires structured spec format in PLAN that TEST can parse and verify programmatically. The UAT pattern from GSD verify-work is the starting point, but needs to be enriched with the e2e chain assertions. |
| **Code-to-Decision Linking** | WHY+SEE comments in code link to ADRs. But extend this: when a settings change or micro-decision affects code, the code comment references the decision log entry. Six months later, "why is this port 8443?" has an answer. | LOW | ADR skill already does WHY+SEE for architectural decisions. Extend the pattern to settings changes and micro-decisions. Same comment format, different reference target. |
| **Preflight Check Automation** | Before starting any stage, automatically scan: retrospective lessons, active ADRs for this domain, pending technical debt, PDCA gaps, and settings changes since last session. Present as a checklist, not a wall of text. | MEDIUM | Stage 1 in current SKILL.md does this manually. Automate: grep retrospectives for "Lessons for Next Phase," scan ADRs for the relevant component, check TODO/FIXME counts. |

### Anti-Features (Commonly Requested, Often Problematic)

| Feature | Why Requested | Why Problematic | Alternative |
|---------|---------------|-----------------|-------------|
| **Pixel-Perfect UI Mockups** | "Show me exactly what it will look like." | HTML+JS mockups should validate FLOW, not visual design. Pixel-perfect mockups create false precision -- the final UI will differ. Users argue about colors instead of validating interactions. Also, Claude generates mediocre CSS. | Functional flow mockups (wireframe-level) that validate interaction chains. Use GSD's existing UI-SPEC.md for visual contracts (spacing, color, typography) separately. |
| **Automatic Stage Progression** | "Just run all stages automatically." | Removes the human checkpoints that catch the exact problems this tool exists to solve. If Stage 3 (TEST) auto-advances to Stage 4 (COMMIT), untested code gets committed. The user's confirmation at each gate IS the value. | Minimum execution modes (hotfix mode skips some stages). But never skip TEST or RETROSPECT automatically. |
| **Full Memory System (External DB)** | "Store everything in a vector DB for perfect recall." | Claude Code skills have no server infrastructure. Adding a DB dependency breaks the "just a SKILL.md" constraint. Also, structured artifacts in files ARE the memory -- they are version-controlled, human-readable, and don't require a running service. | File-based artifacts as memory. CLAUDE.md for session bootstrap. Living state document for current truth. Git history for temporal queries. |
| **GSD/PDCA Core Modification** | "Change how GSD plans work to fit the lifecycle." | Out of scope per PROJECT.md. Modifying upstream skills creates maintenance burden and version conflicts. The orchestrator sits ABOVE existing skills. | Orchestration layer only. Wrap, invoke, and chain existing skills. If a skill can't do something, add a new lightweight skill rather than modifying GSD/PDCA. |
| **Real-Time Collaboration** | "Multiple developers using the same lifecycle state." | Claude Code is single-user by design. Adding collaboration means conflict resolution, locking, and state synchronization -- all without a server. | Git-based workflows handle multi-dev naturally. Each developer runs their own lifecycle instance. Merge conflicts in planning docs are resolved like code conflicts. |
| **AI-Generated E2E Test Code** | "Auto-generate Cypress/Playwright tests from the e2e spec." | Generated test code is brittle, maintenance-heavy, and gives false confidence. The spec itself IS the test contract. Manual verification against the spec (via UAT flow) catches real issues. Auto-generated tests catch selector changes. | E2E spec as human-readable verification checklist (current UAT pattern). If the project has an existing test framework, the spec informs what to test -- but the orchestrator doesn't generate test code. |
| **Stage 9 (PROMOTE) as Default** | "Always generate demo videos." | Demo video generation requires Maestro + ADB + FFmpeg -- heavy dependencies most projects don't have. Making it default adds friction for 90% of use cases. | Keep PROMOTE as opt-in. The orchestrator mentions it exists but doesn't require it. |

## Feature Dependencies

```
[Project-Agnostic Configuration]
    └──requires──> [9-Stage Pipeline Orchestration]
                       └──requires──> [Stage Artifact Chaining]
                                          └──requires──> [Session Context Restoration]

[User Interaction Prototype]
    └──requires──> [End-to-End Feature Specification]
                       └──requires──> [Cross-Stage Verification Chain]

[Settings Change Tracking]
    └──enhances──> [Decision History (Lightweight)]
                       └──enhances──> [Code-to-Decision Linking]

[Preflight Check Automation]
    └──requires──> [Retrospective Integration]
    └──requires──> [ADR Auto-Detection]
    └──requires──> [Decision History (Lightweight)]

[Living State Document]
    └──requires──> [Session Context Restoration]
    └──requires──> [Stage Artifact Chaining]

[User Interaction Prototype] ──conflicts──> [Pixel-Perfect UI Mockups]
```

### Dependency Notes

- **End-to-End Feature Spec requires Stage Artifact Chaining:** The e2e spec IS the artifact that chains PLAN to TEST. Without artifact chaining infrastructure, the spec has no way to flow to verification.
- **User Interaction Prototype requires End-to-End Feature Spec:** The clickable mockup is the visual representation of the e2e spec. Without the spec, the mockup has no defined interaction chain to simulate.
- **Cross-Stage Verification Chain requires End-to-End Feature Spec:** Verification needs a structured spec to verify against. Without it, TEST falls back to "does it look right?" which is the current broken pattern.
- **Preflight Check Automation requires three inputs:** Retrospective lessons, ADR decisions, and decision history must all exist before they can be scanned automatically.
- **Living State Document requires both Session Context Restoration and Stage Artifact Chaining:** The living document is the synthesis of artifacts produced by the chain, formatted for session restoration.
- **Settings Change Tracking enhances Decision History:** Settings changes are a subset of decisions. Tracking them feeds into the broader decision history. They share a reference format for code linking.
- **User Interaction Prototype conflicts with Pixel-Perfect UI Mockups:** Building both creates confusion about which mockup is authoritative. Flow mockups and visual contracts (UI-SPEC.md) serve different purposes and should remain separate.

## MVP Definition

### Launch With (v1)

Minimum viable product -- what's needed to validate that the orchestrator solves the three core problems.

- [ ] **Project-Agnostic Pipeline** -- Remove muse hardcoding. Pipeline works with any project type via configurable templates (deploy commands, build commands, paths).
- [ ] **Stage Artifact Chaining** -- Each stage output file becomes the next stage's required input. Define the artifact contract: PLAN produces X, DO consumes X and produces Y, TEST consumes Y.
- [ ] **End-to-End Feature Specification Format** -- Structured spec format: Screen > Connection > Processing > Response > Error. Each link is a testable assertion. This is the core innovation.
- [ ] **Session Context Restoration via Living State Document** -- Single file updated on every stage transition. New session reads it and knows: current stage, active decisions, pending work, next action.
- [ ] **Settings Change Auto-Tracking** -- Append-only log. When the orchestrator detects a config/settings change in conversation, it records what/when/why automatically.
- [ ] **Preflight Check (Manual)** -- Stage 1 reads retrospectives + ADRs + decision history and presents checklist. Not fully automated yet -- but the data sources exist and are consulted.

### Add After Validation (v1.x)

Features to add once the core pipeline is working and tested on a real project.

- [ ] **User Interaction Prototype (HTML+JS)** -- Add when: e2e spec format is stable and users confirm the spec alone isn't enough to prevent "done but broken" issues. The prototype visualizes the spec.
- [ ] **Cross-Stage Verification Chain** -- Add when: UAT pattern proves insufficient for catching e2e gaps. Enriches TEST stage with spec-aware assertions.
- [ ] **Decision History (Lightweight)** -- Add when: settings tracking is working and users find they also need to track non-settings micro-decisions.
- [ ] **Code-to-Decision Linking (Extended)** -- Add when: decision history exists and users want to trace code back to decisions beyond ADRs.
- [ ] **Preflight Check Automation** -- Add when: all data sources (retrospectives, ADRs, decision history, tech debt) are reliably produced by the pipeline.

### Future Consideration (v2+)

Features to defer until the orchestrator is proven on multiple projects.

- [ ] **Multi-Project Orchestration** -- Manage lifecycle state across related projects (e.g., frontend + backend + mobile). Defer because: single-project must work first.
- [ ] **Custom Stage Injection** -- Let users add custom stages (e.g., SECURITY REVIEW between TEST and COMMIT). Defer because: the 9-stage pipeline must prove its structure first.
- [ ] **Metric Aggregation Dashboard** -- Aggregate retrospective metrics across phases to show trends (bug rate, rework frequency). Defer because: needs multiple retrospectives to be meaningful.
- [ ] **Template Marketplace** -- Share project-type-specific templates (web deploy, mobile deploy, API deploy). Defer because: templates must be battle-tested on real projects first.

## Feature Prioritization Matrix

| Feature | User Value | Implementation Cost | Priority |
|---------|------------|---------------------|----------|
| Project-Agnostic Pipeline | HIGH | MEDIUM | P1 |
| Stage Artifact Chaining | HIGH | MEDIUM | P1 |
| End-to-End Feature Spec | HIGH | HIGH | P1 |
| Session Context Restoration | HIGH | MEDIUM | P1 |
| Settings Change Tracking | MEDIUM | LOW | P1 |
| Preflight Check (Manual) | MEDIUM | LOW | P1 |
| User Interaction Prototype | HIGH | HIGH | P2 |
| Cross-Stage Verification | HIGH | HIGH | P2 |
| Decision History | MEDIUM | LOW | P2 |
| Code-to-Decision Linking | MEDIUM | LOW | P2 |
| Preflight Check Automation | MEDIUM | MEDIUM | P2 |
| Multi-Project Orchestration | LOW | HIGH | P3 |
| Custom Stage Injection | LOW | MEDIUM | P3 |
| Metric Aggregation | LOW | MEDIUM | P3 |

**Priority key:**
- P1: Must have for launch -- solves the three core problems
- P2: Should have, add after validating core on a real project
- P3: Nice to have, future consideration after multi-project usage

## Competitor Feature Analysis

| Feature | Raw GSD (current) | Raw PDCA (current) | Dev Lifecycle Orchestrator (our plan) |
|---------|-------------------|-------------------|--------------------------------------|
| Planning | plan-phase with research + discussion | plan with gap analysis | Wraps GSD plan + adds preflight check from retrospectives/ADRs |
| Execution | execute-phase with sub-agents | do with ADR detection | Wraps PDCA do + auto-triggers ADR + settings tracking |
| Verification | verify-work UAT with gap diagnosis | check/analyze cycle | Extends UAT with e2e spec chain verification |
| Context Persistence | STATE.md + CLAUDE.md | pdca-status.json | Living state document + settings log + decision history |
| UI Design | ui-phase with UI-SPEC.md | None | User Interaction Prototype (flow validation, not visual design) |
| Retrospective | gsd-retrospective skill | act phase (partial) | Integrated retrospective that feeds preflight checks |
| Decision Tracking | Separate ADR skill | None | ADR (heavy) + decision history (light) + settings log (automatic) |
| Deployment | None (manual) | None | Configurable deploy templates per project type |
| Feature Completeness Check | UAT pass/fail | None | E2E spec: screen > connection > processing > response > error |

## Sources

- [How Claude remembers your project](https://code.claude.com/docs/en/memory) -- Claude Code CLAUDE.md memory system
- [Extend Claude with skills](https://code.claude.com/docs/en/skills) -- Claude Code skills documentation
- [Addy Osmani LLM Coding Workflow 2026](https://addyosmani.com/blog/ai-coding-workflow/) -- Current state of AI-assisted development workflows
- [AI Code Generation Limitations](https://devops.com/why-ai-based-code-generation-falls-short/) -- Why AI code generation produces incomplete features
- [CodeRabbit AI vs Human Code Report](https://www.coderabbit.ai/blog/state-of-ai-vs-human-code-generation-report) -- AI code creates 1.7x more issues than human code
- [Lightweight Architecture Decision Records](https://www.thoughtworks.com/radar/techniques/lightweight-architecture-decision-records) -- Thoughtworks recommendation for lightweight ADRs
- [OneContext Persistent Context Layer](https://supergok.com/onecontext-persistent-context-layer-ai-coding-agents/) -- Emerging solutions for AI coding agent memory persistence
- [Awesome Claude Skills](https://github.com/travisvn/awesome-claude-skills) -- Community Claude Code skills catalog
- [Martin Fowler: Pushing AI Autonomy](https://martinfowler.com/articles/pushing-ai-autonomy.html) -- Boundaries of AI code generation autonomy

---
*Feature research for: AI Development Lifecycle Orchestration*
*Researched: 2026-03-22*
