# Phase 4: PLAN Stage - Research

**Researched:** 2026-03-22
**Domain:** PLAN Stage adapter -- preflight check, E2E spec generation, prototype generation, user agreement gate
**Confidence:** HIGH

## Summary

Phase 4 implements the PLAN Stage adapter, the first stage in the dev-lifecycle inner loop. This phase takes the spec format (Phase 2) and prototype format (Phase 3) from template/reference/example artifacts and turns them into an executable workflow: when a user requests a feature, dev-lifecycle (1) runs a preflight check surfacing lessons/gaps, (2) generates an E2E spec by filling the spec template, (3) generates a clickable prototype from the spec, and (4) blocks until the user explicitly approves both artifacts in a feedback loop.

The critical insight from CONTEXT.md is that dev-lifecycle does this independently -- it does NOT call GSD plan-phase or PDCA plan to produce the spec+prototype. GSD remains available for project-level planning, but the PLAN Stage's spec+prototype generation is dev-lifecycle's own responsibility. This means the PLAN Stage adapter is a self-contained workflow within SKILL.md's references, not a delegation to other skills.

**Primary recommendation:** Implement the PLAN Stage as a single reference file (`references/plan-stage.md`) containing: preflight check procedure, spec generation instructions (using the existing template), prototype generation instructions (using the existing template), agreement gate protocol, and artifact registration in manifest.json. The adapter is Claude-instructions-as-code, not executable code.

<user_constraints>
## User Constraints (from CONTEXT.md)

### Locked Decisions
- dev-lifecycle independently generates spec+prototype (does NOT call GSD/PDCA for PLAN)
- GSD is primary project planning tool, PDCA is quality support, dev-lifecycle adds spec+prototype layer
- Preflight: retrospective lessons are highest priority, failure = warning only (not blocking)
- Agreement gate: user clicks prototype in browser, feedback loop until approval
- Hard gate: PLAN does not complete without explicit user approval

### Claude's Discretion
- Preflight check output format
- Spec generation conversation style (question-then-generate vs draft-then-revise)
- Prototype generation time/complexity management
- .lifecycle/ directory structure initialization timing

### Deferred Ideas (OUT OF SCOPE)
None -- discussion stayed within phase scope
</user_constraints>

<phase_requirements>
## Phase Requirements

| ID | Description | Research Support |
|----|-------------|-----------------|
| SPEC-02 | E2E Spec written in PLAN stage, user agreement workflow | Spec template exists (`skill/templates/spec-template.md`), format reference exists (`skill/references/e2e-spec.md`), example exists (`skill/examples/user-login.spec.md`). Adapter fills template from user description, manages status lifecycle `draft -> agreed`. |
| PROTO-05 | User clicks prototype in browser, provides feedback, agreement gate | Prototype template exists (`skill/templates/prototype-template.html`), format reference exists (`skill/references/prototype.md`), example exists (`skill/examples/user-login.prototype.html`). Adapter generates from agreed spec, opens for user review, iterates until approval. |
| PIPE-01 | Stage 1 PLAN -- preflight check + E2E Spec + Prototype + user agreement gate | Full PLAN Stage pipeline: (1) preflight surfaces lessons/gaps, (2) spec generation, (3) prototype generation, (4) agreement gate. All artifacts registered in manifest.json, state.json updated. |
</phase_requirements>

## Standard Stack

This project is a Claude Code skill (pure Markdown instructions + JSON state), not a software application. There are no npm packages or libraries. The "stack" is the set of existing artifacts that the PLAN Stage adapter builds upon.

### Core Assets (Already Built in Phases 1-3)

| Asset | Path | Purpose | Status |
|-------|------|---------|--------|
| Spec template | `skill/templates/spec-template.md` | 5-step chain template with YAML frontmatter | Complete |
| Prototype template | `skill/templates/prototype-template.html` | 4-section HTML boilerplate | Complete |
| Spec format reference | `skill/references/e2e-spec.md` | ID convention, cross-artifact linking, status lifecycle, storage rules | Complete |
| Prototype format reference | `skill/references/prototype.md` | data-* attributes, manifest schema, hash routing, spec linking | Complete |
| Login spec example | `skill/examples/user-login.spec.md` | Working 5-step spec for login feature | Complete |
| Login prototype example | `skill/examples/user-login.prototype.html` | Working clickable prototype for login | Complete |
| Stage transitions | `skill/references/stage-transitions.md` | Gate conditions, status values, transition procedure | Complete |
| Role matrix | `skill/references/role-matrix.md` | Stage-skill mapping, orchestration pattern | Complete |
| State template | `skill/templates/state.json` | Current stage, feature, mode tracking | Complete |
| Manifest template | `skill/templates/manifest.json` | Artifact registry with gate rules | Complete |
| SKILL.md | `skill/SKILL.md` | Main skill file, 252 lines, progressive disclosure | Complete |

### New Artifacts to Create (Phase 4 Output)

| Asset | Path | Purpose |
|-------|------|---------|
| PLAN Stage reference | `skill/references/plan-stage.md` | Complete PLAN Stage adapter: preflight, spec gen, prototype gen, agreement gate |

## Architecture Patterns

### Recommended Project Structure (No Changes to Existing)

```
skill/
  references/
    plan-stage.md          # NEW -- Phase 4 primary output
    stage-transitions.md   # Existing
    role-matrix.md         # Existing
    e2e-spec.md            # Existing
    prototype.md           # Existing
    ...
  templates/
    spec-template.md       # Existing (consumed by plan-stage.md)
    prototype-template.html # Existing (consumed by plan-stage.md)
    state.json             # Existing
    manifest.json          # Existing
  examples/
    user-login.spec.md     # Existing (reference for generation)
    user-login.prototype.html # Existing (reference for generation)
  SKILL.md                 # Existing -- Stage 1 section will point to plan-stage.md
```

### Pattern 1: PLAN Stage as Sequential Pipeline

**What:** The PLAN Stage adapter defines a 4-step sequential pipeline: Preflight -> Spec Generation -> Prototype Generation -> Agreement Gate. Each step has entry conditions, actions, and outputs.

**When to use:** Every time a user starts a new feature in dev-lifecycle.

**Flow:**

```
User requests feature
    |
    v
[1. PREFLIGHT CHECK]
    | - Read previous retrospective lessons (highest priority)
    | - Surface related ADRs
    | - Check for unresolved gaps / tech debt
    | - Output: warnings (non-blocking)
    |
    v
[2. E2E SPEC GENERATION]
    | - Gather feature details from user
    | - Fill spec-template.md for each interaction
    | - Ensure all 5 chain steps present
    | - Ensure Storage field populated
    | - Save to .lifecycle/features/{name}/spec.md
    | - Status: draft
    |
    v
[3. PROTOTYPE GENERATION]
    | - Read the spec just created
    | - Generate prototype.html from prototype-template.html
    | - Map each spec step to data-spec-id elements
    | - Populate manifest JSON in prototype
    | - Save to .lifecycle/features/{name}/prototype.html
    |
    v
[4. AGREEMENT GATE]
    | - Tell user to open prototype.html in browser
    | - Ask user to click through all flows
    | - Collect feedback
    |   |
    |   +-- User requests changes --> back to step 2 or 3
    |   |
    |   +-- User approves -->
    |       - Update spec status: draft -> agreed
    |       - Register artifacts in manifest.json
    |       - Update state.json: stage status -> completed
    |       - PLAN complete, ready for DO
```

### Pattern 2: Preflight Check Structure

**What:** A non-blocking check that surfaces historical lessons before planning begins. Failure is warning-only -- it informs but does not block.

**Preflight sources (priority order):**

1. **Retrospective lessons** (HIGHEST) -- Read `.lifecycle/history/` and any retrospective artifacts for "Lessons for Next Phase" sections. This prevents repeating the same mistakes.
2. **Related ADRs** -- Scan `docs/decisions/` for ADRs related to the feature domain. Surface trade-offs already made.
3. **Unresolved gaps** -- Check manifest.json for any previous features with `failed` verification status.
4. **Technical debt** -- Scan codebase for TODO/FIXME/HACK comments in related files.

**Output format (Claude's Discretion recommendation):**

```
PLAN Preflight -- {feature-name}
================================

Lessons from previous retrospectives:
  - [LESSON] {lesson text} (from: {retro date})
  - [LESSON] {lesson text}

Related ADRs:
  - ADR-003: {title} -- {one-line summary}

Unresolved gaps:
  - {feature}: {spec-id} failed at {step}

Technical debt in related area:
  - {file}:{line} -- TODO: {description}

(Warnings only -- proceeding to spec generation)
```

**Edge case:** First feature ever -- no retrospectives, no ADRs exist. Preflight outputs "No previous lessons found. First feature." and proceeds.

### Pattern 3: Spec Generation from User Description

**What:** Claude generates the E2E spec by filling `spec-template.md` based on user's feature description.

**Recommended approach (Claude's Discretion -- draft-then-revise):**

1. Ask user to describe the feature in their own words
2. Generate a complete draft spec with all 5 chain steps per interaction
3. Present the draft to user for review
4. Iterate until user is satisfied
5. Set status to `draft` (becomes `agreed` only at agreement gate)

**Why draft-then-revise over question-then-generate:** The user sees a concrete artifact immediately and can react to what is wrong, rather than answering abstract questions. This matches the project's philosophy -- "show, don't ask."

**Spec generation checklist:**

- [ ] Feature name in kebab-case for frontmatter
- [ ] One-line round-trip summary
- [ ] Each interaction has exactly 5 chain steps (Screen, Connection, Processing, Response, Error)
- [ ] Every Processing step has mandatory Storage field
- [ ] Error step has minimum 3 conditions (client validation, server error, network error)
- [ ] Spec IDs follow `e2e-{feature}-{NNN}` convention, sequential across interactions
- [ ] Verification Criteria are concrete and checkable (not vague assertions)

### Pattern 4: Prototype Generation from Spec

**What:** After spec is created, Claude generates a prototype HTML file using `prototype-template.html` as the boilerplate.

**Generation rules:**

1. Read the spec just created
2. For each Screen step: create a `<div data-screen="name">` with appropriate form fields
3. For each Connection step: create a `<button data-action="name">` on the corresponding screen
4. For each Processing step: create a loading screen with `data-state="loading"` and 800ms auto-transition
5. For each Response step: create a result screen showing success data with `data-mock` bindings
6. For each Error step: create `<div data-error="name">` elements on the appropriate screen
7. Wire up `handleAction()` in ENGINE section with feature-specific logic
8. Populate MANIFEST JSON with screens, specMapping, interactions, fields, errorStates
9. Ensure dual spec linking: `data-spec-id` attributes AND manifest `specMapping` entries

**Key constraint:** Prototype MUST work with `file://` protocol. No fetch(), no external resources, no CDN links. Everything inline.

### Pattern 5: Agreement Gate Protocol

**What:** The hard gate that blocks PLAN completion until user explicitly approves.

**Protocol:**

```
1. Announce: "Prototype saved to .lifecycle/features/{name}/prototype.html"
2. Instruct: "Please open this file in your browser and click through all interactions."
3. Ask: "Does this match what you expect? Any changes needed?"
4. IF user requests changes:
   a. Clarify what needs to change
   b. Regenerate spec and/or prototype
   c. Save updated files
   d. Return to step 1
5. IF user approves:
   a. Update spec frontmatter: status: draft -> agreed
   b. Register spec.md in manifest.json under 1_plan.outputs
   c. Register prototype.html in manifest.json under 1_plan.outputs
   d. Set manifest 1_plan.completed_at to current timestamp
   e. Update state.json: current.status -> completed
   f. Announce: "PLAN complete. Ready for DO stage."
```

**Hard gate enforcement:** The PLAN Stage adapter MUST NOT set `1_plan.completed_at` until the user says words equivalent to approval ("OK", "approved", "looks good", "yes", etc.). No auto-completion. No "looks like it's good enough." The user's explicit approval is the gate.

### Anti-Patterns to Avoid

- **Auto-completing PLAN without user approval:** The entire point of Phase 4 is the agreement gate. Never skip it.
- **Generating prototype before spec:** Spec must come first because prototype is derived from spec IDs. Prototype without spec has no traceability.
- **Blocking on preflight failures:** CONTEXT.md explicitly says preflight failure = warning only. Do not prevent spec generation because of preflight warnings.
- **Calling GSD/PDCA for spec generation:** CONTEXT.md explicitly says dev-lifecycle generates spec+prototype independently. Do not delegate to `/gsd:plan-phase` or `/pdca plan` for this work.
- **Generating spec without Storage field:** The spec format reference makes Storage mandatory in Processing. Missing Storage is the most common "done but broken" failure mode.
- **Skipping error conditions:** Error step requires minimum 3 conditions. This is non-negotiable -- error handling is where most "done but broken" bugs live.

## Don't Hand-Roll

| Problem | Don't Build | Use Instead | Why |
|---------|-------------|-------------|-----|
| Spec format | New spec structure | `skill/templates/spec-template.md` + `skill/references/e2e-spec.md` | Format already defined, tested with login example |
| Prototype structure | New HTML layout | `skill/templates/prototype-template.html` + `skill/references/prototype.md` | Four-section layout, data-* conventions, manifest schema all defined |
| State tracking | Custom state management | `skill/templates/state.json` + `skill/templates/manifest.json` | Schema and gate rules already defined |
| Stage transition logic | Custom transition code | `skill/references/stage-transitions.md` | Transition procedure, gate conditions, history logging all defined |
| Spec ID convention | New ID format | `e2e-{feature}-{NNN}` per `skill/references/e2e-spec.md` | Convention established, cross-artifact linking rules defined |

**Key insight:** Phase 4 assembles existing pieces into a workflow. It does NOT create new formats or schemas. The formats were defined in Phases 2-3. Phase 4 is the "glue" that connects them into a coherent stage adapter.

## Common Pitfalls

### Pitfall 1: Role Matrix Inconsistency

**What goes wrong:** The role-matrix.md says Stage 1 PLAN's primary skill is GSD, but CONTEXT.md says dev-lifecycle generates spec+prototype independently. This creates confusion about whether GSD should be invoked.

**Why it happens:** The role matrix was designed in Phase 1 before the CONTEXT.md decision to make dev-lifecycle independent for spec+prototype generation.

**How to avoid:** The PLAN Stage adapter must clarify: GSD remains available for project-level planning (roadmap, phases), but spec+prototype generation is dev-lifecycle's own work. The role-matrix.md and SKILL.md Stage 1 section need updating to reflect this decision. The PLAN Stage reference should explicitly state the relationship: "dev-lifecycle generates spec+prototype. Users may also use GSD for broader planning. These are complementary, not competing."

**Warning signs:** If the adapter instructions say "invoke /gsd:plan-phase" for spec generation.

### Pitfall 2: .lifecycle/ Directory Not Initialized

**What goes wrong:** PLAN Stage tries to save spec.md to `.lifecycle/features/{name}/` but the directory does not exist yet.

**Why it happens:** `.lifecycle/` initialization is mentioned in SKILL.md Session Start, but the timing of first feature creation may not coincide with session start.

**How to avoid:** PLAN Stage adapter must include directory creation as step 0: ensure `.lifecycle/`, `.lifecycle/features/`, `.lifecycle/features/{feature-name}/`, and `.lifecycle/history/` all exist before writing any files. Also copy state.json and manifest.json from templates if they do not exist.

**Warning signs:** File write errors mentioning "directory does not exist."

### Pitfall 3: Prototype-Spec ID Mismatch

**What goes wrong:** Prototype uses different spec IDs than the spec file, or manifest specMapping refers to nonexistent IDs.

**Why it happens:** Spec and prototype are generated in separate steps. If the spec is revised after initial prototype generation, IDs can drift.

**How to avoid:** Always regenerate prototype from the current spec, never patch an old prototype. The spec is the source of truth for IDs. When spec is revised in the feedback loop, always regenerate the prototype from scratch.

**Warning signs:** `data-spec-id` values in prototype that do not match `## e2e-{feature}-{NNN}` headings in spec.

### Pitfall 4: Agreement Gate Without Browser Verification

**What goes wrong:** User "approves" the spec text without actually opening the prototype in a browser. The prototype may have rendering bugs, broken interactions, or missing screens.

**Why it happens:** It is easier to read spec text in the terminal than to open an HTML file in a browser.

**How to avoid:** The agreement gate protocol must explicitly ask "Did you open prototype.html in your browser?" and "Did you click through all interactions?" before accepting approval. The adapter should provide the exact file path and suggest `open .lifecycle/features/{name}/prototype.html` (macOS) command.

**Warning signs:** User approves within seconds of prototype generation (too fast to have opened a browser).

### Pitfall 5: Overly Complex First Feature

**What goes wrong:** The first feature attempted with the PLAN Stage has 20+ interactions, making the spec enormous and the prototype unwieldy.

**Why it happens:** No guidance on scope for the first use.

**How to avoid:** The adapter should recommend starting with a small feature (1-3 interactions, 5-15 spec steps). The spec format reference already has a soft limit of 25 steps per feature. The adapter should surface this guidance.

**Warning signs:** Spec file exceeds 5 interactions (25 steps) on first use.

## Code Examples

### Example: Spec File After PLAN Completion

The login example at `skill/examples/user-login.spec.md` shows the exact format. Key fields for PLAN:

```yaml
---
feature: user-login
title: User Login Flow
created: 2026-03-22
updated: 2026-03-22
status: agreed          # <-- Changed from 'draft' to 'agreed' at gate
depends_on: []
steps: 5
tags: [auth, core]
---
```

Status transitions during PLAN: `draft` (initial creation) -> `agreed` (user approves at gate).

### Example: Manifest Registration After PLAN

```json
{
  "1_plan": {
    "outputs": [
      {
        "type": "spec",
        "path": ".lifecycle/features/user-login/spec.md",
        "created_at": "2026-03-22T10:00:00Z",
        "status": "complete"
      },
      {
        "type": "prototype",
        "path": ".lifecycle/features/user-login/prototype.html",
        "created_at": "2026-03-22T10:05:00Z",
        "status": "complete"
      }
    ],
    "completed_at": "2026-03-22T10:10:00Z"
  }
}
```

Gate rule `1_to_2` checks: `artifacts.1_plan.outputs.length > 0 && artifacts.1_plan.completed_at != null` -- both spec and prototype must be registered and completed_at must be set.

### Example: State After PLAN Completion

```json
{
  "current": {
    "stage": 1,
    "stage_name": "PLAN",
    "status": "completed",
    "feature": "user-login",
    "mode": "feature"
  },
  "progress": {
    "stages_completed": [],
    "current_stage_started_at": "2026-03-22T09:50:00Z",
    "last_transition_at": null
  }
}
```

Note: `stages_completed` does not include 1 yet -- that happens when transitioning TO stage 2.

### Example: PLAN Stage Reference Structure (plan-stage.md outline)

```markdown
# PLAN Stage Adapter

## When This Runs
- User starts a new feature (state.json current.stage = 1)
- User explicitly requests "plan" or "start feature"

## Prerequisites
- .lifecycle/ directory initialized
- state.json and manifest.json exist

## Step 1: Preflight Check
[detailed instructions]

## Step 2: E2E Spec Generation
Read: $CLAUDE_SKILL_DIR/templates/spec-template.md
Read: $CLAUDE_SKILL_DIR/references/e2e-spec.md
Read: $CLAUDE_SKILL_DIR/examples/user-login.spec.md
[detailed instructions]

## Step 3: Prototype Generation
Read: $CLAUDE_SKILL_DIR/templates/prototype-template.html
Read: $CLAUDE_SKILL_DIR/references/prototype.md
Read: $CLAUDE_SKILL_DIR/examples/user-login.prototype.html
[detailed instructions]

## Step 4: Agreement Gate
[detailed instructions]

## Artifact Registration
[manifest.json and state.json updates]
```

## State of the Art

| Old Approach | Current Approach | When Changed | Impact |
|--------------|------------------|--------------|--------|
| GSD as primary PLAN skill | dev-lifecycle generates spec+prototype independently | Phase 4 CONTEXT decision | role-matrix.md and SKILL.md Stage 1 section need updating |
| role-matrix Stage 1 outputs: "plan, spec (optional)" | Stage 1 outputs: spec (required) + prototype (required) | Phase 4 CONTEXT decision | Spec and prototype are NOT optional in feature mode |
| Preflight = ROADMAP.md check | Preflight = retrospective lessons first, then ADR/gaps/debt | Phase 4 CONTEXT decision | Lessons from past mistakes are highest priority |

**Key update needed:** SKILL.md's Stage 1 section and role-matrix.md currently describe GSD as the primary skill for Stage 1 PLAN. Phase 4 implementation must update these to reflect the new reality: dev-lifecycle is primary for spec+prototype, GSD is complementary for broader planning.

## Open Questions

1. **SKILL.md Update Scope**
   - What we know: Stage 1 section needs to point to `references/plan-stage.md` and reflect independent spec+prototype generation
   - What's unclear: How much of the Stage 1 description in SKILL.md should change vs. remain as-is
   - Recommendation: Keep SKILL.md Stage 1 section brief, add "Read: references/plan-stage.md" pointer, let the reference file carry the detailed instructions

2. **Role Matrix Update**
   - What we know: role-matrix.md says "Primary Skill: GSD" for Stage 1
   - What's unclear: Should this change to "Primary: dev-lifecycle (spec+prototype)" or keep GSD as primary with a note
   - Recommendation: Update to "Primary: dev-lifecycle (spec+prototype generation), Supporting: GSD (broader planning if needed)"

3. **First Run vs. Subsequent Runs**
   - What we know: First feature has no retrospectives to surface in preflight
   - What's unclear: Should the adapter handle "first feature" differently beyond skipping preflight lessons
   - Recommendation: No special handling beyond "No previous lessons found" -- keep it simple

## Validation Architecture

### Test Framework

This project is a Claude Code skill (Markdown instructions + JSON state). There is no automated test framework. Validation is manual: invoke the PLAN Stage with a sample feature and verify correct artifacts are produced.

| Property | Value |
|----------|-------|
| Framework | Manual verification (Claude Code skill, no executable code) |
| Config file | None |
| Quick run command | Invoke dev-lifecycle PLAN with a test feature, verify outputs |
| Full suite command | Walk through complete PLAN Stage with user-login feature |

### Phase Requirements -> Test Map

| Req ID | Behavior | Test Type | Automated Command | File Exists? |
|--------|----------|-----------|-------------------|-------------|
| SPEC-02 | Spec generated from template, user reviews and agrees | manual | Open generated spec.md, verify 5-step chain + correct frontmatter | Wave 0 |
| PROTO-05 | Prototype opened in browser, user clicks through, feedback loop | manual | `open .lifecycle/features/{name}/prototype.html` in browser | Wave 0 |
| PIPE-01 | Full pipeline: preflight -> spec -> prototype -> gate | manual | Invoke PLAN stage, verify all 4 steps execute in order | Wave 0 |

### Sampling Rate
- **Per task:** Verify plan-stage.md instructions produce correct output for user-login example
- **Per wave:** Walk through complete PLAN Stage with a new feature (not user-login)
- **Phase gate:** Manually execute PLAN Stage end-to-end, verify spec.md + prototype.html + manifest.json + state.json all correct

### Wave 0 Gaps
- [ ] `skill/references/plan-stage.md` -- the primary deliverable, does not exist yet
- [ ] SKILL.md Stage 1 update -- needs to reference plan-stage.md
- [ ] role-matrix.md update -- needs to reflect dev-lifecycle as primary for spec+prototype
- [ ] skill-invocation.md update -- Stage 1 section needs to describe the independent generation flow

## Sources

### Primary (HIGH confidence)
- `skill/templates/spec-template.md` -- spec format, all fields verified
- `skill/templates/prototype-template.html` -- prototype structure, all sections verified
- `skill/references/e2e-spec.md` -- ID convention, status lifecycle, cross-artifact linking, storage rules
- `skill/references/prototype.md` -- data-* attributes, manifest schema, hash routing, anti-patterns
- `skill/references/stage-transitions.md` -- gate conditions, transition procedure
- `skill/references/role-matrix.md` -- stage-skill mapping
- `skill/examples/user-login.spec.md` -- working spec example
- `skill/examples/user-login.prototype.html` -- working prototype example
- `.planning/phases/04-plan-stage/04-CONTEXT.md` -- user decisions constraining implementation

### Secondary (MEDIUM confidence)
- `.planning/research/ARCHITECTURE.md` -- dual-artifact PLAN output pattern, project structure rationale
- `.planning/research/FEATURES.md` -- spec generation flow, agreement protocol, anti-features list
- `skill/SKILL.md` -- current Stage 1 description (will need updating)
- `skill/references/skill-invocation.md` -- current invocation patterns (will need updating)

## Metadata

**Confidence breakdown:**
- Standard stack: HIGH -- all assets are project-internal, fully readable, no external dependencies
- Architecture: HIGH -- patterns derived from existing templates, references, and examples already built in Phases 1-3
- Pitfalls: HIGH -- identified from direct analysis of CONTEXT.md decisions vs. existing SKILL.md/role-matrix.md inconsistencies

**Research date:** 2026-03-22
**Valid until:** Indefinite (project-internal artifacts, no external version drift)
