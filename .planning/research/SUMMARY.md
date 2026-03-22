# Project Research Summary

**Project:** dev-lifecycle (Claude Code Skill-Based Development Lifecycle Orchestrator)
**Domain:** AI-assisted development workflow orchestration
**Researched:** 2026-03-22
**Confidence:** HIGH

## Executive Summary

This project builds a Claude Code skill that orchestrates the full development lifecycle (PLAN, DO, TEST, COMMIT, DEPLOY, DEPLOY TEST, DOCUMENT, RETROSPECT, PROMOTE) by wrapping existing GSD and PDCA skills without modifying them. The "stack" is not traditional software -- it is structured markdown instructions (SKILL.md), a Node.js CLI utility (gsd-tools.cjs), filesystem-based state management, and subagent delegation. Everything runs inside Claude Code with zero external infrastructure. The proven GSD orchestration pattern (orchestrator-executor separation, progressive disclosure, markdown-as-memory) provides a solid foundation.

The recommended approach is an adapter-based architecture where the orchestrator delegates to existing skills (GSD, PDCA, ADR, retrospective, work-log) through logical adapters defined in instruction blocks. A manifest-based artifact flow system ensures each stage's output feeds the next stage's input -- solving the core problem that AI declares features "done" when they are only partially wired. The orchestrator maintains its own state in `.lifecycle/` (separate from GSD's `.planning/`) and supports multiple execution modes (full, feature, hotfix, release) to avoid process overhead on small tasks.

The three biggest risks are: (1) the "done but not done" illusion where Claude declares completion on non-functional code -- mitigated by end-to-end feature specs and mandatory wiring verification gates; (2) memory amnesia across sessions and after context compaction -- mitigated by file-based artifacts as external memory and a living state document; and (3) role overlap confusion between GSD, PDCA, and dev-lifecycle -- mitigated by making dev-lifecycle the single user-facing entry point with internal delegation invisible to the user. A confirmed Claude Code bug (context loss after compaction, GitHub issue #13919) makes compaction-resistant design non-negotiable.

## Key Findings

### Recommended Stack

The stack is entirely Claude Code native: SKILL.md files (Agent Skills 1.0 cross-platform standard), Claude Code hooks for deterministic lifecycle enforcement, subagents for parallel execution with fresh context windows, and a single-file Node.js CLI (gsd-tools.cjs) for state management. No npm packages, no frameworks, no servers.

**Core technologies:**
- **SKILL.md (Agent Skills 1.0):** Primary orchestration mechanism -- cross-platform standard, YAML frontmatter + markdown body, auto-invocation via description matching
- **Claude Code Hooks:** Deterministic lifecycle automation -- fires every time (not probabilistic), essential for stage gates and context re-injection after compaction
- **Claude Code Subagents:** Parallel task execution -- fresh 200k context per agent, model routing (haiku/sonnet/opus), up to 10 simultaneous
- **Node.js CLI (gsd-tools.cjs):** State management -- single-file CJS, zero dependencies, handles state load/save, phase operations, frontmatter CRUD
- **Filesystem State (JSON + Markdown):** Persistence across sessions and compaction -- files survive everything; in-memory state and env vars do not

**Critical constraints:**
- SKILL.md must stay under 500 lines (progressive disclosure mandatory)
- Skills share 2% of context window for descriptions -- bloat crowds out other skills
- No modification of GSD/PDCA core -- orchestration layer only

### Expected Features

**Must have (table stakes -- P1):**
- Project-agnostic 9-stage pipeline (remove muse hardcoding)
- Stage artifact chaining (each stage's output feeds the next)
- End-to-end feature specification format (screen > connection > processing > response > error)
- Session context restoration via living state document
- Settings change auto-tracking (append-only log with rationale)
- Preflight check at PLAN stage (retrospectives + ADRs + decision history)
- Multiple execution modes (full, feature, hotfix, release)
- Phase transition detection (signal-based, not keyword-based)

**Should have (differentiators -- P2):**
- User interaction prototype (clickable HTML+JS mockups at PLAN stage)
- Cross-stage verification chain (TEST validates against PLAN spec, not just "does it work")
- Lightweight decision history (between ADR and nothing)
- Code-to-decision linking (extended WHY+SEE comments)
- Preflight check automation (automated scanning vs manual checklist)

**Defer (v2+):**
- Multi-project orchestration
- Custom stage injection
- Metric aggregation dashboard
- Template marketplace

**Anti-features (explicitly avoid):**
- Pixel-perfect UI mockups (validate flow, not visual design)
- Automatic stage progression (human checkpoints ARE the value)
- External database for memory (files are the memory)
- AI-generated E2E test code (spec is the test contract)

### Architecture Approach

The architecture follows an orchestrator-adapter pattern with four core components: Stage Router (intent detection + stage mapping), State Engine (JSON-based lifecycle tracker in `.lifecycle/state.json`), Artifact Bus (manifest-based output-to-input connector in `.lifecycle/manifest.json`), and Skill Adapters (translation layers for each wrapped skill). The orchestrator maintains its own state separate from GSD's `.planning/` to avoid coupling. All state is file-based for session resilience.

**Major components:**
1. **Stage Router** -- detects user intent, maps to correct stage, enforces ordering rules per execution mode
2. **State Engine** -- tracks current stage in `.lifecycle/state.json`, records transitions in `history/`, enables session resumption
3. **Artifact Bus** -- manifest registry mapping stages to artifact paths; validates artifact existence before allowing stage transitions
4. **Skill Adapters** -- one per wrapped skill (GSD, PDCA, ADR, retro, work-log); translates orchestrator intent into skill invocations without modifying the skill
5. **Prototype Engine** -- generates clickable HTML+JS mockups during PLAN for user validation (P2 feature)

### Critical Pitfalls

1. **"Done but not done" illusion** -- Claude declares completion on partially-wired code. Mitigate with end-to-end feature specs and mandatory wiring verification gates before COMMIT. This is the single most important pitfall (user reports 100% failure rate).
2. **Memory amnesia (sessions + compaction)** -- Context lost between sessions and silently dropped after compaction (confirmed bug #13919). Mitigate with file-based artifacts as memory, living state document, and explicit re-read instructions in SKILL.md.
3. **Role overlap confusion (GSD/PDCA/lifecycle)** -- Three systems with overlapping planning/execution territory. Mitigate by making dev-lifecycle the sole user entry point with a responsibility matrix defining which system does what within each stage.
4. **Artifact fragmentation** -- Stage outputs not actually connected despite linear pipeline appearance. Mitigate with manifest-based artifact flow and mandatory file reads at stage entry.
5. **Settings archaeology** -- Configuration changes without recorded rationale make future modifications impossible. Mitigate with lightweight append-only decision log, automated detection in DO stage.

## Implications for Roadmap

Based on research, suggested phase structure:

### Phase 1: Foundation and State Engine
**Rationale:** Everything depends on state management and artifact tracking. The state engine and manifest schema must exist before any stage adapter can function. This also establishes the `.lifecycle/` directory structure and execution mode definitions.
**Delivers:** state.json schema, manifest.json schema, execution mode definitions (full/feature/hotfix/release), session resumption logic, responsibility matrix (GSD vs PDCA vs lifecycle boundaries)
**Addresses:** Session context restoration, phase transition detection, role overlap confusion
**Avoids:** Pitfall 3 (role confusion) by defining boundaries first; Pitfall 6 (artifact fragmentation) by establishing manifest schema

### Phase 2: Core Pipeline -- Inner Loop (Stages 1-4)
**Rationale:** Stages 1-4 (PLAN, DO, TEST, COMMIT) cover 80% of daily development work. Building these first enables real-project validation before investing in deploy/document stages. The inner loop is where the three core problems manifest most acutely.
**Delivers:** PLAN adapter (wraps GSD plan + PDCA gap analysis + preflight check), DO adapter (wraps PDCA do + ADR auto-detection + settings tracking), TEST adapter (wraps GSD verify + wiring verification gate), COMMIT adapter (git conventions + artifact validation)
**Addresses:** Project-agnostic pipeline, stage artifact chaining, end-to-end feature spec format, settings change tracking, preflight check (manual)
**Avoids:** Pitfall 1 ("done but not done") by building wiring verification into TEST; Pitfall 2 (amnesia) by enforcing artifact-as-memory; Pitfall 6 (fragmentation) by mandating artifact reads at stage entry

### Phase 3: User Interaction Prototype
**Rationale:** The clickable HTML+JS mockup system depends on a stable PLAN adapter (Phase 2) but is the key differentiator that separates this from raw GSD usage. It solves both the specification gap (user cannot express UI interactions) and the "done but not done" problem (agreement before implementation). Building it as a separate phase allows Phase 2 to be validated on a real project first.
**Delivers:** Prototype generator (HTML+JS mockup creation), e2e spec generator (feature spec template), integration into PLAN adapter, screenshot-driven verification flow
**Addresses:** User interaction prototype, cross-stage verification chain (partial)
**Avoids:** Pitfall 4 (specification gap) by providing clickable validation before implementation

### Phase 4: Decision Trail
**Rationale:** With the inner loop working, the decision tracking mechanisms can be layered on. Settings tracking was started in Phase 2 as part of the DO adapter; this phase formalizes it with the lightweight decision log and code-to-decision linking.
**Delivers:** Lightweight decision history format, settings changelog auto-detection, code-to-decision linking (extended WHY+SEE), preflight check automation (scanning all decision sources)
**Addresses:** Decision history, code-to-decision linking, preflight check automation
**Avoids:** Pitfall 5 (settings archaeology) with formal tracking; Pitfall 2 (amnesia) with comprehensive decision trail

### Phase 5: Outer Loop (Stages 5-9)
**Rationale:** Deploy, deploy test, document, retrospect, and promote stages are less frequent than the inner loop. Building them after the inner loop is validated ensures the foundation is solid. Deploy templates must be project-agnostic (the hardest part of removing muse-specific code).
**Delivers:** DEPLOY adapter (generic, configurable templates), DEPLOY TEST adapter (smoke test framework), DOCUMENT adapter (canvas-design + CLAUDE.md update), RETROSPECT adapter (gsd-retrospective + ADR gap check + work-log), PROMOTE adapter (optional, manual trigger)
**Addresses:** Remaining table stakes features, retrospective integration
**Avoids:** Pitfall 7 (overhead) by having execution modes ready from Phase 1

### Phase 6: Polish and Resilience
**Rationale:** After all stages work, add robustness features: reconcile command (reconstruct state from filesystem when out of sync), scope detection (auto-suggest execution mode based on task size), skip-mode validation, and compaction resilience hardening.
**Delivers:** Reconcile command, automatic scope detection, verbose/compact output modes, history visualization, compaction recovery protocol
**Addresses:** Orchestrator overhead reduction, edge case handling
**Avoids:** Pitfall 7 (overhead paralysis) with scope detection; Pitfall 2 (compaction amnesia) with hardened recovery

### Phase Ordering Rationale

- **Foundation before features:** State engine and manifest schema are prerequisites for every stage adapter. Building features without state management creates the exact artifact fragmentation problem the orchestrator exists to solve.
- **Inner loop before outer loop:** Stages 1-4 are used daily; stages 5-9 are used per-release. Validate the frequent path first, then extend.
- **Prototype after inner loop:** The prototype system enhances PLAN but is not blocking for the core pipeline. Shipping a working inner loop without prototypes is useful; shipping prototypes without a working pipeline is not.
- **Decision trail after prototype:** Decision tracking enhances the pipeline but is a separate concern. Each phase layers on top without requiring changes to previous phases.
- **Polish last:** Resilience and convenience features only matter once the core system works.

### Research Flags

Phases likely needing deeper research during planning:
- **Phase 3 (User Interaction Prototype):** Novel feature with no established pattern in Claude Code skills. HTML+JS mockup generation from feature specs needs design exploration. The boundary between "flow validation" and "visual design" needs explicit specification.
- **Phase 5 (Outer Loop -- DEPLOY adapter):** Project-agnostic deploy templates are the hardest generalization problem. Need to research how to handle the diversity of deploy targets (rsync, Docker, cloud platforms, mobile stores) without hardcoding.

Phases with standard patterns (skip research-phase):
- **Phase 1 (Foundation):** JSON state management and file-based state machines are well-documented. GSD's existing STATE.md pattern is the direct reference.
- **Phase 2 (Core Pipeline):** Adapter pattern for skill wrapping is proven by GSD. Stage implementation follows established orchestrator-executor pattern.
- **Phase 4 (Decision Trail):** Append-only logs and changelog formats are straightforward. ADR pattern already exists.
- **Phase 6 (Polish):** Reconcile commands and scope detection are standard patterns.

## Confidence Assessment

| Area | Confidence | Notes |
|------|------------|-------|
| Stack | HIGH | All technologies verified against official Claude Code docs (2026-03-22). GSD codebase provides proven reference implementation. Agent Skills 1.0 is a cross-platform standard. |
| Features | HIGH | Features derived from user's direct experience with three specific problems. Competitor analysis against existing GSD/PDCA clarifies what the orchestrator adds vs duplicates. Anti-features well-reasoned. |
| Architecture | HIGH | Architecture based on direct analysis of 6 existing skill implementations and GSD internals. Adapter pattern and manifest-based flow are proven patterns. `.lifecycle/` separation from `.planning/` is a sound boundary decision. |
| Pitfalls | HIGH | Pitfalls grounded in user's real experience (100% failure rate on "done but not done") and confirmed community bugs (compaction amnesia #13919). Prevention strategies are concrete and actionable. |

**Overall confidence:** HIGH

### Gaps to Address

- **Compaction recovery mechanism:** The compaction bug (#13919) is confirmed but unresolved upstream. The SKILL.md re-read strategy is a workaround, not a fix. Monitor the issue for upstream resolution. During planning, design the skill to degrade gracefully when compaction occurs.
- **Prototype generation quality:** No reference implementation exists for generating clickable HTML+JS mockups from feature specs in a Claude Code skill context. Phase 3 planning should include a spike/proof-of-concept before committing to a full implementation approach.
- **Deploy template generalization:** The current SKILL.md has muse-specific deploy commands (rsync, flutter, systemd). The generic replacement needs to handle arbitrary deploy targets. During Phase 5 planning, define a template format that is extensible without being overengineered.
- **SKILL.md size budget:** With 9 stage adapters, the SKILL.md could easily exceed 500 lines. The progressive disclosure split (which files get referenced vs inlined) needs to be decided during Phase 1 planning.
- **Multi-feature parallel state:** The architecture mentions per-feature state tracking for complex projects but does not specify the schema. Defer to Phase 6 unless a real project demands it earlier.

## Sources

### Primary (HIGH confidence)
- [Claude Code Skills Documentation](https://code.claude.com/docs/en/skills) -- SKILL.md format, frontmatter, progressive disclosure, invocation control
- [Claude Code Hooks Guide](https://code.claude.com/docs/en/hooks-guide) -- hook events, matchers, input/output
- [Claude Code Subagents Documentation](https://code.claude.com/docs/en/sub-agents) -- subagent configuration, tool restrictions, memory
- [Agent Skills Open Standard](https://agentskills.io/home) -- cross-platform specification
- GSD codebase (`~/.claude/get-shit-done/`) -- proven orchestration patterns, CLI tool, workflows
- Existing skill implementations (`~/.claude/skills/`) -- dev-lifecycle, adr, retrospective, work-log, canvas-design

### Secondary (MEDIUM confidence)
- [IEEE Spectrum: AI Coding Degrades](https://spectrum.ieee.org/ai-coding-degrades) -- AI models optimized for confidence over correctness
- [Addy Osmani: AI Coding Workflow 2026](https://addyosmani.com/blog/ai-coding-workflow/) -- current state of AI-assisted development
- [CodeRabbit AI vs Human Code Report](https://www.coderabbit.ai/blog/state-of-ai-vs-human-code-generation-report) -- AI code creates 1.7x more issues
- [GitHub Issue #13919](https://github.com/anthropics/claude-code/issues/13919) -- confirmed compaction bug with community reproduction

### Tertiary (LOW confidence)
- [OneContext Persistent Context Layer](https://supergok.com/onecontext-persistent-context-layer-ai-coding-agents/) -- emerging solutions for AI agent memory (early-stage, unproven)

---
*Research completed: 2026-03-22*
*Ready for roadmap: yes*
