# Project Research Summary

**Project:** Dev-Lifecycle Orchestrator (Claude Code Skill)
**Domain:** AI-assisted development lifecycle orchestration
**Researched:** 2026-03-22
**Confidence:** HIGH

## Executive Summary

This project builds a Claude Code skill that orchestrates the full development lifecycle (PLAN, DO, TEST, COMMIT, and beyond) with two core innovations that attack the fundamental problem: AI-generated features have a 100% failure rate on completeness. Buttons are missing, API calls are not wired, error handling is absent. The User Interaction Prototype (UIP) gives the user a clickable HTML file to validate interaction flow before code is written. The End-to-End Feature Spec (E2E Spec) gives the machine a structured 5-layer chain (Screen, Connection, Processing, Response, Error) to verify implementation completeness layer by layer. Together they define what "done" means -- the prototype is the visual contract for the human, the spec is the functional contract for the machine.

The recommended approach builds entirely on Claude Code's native skill system (SKILL.md files with YAML frontmatter), hooks for deterministic lifecycle enforcement, subagents for parallel execution, and a Node.js CLI utility for state management. No external frameworks, servers, or databases. The orchestrator follows the proven GSD pattern: a lean orchestrator coordinates stage transitions while subagents execute with fresh context. All state persists in markdown/JSON files, making the system resilient to session loss and context compaction. Prototypes are vanilla HTML+CSS+JS single files using hash-based routing -- zero dependencies, open with file:// in any browser.

The three primary risks are: (1) self-verification blindness -- Claude generates both spec and code, sharing the same blind spots, mitigated by mandatory human click-through gates; (2) prototype-implementation drift -- implementation silently diverges from spec during DO, mitigated by making the E2E spec (not the prototype) the binding contract with a deviation log; and (3) verification theater -- structural checks pass but features are behaviorally broken, mitigated by distinguishing Claude's structural verification from the user's behavioral verification and producing manual test steps. These risks share a common mitigation: human gates are non-negotiable at every stage boundary.

## Key Findings

### Recommended Stack

The entire project runs as Claude Code skills with no external infrastructure. The "stack" is structured markdown, shell scripts, and a single-file Node.js CLI.

**Core technologies:**
- **Claude Code Skills (SKILL.md)** -- primary orchestration mechanism. Cross-platform Agent Skills 1.0 standard. YAML frontmatter for routing, markdown body for instructions, supporting files for depth. Must stay under 500 lines (progressive disclosure mandatory).
- **Claude Code Hooks** -- deterministic lifecycle enforcement. Fires on SessionStart (re-inject state after compaction), PreToolUse (stage gates -- no code edits during PLAN), SubagentStop (collect results). Not probabilistic like skills.
- **Claude Code Subagents** -- parallel task execution with fresh 200k context per agent. Model routing: haiku for exploration, sonnet for execution, opus for planning. Up to 10 simultaneous.
- **Node.js CLI (gsd-tools.cjs)** -- single-file CJS utility for state management, config parsing, git operations, frontmatter CRUD. Zero dependencies. Node 18+.
- **Vanilla HTML+CSS+JS** -- prototype format. Single file per feature. Hash-based SPA routing (works with file:// protocol). Embedded JSON manifest with data attributes (data-spec-id, data-screen, data-action, data-field). Wireframe-level styling only.

**Critical version note:** Skills share 2% of context window for descriptions. Progressive disclosure (SKILL.md < 500 lines + supporting files loaded on demand) is not optional.

### Expected Features

**Must have (table stakes -- all P1):**
- E2E Feature Spec format with 5-layer chain per feature interaction
- E2E Spec generation during PLAN with user review and agreement
- E2E Spec as DO-stage implementation checklist with per-layer status tracking
- E2E Spec as TEST-stage verification criteria with per-layer PASS/FAIL reporting
- User Interaction Prototype generation (single HTML file with embedded interaction checklist and manifest)
- UIP user agreement flow (click-through, feedback loop, sign-off as hard gate)
- Spec ID linking across all artifacts (e2e-NNN in spec, data-spec-id in prototype, comments in code, keys in verification.json)

**Should have (differentiators -- P2):**
- State machine visualization in prototypes
- Layer-level progress dashboard during DO
- Data flow annotations (togglable overlay showing API calls)
- Cross-feature connection mapping

**Defer (v2+):**
- Regression chain tracking across spec versions
- Automated diff-against-implementation tool
- Spec template library for common patterns (CRUD, auth, file upload)

**Anti-features (never build):**
- Pixel-perfect UI mockups in prototypes (validate flow, not visual design)
- Auto-generated Playwright/Cypress tests from spec (brittle, false confidence)
- Gherkin/BDD format (hides the connection chain -- the exact layer where bugs live)
- Real API calls in prototypes (breaks the zero-dependency constraint)
- AI self-verification without human gate (the broken pattern we are fixing)

### Architecture Approach

The architecture centers on a dual-artifact PLAN output: every feature produces both an E2E spec and a clickable prototype, linked by shared spec IDs (e2e-NNN). These flow through DO (spec-as-checklist with status tracking) and TEST (two-pass verification: spec compliance + prototype structural diff). A verification gate blocks COMMIT until all spec steps reach "verified" status. Features are grouped by directory (.lifecycle/features/{name}/ containing spec.md, prototype.html, and verification.json).

**Major components:**
1. **E2E Spec Generator** -- produces structured 5-layer chain per feature during PLAN
2. **Prototype Generator** -- produces clickable HTML+JS with data-spec-id attributes and embedded coverage JSON
3. **Spec Checklist Engine** -- presents spec as implementation TODO during DO, updates per-step status
4. **Prototype Differ** -- compares prototype structure against implementation at TEST (conceptual, not DOM)
5. **Verification Gate** -- blocks COMMIT until all spec steps pass; produces specific gap list for DO-TEST rework loop (max 3 loops before escalating to user)

**Key architectural patterns:**
- Progressive disclosure: SKILL.md < 500 lines, supporting files loaded on demand
- Orchestrator-executor separation: orchestrator never touches code directly, always delegates to subagents
- Filesystem state: STATE.md/state.json survives sessions, compactions, crashes
- Spec ID as cross-stage thread: e2e-NNN appears in every artifact from PLAN to COMMIT
- Dual-artifact PLAN: spec + prototype are inseparable; neither is optional for feature work

### Critical Pitfalls

1. **Self-Verification Blindness** -- Claude generates both spec and implementation, sharing the same blind spots. Prototype confirms implementation, but both may be wrong in the same way. Avoid by requiring human click-through, explicit "what did I miss?" self-audit, and user sign-off as a hard gate. User clicks the prototype, not Claude.

2. **Prototype-Implementation Drift** -- implementation silently diverges from spec when Claude encounters technical constraints during DO. Avoid by making E2E spec (not prototype) the binding contract, requiring a deviation log for any spec departures, and implementing one spec step at a time with immediate status updates.

3. **Verification Theater** -- structural checks pass (handler exists, route matches) but feature is behaviorally broken. Avoid by distinguishing Claude's structural verification from user's behavioral verification. TEST must produce specific manual test steps the user can execute. Both gates required for COMMIT.

4. **Over-Specification** -- prototype evolves into a mini-app, building everything twice. 33 min / 2,577 lines of spec for 689 lines of code is a real measured outcome. Avoid by hard constraint: single HTML file, no build step, time-boxed to 15 minutes, disposable after approval. One feedback round, one revision, done.

5. **E2E Format Gaps** -- 5-layer chain covers request-response but misses state management, concurrent interactions, loading states, preconditions, and side effects. Avoid by starting with 5 layers but including preconditions and edge cases sections. Accept that some gaps are discovered during DO -- the key is having a place to record them.

## Implications for Roadmap

Based on combined research findings, suggested phase structure:

### Phase 1: Foundation + Spec-Prototype Inner Loop

**Rationale:** The spec+prototype system IS the core value proposition. Architecture research explicitly warns: "The previous ARCHITECTURE.md placed prototypes in Phase 4. This is wrong. Prototypes and E2E specs are the core value proposition. They must be in Phase 1 or early Phase 2." Building foundation without them produces a pipeline that routes between stages but does not solve the 100% failure rate problem.
**Delivers:** Working PLAN-DO-TEST loop for a single feature. State management (state.json), artifact registry (manifest.json), E2E spec format definition, prototype template, PLAN adapter (produces spec.md + prototype.html), DO adapter (reads spec as checklist, updates status), TEST adapter (verifies spec + structural diff), verification gate (blocks COMMIT if any step failed).
**Addresses:** E2E Spec format, spec generation, spec-as-checklist, spec verification, prototype generation, user agreement flow, spec ID linking, verification gate
**Avoids:** Pitfall 5 (over-specification) by defining prototype constraints from day one; Pitfall 4 (E2E format gaps) by building the extended format template early; Pitfall 1 (self-verification blindness) by building human gates into the first iteration

### Phase 2: Orchestrator Skill + Full Stage Pipeline

**Rationale:** With the inner loop proven, wrap it in the full orchestrator skill with all stage adapters (COMMIT, DEPLOY, DOCUMENT, RETROSPECT, PROMOTE). This is where GSD/PDCA integration happens. Hook-based enforcement ensures stage gates fire deterministically.
**Delivers:** Complete lifecycle orchestration (all 9 stages). Hook configuration (SessionStart for state re-injection, PreToolUse for stage gates, SubagentStop for result collection). Session resumption with spec-aware state display. Execution mode definitions (full/feature/hotfix/release).
**Uses:** Claude Code Hooks, Node.js CLI (gsd-tools.cjs), subagent delegation
**Implements:** Orchestrator-executor separation, progressive disclosure, markdown-as-memory

### Phase 3: Multi-Feature + Project Variants

**Rationale:** Phases 1-2 handle single-feature web projects. Real projects have multiple features with dependencies, different project types (API-only, CLI, mobile, desktop), and scope detection needs. The 5-layer chain is theoretically universal but needs concrete templates per project type.
**Delivers:** Multi-feature spec management with dependency declaration (depends_on field). Prototype template variants (API: request/response simulator, CLI: terminal session simulator, mobile: responsive wireframe). Scope detection (skip prototype for hotfix/config modes). Feature-level parallel execution.
**Addresses:** Feature dependencies, non-web project adaptation, cross-feature connection mapping

### Phase 4: Decision Trail + Polish

**Rationale:** After core functionality is solid, layer on decision tracking (lightweight decision history, settings changelog, code-to-decision linking) and add the differentiator features (state machine visualization, progress dashboards, data flow annotations). These enhance the system but do not change its fundamental value proposition.
**Delivers:** Lightweight decision history format, settings change auto-tracking, state machine visualization in prototypes, layer-level progress dashboard, data flow annotations, reconcile command (reconstruct state from filesystem), compaction resilience hardening
**Addresses:** All "should have" differentiator features, decision trail, resilience edge cases

### Phase Ordering Rationale

- **Inner loop first, outer loop second.** PLAN-DO-TEST is used on every feature. DEPLOY-DOCUMENT-RETROSPECT is used per release. Validate the frequent path first.
- **Spec+prototype cannot be deferred.** They are not an enhancement -- they are the mechanism that makes the pipeline solve the actual problem. A pipeline without them is just stage routing, which GSD already does.
- **Feature grouping follows architecture boundaries.** Phase 1 = inner loop artifacts. Phase 2 = full lifecycle orchestration. Phase 3 = horizontal scaling (more project types). Phase 4 = vertical depth (richer features within existing structure).
- **Pitfall avoidance drives ordering.** Building prototype constraints (Phase 1) before orchestration (Phase 2) prevents the prototype from becoming a mini-app. Building verification gates (Phase 1) before multi-feature support (Phase 3) prevents verification theater from scaling.

### Research Flags

Phases likely needing deeper research during planning:
- **Phase 1:** Spec format needs resolution. FEATURES.md recommends YAML (.e2e.yaml). ARCHITECTURE.md uses Markdown with frontmatter (.spec.md). Both have merits. Recommend resolving before implementation -- likely YAML for spec data with .md wrapper for readability.
- **Phase 3:** Non-web prototype adaptation has sparse examples. API-only, CLI, and mobile prototypes need concrete templates. The 5-layer chain is universal but concrete manifestation varies significantly.

Phases with standard patterns (skip research-phase):
- **Phase 2:** GSD codebase provides proven orchestration patterns. Hook system, subagent spawning, state management are well-documented with working examples.
- **Phase 4:** Differentiator features and decision tracking are additive. Standard web development and append-only log patterns apply.

## Confidence Assessment

| Area | Confidence | Notes |
|------|------------|-------|
| Stack | HIGH | All technologies verified against official Claude Code docs (2026-03-22). GSD codebase provides proven reference. Agent Skills 1.0 is cross-platform standard adopted by Anthropic, Microsoft, OpenAI, GitHub, Cursor. |
| Features | HIGH | Core problem (100% feature failure rate) grounded in user experience. Feature design opinionated but supported by spec-driven development research (Addy Osmani, GitHub Blog, Red Hat, Thoughtworks). Anti-features well-reasoned with concrete alternatives. |
| Architecture | HIGH | Based on direct analysis of existing skill implementations + proven GSD patterns. Dual-artifact approach is novel but each component (spec, prototype, verification gate) is well-understood individually. |
| Pitfalls | HIGH | Grounded in user experience, verified against community patterns (verification debt, spec drift), and academic/industry literature on AI-generated code quality. Honest about what the approach cannot solve. |

**Overall confidence:** HIGH

### Gaps to Address

- **Spec format resolution:** FEATURES.md recommends YAML (.e2e.yaml) while ARCHITECTURE.md uses Markdown with frontmatter (.spec.md). Resolve during Phase 1 planning. Recommendation: YAML for spec data, .md wrapper for human readability.
- **Prototype complexity calibration:** Research says 3-8 spec steps per feature and < 15 minutes to generate. These are untested heuristics. Validate during Phase 1 with real features.
- **Realistic failure rate improvement:** PITFALLS.md honestly states the approach may improve 100% failure rate to 20-30%, not 0%. Gap is in behavioral verification (user must test) and cross-feature interactions (not covered by single-feature specs). Set expectations accordingly.
- **E2E spec extended format:** Research suggests extending from 5 to 7 layers (adding state change + side effects) plus preconditions and edge cases. Start with 5 layers in Phase 1, evaluate extensions based on real usage.
- **SKILL.md size budget:** With multiple stage adapters, the SKILL.md could exceed 500 lines. The progressive disclosure split (which content is inlined vs referenced) must be decided during Phase 1 planning.

## Sources

### Primary (HIGH confidence)
- [Claude Code Skills Documentation](https://code.claude.com/docs/en/skills) -- SKILL.md format, frontmatter, progressive disclosure, invocation control
- [Claude Code Hooks Guide](https://code.claude.com/docs/en/hooks-guide) -- hook events, matchers, lifecycle automation
- [Claude Code Subagents Documentation](https://code.claude.com/docs/en/sub-agents) -- subagent configuration, tool restrictions, isolation
- [Agent Skills Open Standard](https://agentskills.io/home) -- cross-platform specification (Anthropic, Microsoft, OpenAI, GitHub, Cursor)
- [Anthropic: Demystifying Evals for AI Agents](https://www.anthropic.com/engineering/demystifying-evals-for-ai-agents) -- agent evaluation approaches
- GSD codebase (~/.claude/get-shit-done/) -- proven orchestration patterns, directly examined
- Existing skills (~/.claude/skills/) -- dev-lifecycle, adr, gsd-retrospective, work-log, directly examined

### Secondary (MEDIUM confidence)
- [Addy Osmani: How to Write a Good Spec for AI Agents](https://addyosmani.com/blog/good-spec/) -- spec-writing best practices
- [CodeRabbit: 2026 Year of AI Quality](https://www.coderabbit.ai/blog/2025-was-the-year-of-ai-speed-2026-will-be-the-year-of-ai-quality) -- AI code 1.7x more issues
- [Verification debt: hidden cost of AI-generated code](https://fazy.medium.com/agentic-coding-ais-adolescence-b0d13452f981) -- verification debt concept
- [Why Spec-Driven Development Fails](https://dev.to/casamia918/why-spec-driven-development-fails-and-what-we-can-learn-from-it-2pec) -- over-specification costs (33 min / 2,577 lines for 689 lines code)
- [Martin Fowler: Understanding SDD tools](https://martinfowler.com/articles/exploring-gen-ai/sdd-3-tools.html) -- SDD tooling analysis
- [GitHub Blog: Spec-driven development with AI](https://github.blog/ai-and-ml/generative-ai/spec-driven-development-with-ai-get-started-with-a-new-open-source-toolkit/) -- SDD patterns
- [Red Hat: How SDD improves AI coding quality](https://developers.redhat.com/articles/2025/10/22/how-spec-driven-development-improves-ai-coding-quality) -- SDD benefits/limitations
- [TestQuality: Gherkin BDD Guide](https://testquality.com/gherkin-bdd-cucumber-guide-to-behavior-driven-development/) -- Gherkin pros/cons

### Tertiary (LOW confidence)
- [taskmd: Task Management for AI Era](https://medium.com/@driangle/taskmd-task-management-for-the-ai-era-92d8b476e24e) -- YAML+Markdown format (single source)

---
*Research completed: 2026-03-22*
*Ready for roadmap: yes*
