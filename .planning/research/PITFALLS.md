# Pitfalls Research

**Domain:** AI Development Lifecycle Orchestrator (Claude Code Skill)
**Researched:** 2026-03-22
**Confidence:** HIGH (based on user's direct experience + verified community reports + existing codebase analysis)

## Critical Pitfalls

### Pitfall 1: The "Done But Not Done" Illusion -- AI Declares Completion on Non-Functional Code

**What goes wrong:**
Claude says "done" but the feature does not work end-to-end. Buttons exist but have no event handlers. API routes exist but are not called from the frontend. Functions exist but return hardcoded data. The user has experienced a 100% failure rate on this -- every single feature Claude declares "complete" fails at some point in the chain: screen -> connection -> processing -> response -> error handling.

**Why it happens:**
Three reinforcing causes:
1. **Existence bias.** AI models equate "file exists with relevant code" with "feature works." A component that renders JSX is "done" even if onClick is `() => {}`.
2. **Agentic loop optimization.** Models are tuned to keep making progress and minimize giving up ([IEEE Spectrum](https://spectrum.ieee.org/ai-coding-degrades)). This creates confident completion signals on half-wired code.
3. **No end-to-end verification in the pipeline.** The current SKILL.md Stage 3 (TEST) lists verification items but has no mechanism to enforce them before Claude declares completion. Verification is a suggestion, not a gate.

**How to avoid:**
1. **End-to-end feature specs at PLAN time.** Each feature gets a chain spec: `UI element -> event handler -> API call -> backend logic -> data mutation -> response -> UI update -> error case`. Missing any link = spec incomplete.
2. **Wiring verification before "done."** The existing GSD verification-patterns.md already has excellent programmatic checks (stub detection, wiring checks, component->API->DB chain verification). These must be mandatory at the TEST stage, not optional.
3. **User Interaction Prototype.** Clickable HTML+JS mockups generated at PLAN stage force agreement on what "done" looks like before implementation starts. This is already in PROJECT.md as a requirement -- it must be the first thing built.
4. **"Looks Done" checklist as a gate.** Convert the verification patterns into a mandatory checklist that blocks Stage 4 (COMMIT) until passed.

**Warning signs:**
- Claude says "I've implemented X" without showing the full chain (UI -> API -> DB)
- Test stage passes in < 30 seconds (no real verification happened)
- Feature described in terms of files created rather than user flows working
- No error handling mentioned in the implementation

**Phase to address:**
Phase 1 (Core Pipeline). The end-to-end spec format and wiring verification gate must be built into the PLAN and TEST stages from day one. Without this, every subsequent phase produces non-functional features.

---

### Pitfall 2: Memory Amnesia -- Context Lost Between Sessions and After Compaction

**What goes wrong:**
Two failure modes:
1. **Cross-session amnesia.** New Claude session has zero knowledge of previous decisions, settings changes, direction pivots, or the "why" behind current state. User must re-explain everything.
2. **Mid-session compaction amnesia.** After auto-compaction (~55K tokens in VS Code), Claude completely loses awareness of active skills, procedures, and methodologies. This is a [confirmed bug](https://github.com/anthropics/claude-code/issues/13919) with community reports of tasks ballooning from 1 hour to 5-6 hours. Skills procedures are silently dropped from context. Claude does not re-read skill files. CLAUDE.md post-compaction instructions are also ignored.

**Why it happens:**
1. Claude does not store preferences, personal facts, or project context across sessions. Any long-term memory must be implemented externally.
2. Compaction summarizes conversation history but loses procedural knowledge (skills). The compaction algorithm treats skill content as regular conversation, not as persistent instructions.
3. Decision history typically lives only in human memory or scattered across chat logs that cannot be recovered.

**How to avoid:**
1. **Stage artifacts ARE the memory.** Every stage produces structured output files that serve as external memory. The key insight from PROJECT.md is correct: "Stage output is the next stage's input." This must be enforced, not suggested.
2. **Living state document.** A single file (like STATE.md or CURRENT.md) that captures the current project state, active decisions, and recent direction changes. Read at session start = instant context restoration. This is already listed in PROJECT.md requirements.
3. **Settings changelog with WHY.** Every settings/configuration change gets a timestamped entry with the motivation. Not ADR-heavy, just: `[date] Changed X from A to B because [reason]`. This prevents the "why is it set this way?" problem.
4. **Compaction-resistant skill design.** Keep skill files short and focused. Put critical procedures in CLAUDE.md (which is re-read more reliably than skills). Use file-based state that Claude can re-read after compaction rather than relying on conversation memory.
5. **Session bootstrap protocol.** Skill should include explicit "on session start, read these files" instructions that reconstruct full context from artifacts.

**Warning signs:**
- Claude asks about something that was decided 2 sessions ago
- Settings are changed without any record of the previous value or reason
- Claude starts contradicting previous architectural decisions
- After compaction, Claude stops following skill procedures silently (no error, just stops)

**Phase to address:**
Phase 1 (Core Pipeline) for artifact-as-memory design. Phase 2 for the settings changelog and living state document. The compaction resilience should be designed from the start but will need iteration based on real usage.

---

### Pitfall 3: Role Overlap Confusion -- GSD vs PDCA vs Dev-Lifecycle Boundary Ambiguity

**What goes wrong:**
Three systems claim overlapping territory:
- GSD has `plan-phase`, `execute-phase`, `verify-work`
- PDCA has `plan`, `design`, `do`, `check`, `act`, `report`
- Dev-lifecycle has 9 stages including PLAN, DO, TEST

When the user says "plan this feature," it is unclear whether to use GSD plan-phase, PDCA plan, or dev-lifecycle Stage 1. The current SKILL.md maps Stage 1 to "GSD + PDCA" but does not specify which does what within that stage. This creates confusion about entry points, duplicate work, and gaps where each system assumes the other handles something.

**Why it happens:**
1. **Organic growth.** Each system was built independently for different purposes. Overlap was not designed, it accumulated.
2. **Orchestration without delegation clarity.** The dev-lifecycle says "use GSD for X and PDCA for Y" but the boundaries are described in terms of tool names, not responsibilities. "GSD does planning" and "PDCA does planning" are both true, which means neither is useful as a disambiguation.
3. **No single entry point.** Users must know which command to invoke (`/gsd:plan-phase` vs `/pdca plan` vs `/dev-lifecycle Stage 1`), and getting it wrong means either duplicate work or missing steps.

**How to avoid:**
1. **Dev-lifecycle is the single entry point.** Users only interact with dev-lifecycle commands. Dev-lifecycle internally delegates to GSD/PDCA as implementation details. The user never needs to know which sub-system handles what.
2. **Responsibility matrix, not tool matrix.** Define boundaries by what gets done, not which tool does it:
   - PLAN stage responsibility: "Produce a feature spec with end-to-end chain, user interaction prototype, and pre-flight check"
   - GSD's role within PLAN: "milestone structure, phase ordering, verification criteria"
   - PDCA's role within PLAN: "gap analysis against previous cycle, continuous improvement items"
3. **Constraint: no modification of GSD/PDCA.** This is already in PROJECT.md Out of Scope. Dev-lifecycle orchestrates on top; it does not change the underlying systems.

**Warning signs:**
- User hesitates about which command to use
- Same information is requested/produced by two different stages
- A responsibility falls through the cracks because each system assumed the other handled it
- Skill description triggers on too many overlapping keywords

**Phase to address:**
Phase 1 (Core Pipeline). The responsibility matrix must be defined before any stage implementation. Without clear boundaries, every subsequent feature amplifies the confusion.

---

### Pitfall 4: Specification Gap -- User Cannot Express Detailed UI Interactions to AI

**What goes wrong:**
The user knows exactly what they want a button to do, how a form should validate, what happens on hover, what the error state looks like. But there is no structured way to communicate this to Claude. Natural language descriptions are ambiguous ("make a settings page") and Claude fills gaps with assumptions that are usually wrong. The result is features that look plausible but do not match what the user actually needed.

**Why it happens:**
1. **Text is a lossy medium for interaction design.** Describing "click this, then that slides out, then type here, then this validates inline" requires dozens of sentences that are hard to write and easy to misinterpret.
2. **AI optimizes for plausible, not correct.** When a spec is vague, Claude generates something that looks reasonable but reflects AI training data patterns, not the user's mental model.
3. **No feedback loop before implementation.** By the time the user sees the result, the code is already written. Fixing it means rework, not refinement.

**How to avoid:**
1. **User Interaction Prototype (HTML+JS mockup).** This is already identified in PROJECT.md as a key solution. Generate clickable prototypes at PLAN stage. User clicks through, confirms or corrects. Only then does implementation begin.
2. **Interaction chain format.** Structured specs that enumerate: trigger -> action -> visual change -> data change -> next state -> error case. Each interaction is a row, not a paragraph.
3. **Screenshot-driven verification.** At TEST stage, capture screenshots and present to user for visual confirmation. Claude cannot verify "looks right" but the user can.

**Warning signs:**
- Feature spec describes components but not interactions
- User says "that's not what I meant" after seeing the implementation
- Multiple rework cycles on the same feature
- Settings or configuration UIs that are technically functional but have wrong defaults, wrong grouping, or missing options

**Phase to address:**
Phase 2 (User Interaction Prototype). This must be implemented early because it solves both the specification gap AND the "done but not done" problem (users verify before and after implementation).

---

### Pitfall 5: Settings Archaeology -- Configuration Changes Without Recorded Rationale

**What goes wrong:**
Settings and configuration values are changed during development. Later, nobody remembers why a value was set to what it is. Changing it risks breaking something that was intentionally configured that way. The user described this as making "future modifications impossible" without re-explaining everything from scratch.

**Why it happens:**
1. **Settings changes feel minor.** Developers (and AI) treat config changes as trivial -- just change the value. But the WHY behind the value is the actual knowledge.
2. **No natural recording point.** Unlike code changes (which have commits and diffs), settings changes often happen in conversation ("change the timeout to 30s") without any artifact recording the decision.
3. **ADR is too heavy.** Architecture Decision Records are great for big decisions but too ceremonial for "changed API_TIMEOUT from 10s to 30s because users on slow networks were getting cut off."

**How to avoid:**
1. **Lightweight decision log.** Not ADR, not a full document. A simple append-only log:
   ```
   [2026-03-22] API_TIMEOUT: 10s -> 30s
   Reason: Users on slow networks getting timeout errors during image upload
   Context: Reported in Phase 3 testing
   ```
2. **Automated detection in DO stage.** When Claude modifies any configuration file (.env, config.*, settings.*, constants.*), the orchestrator automatically prompts for a reason and records it.
3. **Code comments with cross-references.** The SKILL.md already mentions "WHY+SEE comments in code linked to ADR." Extend this to settings: every non-obvious config value gets a comment with the decision log entry reference.

**Warning signs:**
- Config values with no comments explaining why
- User asks "why is this set to X?" and nobody knows
- Settings changes in git diff with no corresponding documentation
- Fear of changing configuration because "it might break something"

**Phase to address:**
Phase 3 (Decision Trail). The settings changelog mechanism should be built after the core pipeline and interaction prototype, but before the system is used on real projects. Otherwise, the first real project will accumulate undocumented settings that become impossible to maintain.

---

### Pitfall 6: Artifact Fragmentation -- Stage Outputs Not Actually Connected

**What goes wrong:**
The 9-stage pipeline looks linear on paper (PLAN -> DO -> TEST -> COMMIT -> ...) but in practice, each stage produces artifacts that the next stage does not actually read. PLAN produces a spec. DO ignores the spec and implements what Claude thinks is right. TEST does not check against the PLAN spec. The pipeline is a sequence of independent steps, not a connected chain.

**Why it happens:**
1. **No enforcement mechanism.** The SKILL.md says "each stage's output is the next stage's input" but there is no mechanism that forces Stage 2 to read Stage 1's output. Claude may or may not read it depending on context window state.
2. **Artifact format mismatch.** PLAN output might be markdown prose. DO needs actionable specs. TEST needs verification criteria. If formats do not align, the connection breaks.
3. **Compaction breaks the chain.** Even if Stage 1 output was in context, compaction may remove it before Stage 2 completes. The artifact exists on disk but Claude does not know to re-read it.

**How to avoid:**
1. **Mandatory file reads at stage entry.** Each stage begins with explicit `Read(previous_stage_artifact)` instructions. Not "check if it exists" -- always read it.
2. **Structured artifact format.** Each stage produces output in a format the next stage can parse. PLAN produces a checklist. DO checks off items. TEST verifies each item. Same data structure flowing through the pipeline.
3. **Cross-reference validation.** TEST stage explicitly compares PLAN spec items against DO deliverables. Any PLAN item without a corresponding DO deliverable is flagged as incomplete.

**Warning signs:**
- Stage 2 output does not reference Stage 1 output
- Features implemented that were not in the plan
- Planned features missing from implementation without explanation
- TEST stage verifies things not mentioned in PLAN

**Phase to address:**
Phase 1 (Core Pipeline). The artifact connection mechanism is the core value proposition of the orchestrator. If artifacts are not connected, the orchestrator adds overhead without benefit.

---

### Pitfall 7: Orchestrator Overhead Paralysis -- Too Much Process for Small Tasks

**What goes wrong:**
The 9-stage pipeline is appropriate for feature development and releases. But for a hotfix, typo fix, or small config change, running through PLAN -> DO -> TEST -> COMMIT -> DEPLOY -> DEPLOY TEST -> DOCUMENT -> RETROSPECT is absurd. Users abandon the orchestrator for small tasks, which means small changes accumulate without documentation or decision records.

**Why it happens:**
1. **One-size-fits-all pipeline.** The current SKILL.md has a "minimal execution mode" table but it is not automated. The user must manually decide which stages to skip.
2. **Process adoption friction.** If the process feels heavy for common cases, it gets abandoned entirely. Developers do not use processes that slow them down on small tasks, even if the process helps on big tasks.

**How to avoid:**
1. **Automatic scope detection.** The orchestrator should detect task size and suggest the appropriate stage subset:
   - Hotfix: DO -> TEST -> COMMIT -> DEPLOY -> DEPLOY TEST (skip PLAN, DOCUMENT, RETROSPECT)
   - Config change: DO -> COMMIT + settings changelog entry (skip everything else)
   - Feature: Full pipeline
   - Milestone: Full pipeline + PROMOTE
2. **Always-on minimum.** Some things should happen regardless of scope: settings changelog (if config touched), commit message quality, and artifact trail. These are not separate stages, they are embedded in every stage.

**Warning signs:**
- User bypasses the orchestrator for "small" tasks
- Accumulation of undocumented changes
- Resistance to starting the lifecycle for anything less than a major feature
- User manually invoking Stage 4 (COMMIT) without any prior stages

**Phase to address:**
Phase 2 or Phase 3. Build the full pipeline first (Phase 1), validate it works on feature-sized tasks, then add scope detection to make it lightweight for small tasks.

---

## Technical Debt Patterns

| Shortcut | Immediate Benefit | Long-term Cost | When Acceptable |
|----------|-------------------|----------------|-----------------|
| Hardcoding project paths in skill | Quick to get working | Cannot use on other projects; defeats "universal skill" goal | Never -- this is the exact problem being solved (muse hardcoding) |
| Skipping artifact format standardization | Faster to ship each stage independently | Stages cannot read each other's output; pipeline is disconnected | MVP only, with explicit plan to standardize in next phase |
| Relying on conversation context instead of files | Less file I/O, faster responses | Compaction destroys the context; cross-session amnesia | Never for decisions or settings; acceptable for ephemeral discussion |
| Minimal error handling in stage transitions | Cleaner happy-path code | Silent failures when a stage produces unexpected output | MVP only, but add error detection (not handling) immediately |
| Putting all 9 stages in one SKILL.md file | Single file to maintain | Exceeds practical skill file size; compaction more likely to drop skill content | Acceptable until file exceeds ~500 lines, then split |

## Integration Gotchas

| Integration | Common Mistake | Correct Approach |
|-------------|----------------|------------------|
| GSD plan-phase | Calling it directly when dev-lifecycle PLAN should orchestrate | Dev-lifecycle PLAN calls GSD internally; user only sees dev-lifecycle commands |
| PDCA cycle | Running full PDCA cycle independently of dev-lifecycle stages | PDCA check/act steps map to specific dev-lifecycle stages; orchestrator manages the mapping |
| ADR skill | Creating ADRs only when explicitly asked | Orchestrator detects tradeoff decisions in DO stage and auto-triggers ADR creation |
| gsd-retrospective | Retrospective as optional afterthought | Retrospective is mandatory Stage 8; even 5 lines of reflection has value (already in SKILL.md edge cases) |
| CLAUDE.md | Treating it as static documentation | CLAUDE.md is a living file updated by Stage 7 (DOCUMENT) with ADR links, retrospective lessons, and deployment info |

## Performance Traps

| Trap | Symptoms | Prevention | When It Breaks |
|------|----------|------------|----------------|
| Skill file too large | Compaction triggers sooner; skill content dropped from context | Keep SKILL.md under 500 lines; split into sub-files with explicit read instructions | Above ~800 lines in VS Code extension (55K token compaction threshold) |
| Too many MCP tools loaded | Context consumed by tool descriptions before any real work; "tens of thousands of tokens" per MCP | Load only the MCPs needed for current stage; unload between stages if possible | More than 3-4 MCPs active simultaneously |
| Reading all artifacts at session start | Consumes context budget on potentially irrelevant history | Read only STATE.md at start; lazy-load stage artifacts when entering that stage | More than 10 phases of artifact history |
| Stage transition detection via keyword matching | False triggers on casual conversation; missed triggers on non-standard phrasing | Use explicit commands (not keyword detection) for stage transitions; keyword detection as secondary | As skill vocabulary grows and overlaps with natural conversation |

## UX Pitfalls

| Pitfall | User Impact | Better Approach |
|---------|-------------|-----------------|
| Requiring user to know which sub-system to use | Cognitive overhead; wrong command -> wrong behavior | Single namespace: `/lifecycle:plan`, `/lifecycle:do`, etc. Internal delegation invisible to user |
| Stage completion without user confirmation | User discovers problems too late; rework | Every stage completion requires explicit user acknowledgment, even if it is just "looks good" |
| Verbose stage output | User skims or ignores; important items missed | Stage output has a 3-line summary at top, details below. Summary format: "Did X. Found Y issue(s). Next: Z." |
| Invisible orchestrator state | User does not know which stage they are in or what happened in previous stages | Persistent status indicator: "Stage 2/9: DO [Feature: user-auth] | Previous: PLAN complete | Next: TEST" |
| Silent skill procedure abandonment after compaction | User assumes skill is still active; quality degrades without warning | Explicit "skill reloaded" or "skill lost -- please re-invoke" notification mechanism |

## "Looks Done But Isn't" Checklist

- [ ] **Feature implementation:** Often missing event handler wiring -- verify every UI element has a handler that calls a real function (not `() => {}`)
- [ ] **API route:** Often missing input validation -- verify request body is validated before processing
- [ ] **API integration:** Often missing error response handling -- verify what happens when the API returns 400/500
- [ ] **Form submission:** Often missing loading/disabled state -- verify button is disabled during submission and re-enabled after
- [ ] **Data display:** Often missing empty state -- verify what shows when there is no data (not just blank screen)
- [ ] **Configuration:** Often missing the WHY comment -- verify every non-obvious config value has a reason documented
- [ ] **Stage artifact:** Often missing cross-reference to previous stage -- verify Stage N artifact references Stage N-1 output
- [ ] **Settings change:** Often missing changelog entry -- verify the settings log captures what changed and why
- [ ] **Auth/permissions:** Often missing authorization check -- verify the endpoint checks user permissions, not just authentication

## Recovery Strategies

| Pitfall | Recovery Cost | Recovery Steps |
|---------|---------------|----------------|
| Feature declared done but not working | MEDIUM | Run wiring verification checklist against the feature. Identify broken chain links. Fix each link. Re-verify end-to-end. |
| Memory loss between sessions | LOW (if artifacts exist) | Read STATE.md + recent stage artifacts. If no STATE.md, read most recent retrospective + CLAUDE.md for context reconstruction. |
| Memory loss after compaction | LOW | Re-read SKILL.md explicitly. Re-read current stage's input artifact. Announce "skill reloaded" to confirm procedures are active. |
| Role confusion (GSD/PDCA/lifecycle) | MEDIUM | Create responsibility matrix. Update SKILL.md to remove ambiguous trigger keywords. Establish single entry point. |
| Undocumented settings changes | HIGH | Audit all config files. For each non-default value, attempt to reconstruct rationale from git log, chat history, or user interview. Retroactively document. |
| Disconnected stage artifacts | MEDIUM | Standardize artifact format. Backfill cross-references. Add mandatory read instructions at each stage entry. |
| Process abandoned for small tasks | LOW | Add scope detection. Define minimum artifact trail for small tasks (just commit message + settings log if applicable). |

## Pitfall-to-Phase Mapping

| Pitfall | Prevention Phase | Verification |
|---------|------------------|--------------|
| "Done but not done" illusion | Phase 1: Core Pipeline | End-to-end spec format exists; wiring verification runs before COMMIT stage gate |
| Memory amnesia (cross-session) | Phase 1-2: Core Pipeline + Decision Trail | STATE.md reconstructs context; settings changelog captures changes |
| Memory amnesia (compaction) | Phase 1: Core Pipeline | SKILL.md includes explicit re-read instructions; artifacts on disk survive compaction |
| Role overlap confusion | Phase 1: Core Pipeline | User uses only dev-lifecycle commands; GSD/PDCA invoked internally |
| Specification gap | Phase 2: User Interaction Prototype | Clickable prototype exists for each feature; user confirms before DO stage |
| Settings archaeology | Phase 3: Decision Trail | Every config change has a changelog entry with reason |
| Artifact fragmentation | Phase 1: Core Pipeline | Each stage reads previous stage output (verified by cross-references in artifact) |
| Orchestrator overhead | Phase 3-4: After core validated | Scope detection suggests appropriate stage subset; small tasks have minimal overhead |

## Sources

- [IEEE Spectrum: AI Coding Degrades: Silent Failures Emerge](https://spectrum.ieee.org/ai-coding-degrades) -- on AI models optimized for confidence over correctness
- [VentureBeat: Why AI coding agents aren't production-ready](https://venturebeat.com/ai/why-ai-coding-agents-arent-production-ready-brittle-context-windows-broken) -- brittle context windows, broken refactors
- [GitHub Issue #13919: Claude Skills context completely lost after auto-compaction](https://github.com/anthropics/claude-code/issues/13919) -- confirmed bug with community reproduction, still unresolved as of Feb 2026
- [Addy Osmani: How to write a good spec for AI agents](https://addyosmani.com/blog/good-spec/) -- spec-driven development patterns
- [Miguel Grinberg: Why Generative AI Coding Tools Do Not Work For Me](https://blog.miguelgrinberg.com/post/why-generative-ai-coding-tools-and-agents-do-not-work-for-me) -- practitioner experience with AI completion failures
- [Claude Code Best Practices](https://code.claude.com/docs/en/best-practices) -- official guidance on skill design and context management
- Existing codebase: `~/.claude/skills/dev-lifecycle/SKILL.md`, `~/.claude/get-shit-done/references/verification-patterns.md` -- current implementation and verification infrastructure

---
*Pitfalls research for: AI Development Lifecycle Orchestrator (Claude Code Skill)*
*Researched: 2026-03-22*
