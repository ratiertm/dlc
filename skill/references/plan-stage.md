# PLAN Stage Adapter

This reference defines the complete PLAN Stage workflow -- Stage 1 of the dev-lifecycle pipeline. When Claude reads this file, it follows the instructions below to run the PLAN Stage for a user's feature.

## When This Runs

- User starts a new feature (`state.json` `current.stage = 1`)
- User says "plan", "start feature", "new feature", or similar
- SKILL.md Stage 1 section directs here via `Read: $CLAUDE_SKILL_DIR/references/plan-stage.md`

## Prerequisites

- `.lifecycle/` directory initialized (Step 0 handles this if not)
- `state.json` and `manifest.json` exist (Step 0 copies from templates if not)

## Step 0: Directory Initialization

Before any PLAN work, ensure the project directory structure exists.

**Actions:**

1. Create directories if they do not exist:
   - `.lifecycle/`
   - `.lifecycle/features/`
   - `.lifecycle/features/{feature-name}/`
   - `.lifecycle/history/`

2. If `.lifecycle/state.json` does not exist:
   - Copy from `$CLAUDE_SKILL_DIR/templates/state.json`
   - This provides the default state schema

3. If `.lifecycle/manifest.json` does not exist:
   - Copy from `$CLAUDE_SKILL_DIR/templates/manifest.json`
   - This provides the artifact registry with gate rules

4. Update `state.json`:
   - `current.stage` = `1`
   - `current.stage_name` = `"PLAN"`
   - `current.status` = `"in_progress"`
   - `current.feature` = `"{feature-name}"` (kebab-case)
   - `current.mode` = keep existing or default to `"feature"`
   - `progress.current_stage_started_at` = current ISO 8601 timestamp

## Step 1: Preflight Check

**Purpose:** Surface historical lessons and known risks before planning begins. This prevents repeating past mistakes.

**CRITICAL:** Preflight failure = warning only. Do NOT block spec generation because of warnings. Always proceed to Step 2 after preflight completes.

**Check sources in priority order:**

1. **Retrospective lessons (HIGHEST PRIORITY):**
   - Read `.lifecycle/history/` for retrospective artifacts
   - Look for "Lessons for Next Phase" or "Lessons Learned" sections
   - Extract actionable warnings relevant to the current feature domain
   - Tag each with `[LESSON]`

2. **Related ADRs:**
   - Scan `docs/decisions/` for ADR files related to the feature domain
   - Surface trade-offs and constraints already decided
   - Tag each with `[ADR]`

3. **Unresolved gaps:**
   - Check `manifest.json` for previous features with `failed` verification status
   - Identify spec steps that failed in past features
   - Tag each with `[GAP]`

4. **Technical debt:**
   - Scan related source files for `TODO`, `FIXME`, `HACK` comments
   - Only scan files directly related to the feature being planned
   - Tag each with `[DEBT]`

**Output format:**

```
PLAN Preflight -- {feature-name}
================================

Lessons from previous retrospectives:
  - [LESSON] {lesson text} (from: {retro date})
  - [LESSON] {lesson text}

Related ADRs:
  - [ADR] ADR-{NNN}: {title} -- {one-line summary}

Unresolved gaps:
  - [GAP] {feature}: {spec-id} failed at {step}

Technical debt in related area:
  - [DEBT] {file}:{line} -- {description}

(Warnings only -- proceeding to spec generation)
```

**Edge case -- first feature ever:** No retrospectives, no previous features, possibly no ADRs. Output:

```
PLAN Preflight -- {feature-name}
================================

No previous lessons found. First feature.

(Proceeding to spec generation)
```

## Step 2: E2E Spec Generation

**Read before generating:**
- `$CLAUDE_SKILL_DIR/templates/spec-template.md` -- the template to fill
- `$CLAUDE_SKILL_DIR/references/e2e-spec.md` -- format rules, ID convention, cross-artifact linking, Storage rules
- `$CLAUDE_SKILL_DIR/examples/user-login.spec.md` -- working example for reference

**Approach: Draft-then-revise**

1. Ask the user to describe the feature in their own words
2. Generate a complete draft spec with all 5 chain steps per interaction
3. Present the draft to the user for review
4. Iterate until user is satisfied with the spec content

**Why draft-then-revise:** The user sees a concrete artifact immediately and can react to what is wrong, rather than answering abstract questions. "Show, don't ask."

**Generation checklist (each item MUST be verified before presenting to user):**

- [ ] Feature name in kebab-case for YAML frontmatter `feature:` field
- [ ] One-line round-trip summary: "User does X -> stored in Y -> user sees Z"
- [ ] Each interaction has exactly 5 chain steps: Screen, Connection, Processing, Response, Error
- [ ] Every Processing step has mandatory **Storage** field with format: `{destination} -- {operation}`
  - Operations: `READ`, `WRITE`, `READ+WRITE`
  - Examples: "PostgreSQL users table -- READ", "OpenAI gpt-4 API -- POST"
- [ ] Every Error step has minimum 3 conditions: (1) client validation, (2) server/processing error, (3) infrastructure/network error
- [ ] Spec IDs follow `e2e-{feature}-{NNN}` convention, zero-padded, sequential across all interactions within the feature
- [ ] Verification Criteria are concrete and checkable (not vague assertions like "sends data to server")
- [ ] `status: draft` in YAML frontmatter
- [ ] `steps:` count in frontmatter matches actual step count

**Scope guidance:**
- Recommend 1-3 interactions (5-15 spec steps) for first features
- Soft limit: 25 steps (5 interactions) per feature
- If a feature exceeds 25 steps, suggest splitting into sub-features

**Save location:** `.lifecycle/features/{feature-name}/spec.md`

## Step 3: Prototype Generation

**Read before generating:**
- `$CLAUDE_SKILL_DIR/templates/prototype-template.html` -- the HTML boilerplate
- `$CLAUDE_SKILL_DIR/references/prototype.md` -- format rules, data-* attributes, manifest schema, hash routing
- `$CLAUDE_SKILL_DIR/examples/user-login.prototype.html` -- working example for reference

**CRITICAL:** MUST read the spec file just created in Step 2 from disk (`.lifecycle/features/{feature-name}/spec.md`). Do NOT generate the prototype from memory -- the spec file is the source of truth for spec IDs.

**Generation rules:**

For each spec step, create the corresponding HTML element:

| Spec Step | HTML Element | Key Attributes |
|-----------|-------------|----------------|
| Screen | `<div data-screen="name" data-spec-id="e2e-..." class="screen">` | Container with form fields |
| Connection | `<button data-action="name" data-spec-id="e2e-...">` | On the corresponding screen |
| Processing | `<div data-screen="loading" data-spec-id="e2e-..." data-state="loading" class="screen">` | Loading screen with 800ms auto-transition |
| Response | `<div data-screen="result" data-spec-id="e2e-..." class="screen">` | Result display with `data-mock` bindings |
| Error | `<div data-error="name" data-spec-id="e2e-..." class="error">` | On the source screen |

**Prototype structure (four sections):**

1. **HEAD:** Meta tags + inline wireframe CSS (no external stylesheets, no CDN links)
2. **MANIFEST:** `<script type="application/json" id="prototype-manifest">` with:
   - `feature`, `specFile`, `version`, `generatedFrom`
   - `screens[]` -- all screen names in navigation order
   - `specMapping[]` -- every spec step linked to its HTML element (dual spec linking)
   - `interactions[]` -- clickable elements with action/result/target
   - `fields[]` -- form inputs with type and required flag
   - `errorStates[]` -- error conditions with trigger, element, message
3. **SCREENS:** `<div id="app">` containing all `<div data-screen="...">` elements
4. **ENGINE:** `<script>` with hash router, delegated click handler, `handleAction()`, mock data, `populateScreen()`, initialization

**Dual spec linking (MANDATORY):**
- Every spec step with a visual representation MUST have BOTH:
  1. `data-spec-id="e2e-{feature}-{NNN}"` attribute on the HTML element
  2. Corresponding entry in the `specMapping` array of the manifest JSON

**Key constraints:**
- MUST work with `file://` protocol -- no `fetch()`, no `XMLHttpRequest`, no external resources, no CDN links
- Everything inline in one HTML file
- Hash-based SPA routing (no History API)
- Processing step simulated as loading screen with 800ms `setTimeout` auto-transition
- Wireframe CSS only: neutral colors, system fonts, no gradients/shadows

**Anti-pattern:** Never generate prototype before spec. Spec is source of truth for IDs. If spec changes in the feedback loop, always regenerate prototype from scratch.

**Save location:** `.lifecycle/features/{feature-name}/prototype.html`

## Step 4: Agreement Gate

**Purpose:** Hard blocking gate -- PLAN does not complete without explicit user approval. This is the core "rework prevention" mechanism.

**Protocol:**

1. **Announce:** "Prototype saved to `.lifecycle/features/{feature-name}/prototype.html`"

2. **Provide open command:**
   ```
   open .lifecycle/features/{feature-name}/prototype.html
   ```
   (macOS -- opens in default browser)

3. **Instruct:** "Please open this file in your browser and click through all interactions."

4. **Ask:** "Does this match what you expect? Any changes needed?"

5. **Verify browser interaction:** Explicitly ask: "Did you open prototype.html in your browser? Did you click through all interactions?"

6. **IF user requests changes:**
   - Clarify what needs to change
   - Regenerate spec and/or prototype as needed
   - If spec changes: ALWAYS regenerate prototype from scratch (spec is source of truth for IDs)
   - Save updated files to the same locations
   - Return to step 1 of this gate (re-announce, re-instruct)

7. **IF user approves** (says "OK", "approved", "looks good", "yes", etc.):
   - Proceed to Spec Baseline Snapshot (step 4b), then Artifact Registration

**4b. Spec Baseline Snapshot** (Read: `$CLAUDE_SKILL_DIR/references/observability.md` Section "Spec Baseline & Diff"):
   - Create `.lifecycle/analytics/spec-baselines/` directory if not exists
   - If `.lifecycle/analytics/spec-baselines/{feature}.baseline.md` does NOT exist:
     - Copy `.lifecycle/features/{feature}/spec.md` to `.lifecycle/analytics/spec-baselines/{feature}.baseline.md`
   - If baseline already exists: skip (preserve original baseline)

**CRITICAL: MUST NOT set `1_plan.completed_at` until user says words equivalent to approval. No auto-completion. No "looks like it's good enough." The user's explicit approval is the gate.**

## Artifact Registration

After user approves at the Agreement Gate:

### 1. Update spec status

Open `.lifecycle/features/{feature-name}/spec.md` and change YAML frontmatter:
```yaml
status: draft    # change to:
status: agreed
```

### 2. Register spec in manifest

Add to `manifest.json` under `artifacts.1_plan.outputs`:
```json
{
  "type": "spec",
  "path": ".lifecycle/features/{feature-name}/spec.md",
  "created_at": "{ISO-8601 timestamp}",
  "status": "complete"
}
```

### 3. Register prototype in manifest

Add to `manifest.json` under `artifacts.1_plan.outputs`:
```json
{
  "type": "prototype",
  "path": ".lifecycle/features/{feature-name}/prototype.html",
  "created_at": "{ISO-8601 timestamp}",
  "status": "complete"
}
```

### 4. Mark PLAN stage complete

Set `manifest.json` `artifacts.1_plan.completed_at` to current ISO 8601 timestamp.

### 5. Update state

Set `state.json`:
- `current.status` = `"completed"`
- `session.last_active` = current ISO 8601 timestamp
- `session.resume_hint` = `"PLAN complete. Ready for DO stage."`

### 6. Announce completion

Tell the user: **"PLAN complete. Ready for DO stage."**

## Anti-Patterns

These are the most common mistakes. Do NOT do any of these:

| # | Anti-Pattern | Why It Is Wrong |
|---|-------------|-----------------|
| 1 | **Auto-completing PLAN without user approval** | The entire point of the agreement gate is user confirmation. Never skip it. |
| 2 | **Generating prototype before spec** | Prototype is derived from spec IDs. Without spec first, there is no traceability. |
| 3 | **Blocking on preflight failures** | CONTEXT.md explicitly says preflight failure = warning only. Never prevent spec generation because of warnings. |
| 4 | **Calling GSD/PDCA for spec generation** | dev-lifecycle generates spec+prototype independently. Do not delegate to `/gsd:plan-phase` or `/pdca plan` for this work. |
| 5 | **Generating spec without Storage field** | The spec format makes Storage mandatory in Processing. Missing Storage is the most common "done but broken" failure mode. |
| 6 | **Skipping error conditions (minimum 3 required)** | Error step requires at least 3 conditions: client validation, server error, network error. Error handling is where most "done but broken" bugs live. |

## Relationship to Other Skills

- **GSD:** Remains available for project-level planning (roadmap, phases, milestones). dev-lifecycle's PLAN Stage handles spec+prototype generation independently. These are complementary, not competing.
- **PDCA:** Quality support skill. dev-lifecycle does not invoke PDCA during PLAN. PDCA may be used separately by the user for quality cycles.
- **ADR:** Architecture Decision Records. Referenced during preflight check if `docs/decisions/` exists. ADR decisions inform spec constraints.

## File Locations

| File | Path | Purpose |
|------|------|---------|
| This adapter | `$CLAUDE_SKILL_DIR/references/plan-stage.md` | PLAN Stage workflow instructions |
| Spec template | `$CLAUDE_SKILL_DIR/templates/spec-template.md` | Template to fill during spec generation |
| Prototype template | `$CLAUDE_SKILL_DIR/templates/prototype-template.html` | HTML boilerplate for prototype |
| Spec format reference | `$CLAUDE_SKILL_DIR/references/e2e-spec.md` | ID convention, cross-artifact linking, Storage rules |
| Prototype format reference | `$CLAUDE_SKILL_DIR/references/prototype.md` | data-* attributes, manifest schema, hash routing |
| Spec example | `$CLAUDE_SKILL_DIR/examples/user-login.spec.md` | Working login spec for reference |
| Prototype example | `$CLAUDE_SKILL_DIR/examples/user-login.prototype.html` | Working login prototype for reference |
| Stage transitions | `$CLAUDE_SKILL_DIR/references/stage-transitions.md` | Gate conditions, status values |
| State template | `$CLAUDE_SKILL_DIR/templates/state.json` | State schema (copied on init) |
| Manifest template | `$CLAUDE_SKILL_DIR/templates/manifest.json` | Artifact registry (copied on init) |

**Runtime location:** `$CLAUDE_SKILL_DIR` = `~/.claude/skills/dev-lifecycle`
**Version-controlled location:** `skill/` directory in the project repository
