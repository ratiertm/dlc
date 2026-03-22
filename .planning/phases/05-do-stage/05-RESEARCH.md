# Phase 5: DO Stage - Research

**Researched:** 2026-03-22
**Domain:** Stage adapter pattern, spec-driven implementation tracking, deviation logging, ADR integration
**Confidence:** HIGH

## Summary

Phase 5 implements the DO Stage adapter (`do-stage.md`) following the exact same structural pattern as the existing `plan-stage.md` (311 lines, Step 0-N structure with anti-patterns). The DO Stage uses the approved E2E spec as an implementation checklist, tracks per-step completion (pending -> implemented), logs deviations inside spec.md with user confirmation, and auto-detects ADR-worthy moments (deviations + tradeoff conversations).

This phase is primarily a "reference document authoring" task -- no code, no scripts, just a Markdown adapter file and updates to SKILL.md and related docs. The patterns are well-established from Phase 4 (plan-stage.md). The new complexity is in three areas: (1) deviation logging workflow with user confirmation gate, (2) ADR trigger detection heuristics, and (3) the interplay between `// SPEC:` comments and `// WHY:` + `// SEE:` comments from the ADR skill.

**Primary recommendation:** Mirror plan-stage.md structure exactly (Step 0 init, Step 1-N workflow, anti-patterns table, file locations). Focus design effort on the deviation log format and ADR detection triggers -- these are the novel elements.

<user_constraints>

## User Constraints (from CONTEXT.md)

### Locked Decisions
- Spec checklist tracking: per-step tracking (Screen done, Connection done, etc.) -- already decided in Phase 2
- Code comments: `// SPEC: e2e-{feature}-{NNN}` inserted near implementations -- already decided in Phase 2
- Deviation log location: inside spec.md (not a separate file) -- decided in Phase 5 discussion
- Deviation workflow: record + user confirmation required before proceeding -- if user rejects, implement per original spec
- ADR triggers: both deviation detection AND tradeoff detection active
- ADR code comments: WHY+SEE pattern from existing ADR skill

### Claude's Discretion
- Deviation log markdown format (spec.md internal positioning, field structure)
- Checklist tracking visual representation
- ADR detection keywords/patterns
- DO Stage reference file (do-stage.md) structure

### Deferred Ideas (OUT OF SCOPE)
None -- discussion stayed within phase scope

</user_constraints>

<phase_requirements>

## Phase Requirements

| ID | Description | Research Support |
|----|-------------|-----------------|
| SPEC-03 | DO에서 E2E Spec의 각 단계를 구현 체크리스트로 추적 | Spec step status lifecycle (pending -> implemented), `// SPEC:` comment convention, manifest registration |
| SPEC-05 | DO 중 Spec에서 이탈 시 deviation log에 기록하는 메커니즘 | Deviation log format (DEV-NNN) already defined in e2e-spec.md and spec-template.md, user confirmation gate |
| PIPE-02 | Stage 2 DO -- Spec 기준 구현 + deviation log + ADR 자동 감지 + WHY+SEE 주석 | Full do-stage.md adapter: Step 0 init, Step 1 spec load, Step 2 implementation loop, Step 3 deviation handling, ADR integration |

</phase_requirements>

## Standard Stack

This phase produces Markdown reference documents, not code. No libraries needed.

### Core Artifacts to Produce

| Artifact | Path | Purpose | Pattern Source |
|----------|------|---------|---------------|
| DO Stage adapter | `skill/references/do-stage.md` | Complete DO Stage workflow instructions | `skill/references/plan-stage.md` (311 lines) |
| SKILL.md update | `skill/SKILL.md` | Stage 2 section points to do-stage.md | Stage 1 section pattern (line 62) |
| Runtime copy | `~/.claude/skills/dev-lifecycle/references/do-stage.md` | Runtime skill location | Same as plan-stage.md deployment |

### Supporting Files (May Need Updates)

| File | Update Needed | Reason |
|------|---------------|--------|
| `skill/references/e2e-spec.md` | Minimal -- deviation format already defined | Verify DO Stage usage section is complete |
| `skill/references/role-matrix.md` | No change needed | Stage 2 DO row already correct |
| `skill/references/stage-transitions.md` | No change needed | 1_to_2 and 2_to_3 gates already defined |

## Architecture Patterns

### Recommended do-stage.md Structure

Following plan-stage.md exactly:

```
skill/references/do-stage.md
├── When This Runs          # Stage 2 entry conditions
├── Prerequisites            # Approved spec required (gate 1_to_2)
├── Step 0: Stage Init      # state.json update, spec loading
├── Step 1: Spec Load       # Read approved spec, build checklist
├── Step 2: Implementation  # Per-step implementation loop
├── Step 3: Deviation Log   # When spec differs from implementation
├── Step 4: ADR Detection   # Tradeoff/deviation -> ADR suggestion
├── Step 5: Completion      # Artifact registration, state update
├── Anti-Patterns           # Table of common mistakes
├── Relationship to Other Skills
└── File Locations          # Reference table
```

### Pattern 1: Per-Step Checklist Tracking

**What:** Each spec step (e2e-{feature}-{NNN}) has a status field that transitions from `pending` to `implemented` as code is written.

**When to use:** Every time a spec step's implementation is completed.

**Mechanism:**
```markdown
## e2e-login-001: Login Form Display

**Chain:** Screen
**Status:** implemented    <!-- was: pending -->
```

The DO stage adapter reads the spec file, identifies all steps with `**Status:** pending`, and presents them as a checklist:

```
DO Stage Checklist -- user-login
================================
[x] e2e-login-001: Login Form Display (Screen)
[x] e2e-login-002: Login Submission (Connection)
[ ] e2e-login-003: Credential Validation (Processing)
[ ] e2e-login-004: Successful Login Response (Response)
[ ] e2e-login-005: Login Error Handling (Error)

Progress: 2/5 steps implemented
```

After each step is implemented, the adapter:
1. Updates the step's `**Status:**` from `pending` to `implemented`
2. Verifies `// SPEC: e2e-{feature}-{NNN}` comment exists in the code
3. Updates the checklist display

### Pattern 2: SPEC Comment Convention

**What:** Code comments linking implementation to spec steps.

**Format:**
```typescript
// SPEC: e2e-login-001 -- Login Form Display
export function LoginForm() {
  // ...
}
```

**Rules:**
- One `// SPEC:` comment per spec step, at the most relevant code location
- Format: `// SPEC: {spec-id} -- {step title}`
- Place directly above the function/component/handler that implements the step
- For steps spanning multiple files, place at the entry point

### Pattern 3: Deviation Log with User Confirmation

**What:** When implementation must differ from spec, record the deviation and get user approval.

**Workflow:**
```
1. Claude detects deviation (endpoint changed, storage differs, etc.)
2. Pause implementation
3. Present to user:
   "Spec과 다르게 구현하려 합니다.
    Step: e2e-login-003 (Credential Validation)
    Original: PostgreSQL users table -- READ
    Proposed: MongoDB users collection -- READ
    Reason: Project uses MongoDB, not PostgreSQL
    계속할까요?"
4a. User approves -> Record deviation in spec.md, continue
4b. User rejects -> Implement per original spec
```

**Deviation log format (already defined in e2e-spec.md and spec-template.md):**
```markdown
## Deviations

### DEV-001: MongoDB instead of PostgreSQL
- **Spec step:** e2e-login-003
- **Original:** PostgreSQL users table -- READ
- **Actual:** MongoDB users collection -- READ
- **Reason:** Project uses MongoDB as primary database
- **Approved:** yes
```

**Key design decisions:**
- Deviations are appended to the `## Deviations` section at the bottom of spec.md
- DEV-NNN numbering is sequential within a feature (same as spec IDs)
- `**Approved:**` field records user's confirmation status
- Only record deviations for substantive changes (see e2e-spec.md "When NOT to record")

### Pattern 4: ADR Auto-Detection

**What:** Detect moments where architectural decisions are being made and suggest ADR creation.

**Two triggers (both active per CONTEXT.md):**

1. **Deviation trigger:** When a deviation is recorded, check if it represents an architectural decision (not just a minor change). If the deviation involves choosing a different technology, pattern, or approach -> suggest ADR.

2. **Tradeoff trigger:** When the conversation contains tradeoff language -> suggest ADR.
   - Detection keywords/patterns:
     - "A 대신 B" / "instead of" / "rather than"
     - "chose X over Y" / "X로 결정"
     - "tradeoff" / "trade-off" / "트레이드오프"
     - "이건 나중에 바꿔야" (intentional tech debt)
     - Comparing 2+ options with pros/cons

**ADR creation flow (delegates to ADR skill):**
1. Detect trigger
2. Suggest: "이 결정은 ADR로 기록할 가치가 있습니다. ADR을 생성할까요?"
3. If user agrees -> Follow ADR skill's 6-step workflow
4. After ADR created -> Insert WHY+SEE comments in code

### Pattern 5: WHY+SEE + SPEC Comment Coexistence

**What:** When a spec step also has an ADR, both comment types appear together.

**Example:**
```typescript
// SPEC: e2e-login-003 -- Credential Validation
// WHY: MongoDB 선택 -- PostgreSQL 대비 스키마 유연성 우선
// SEE: docs/decisions/003-mongodb-over-postgresql.md
async function validateCredentials(email: string, password: string) {
  // ...
}
```

**Rules:**
- `// SPEC:` always comes first (it's the structural link)
- `// WHY:` + `// SEE:` follow (they explain the decision)
- If no ADR exists for a step, only `// SPEC:` appears

### Anti-Patterns to Avoid

- **Starting DO without approved spec:** Gate 1_to_2 requires `artifacts.1_plan.outputs.length > 0 && artifacts.1_plan.completed_at != null`. Never skip this.
- **Silently deviating from spec:** Every substantive deviation must be recorded and user-confirmed. Silent deviations defeat the purpose of the spec.
- **Auto-approving deviations:** User must explicitly confirm. The adapter pauses and waits.
- **Skipping SPEC comments:** Every implemented step MUST have a `// SPEC:` comment in the code. This is the traceability thread.
- **Creating ADRs for every deviation:** Not all deviations are architectural decisions. Minor changes (endpoint path adjustment, field name change) don't need ADRs.

## Don't Hand-Roll

| Problem | Don't Build | Use Instead | Why |
|---------|-------------|-------------|-----|
| ADR document creation | Custom ADR format | Existing ADR skill (6-step workflow) | ADR skill already handles numbering, format, WHY+SEE comments |
| Spec status tracking | Custom status tracking | Spec file's built-in `**Status:**` fields | Status lifecycle already defined in e2e-spec.md |
| Deviation log format | Custom deviation format | Format already in spec-template.md and e2e-spec.md | DEV-NNN format with 5 fields already standardized |
| Stage transition logic | Custom gate checks | stage-transitions.md rules | Gates 1_to_2 and 2_to_3 already defined |

**Key insight:** Most of the DO Stage's mechanisms are already defined in Phase 2 (spec format) and Phase 1 (stage transitions). The adapter's job is to orchestrate these existing mechanisms into a coherent workflow.

## Common Pitfalls

### Pitfall 1: Spec Status vs Spec-Level Status Confusion
**What goes wrong:** Mixing up step-level status (`pending -> implemented`) with spec-level status (`agreed -> implementing`).
**Why it happens:** Two different status lifecycles operate simultaneously.
**How to avoid:** Step 0 of DO Stage sets spec-level status to `implementing`. Each step's status updates independently. Only TEST stage changes spec-level to `verified` or `failed`.
**Warning signs:** Spec status showing `implemented` (not a valid spec-level status).

### Pitfall 2: Deviation Without User Confirmation
**What goes wrong:** Recording a deviation and continuing without asking the user.
**Why it happens:** Implementation momentum -- it feels natural to note the change and keep going.
**How to avoid:** DO Stage adapter MUST pause and present the deviation to the user with a clear yes/no question. No auto-proceeding.
**Warning signs:** Deviations in spec.md with `**Approved:** ` blank or missing.

### Pitfall 3: Over-triggering ADR Suggestions
**What goes wrong:** Suggesting ADR creation for every minor decision, causing fatigue.
**Why it happens:** Overly broad keyword matching on tradeoff language.
**How to avoid:** ADR suggestion threshold: the decision must involve (a) choosing between 2+ viable approaches AND (b) having lasting consequences. Routine implementation choices don't qualify.
**Warning signs:** User consistently declining ADR suggestions.

### Pitfall 4: Missing Manifest Registration
**What goes wrong:** DO Stage completes but artifacts aren't registered in manifest.json, blocking the 2_to_3 gate.
**Why it happens:** Forgetting the artifact registration step at the end.
**How to avoid:** Step 5 (Completion) must register outputs in `artifacts.2_do.outputs` before updating state to completed.
**Warning signs:** `artifacts.2_do.outputs.length === 0` when trying to transition to TEST.

### Pitfall 5: Spec File Not Read from Disk
**What goes wrong:** Working from memory of the spec instead of reading the actual file.
**Why it happens:** Claude's context window may have the spec from the PLAN stage.
**How to avoid:** DO Stage Step 1 MUST explicitly read spec.md from `.lifecycle/features/{feature-name}/spec.md`. This mirrors plan-stage.md's prototype generation rule: "Do NOT generate from memory -- the spec file is the source of truth."
**Warning signs:** SPEC comment IDs don't match the actual spec file.

## Code Examples

### DO Stage Step 0: State Initialization
```markdown
# From do-stage.md (to be created)

## Step 0: Stage Initialization

**Actions:**

1. Read `.lifecycle/state.json` and verify:
   - `current.stage` = `2` (or transition from 1 to 2)
   - `current.feature` has a value

2. Check gate condition (1_to_2):
   - Read `.lifecycle/manifest.json`
   - Verify: `artifacts.1_plan.outputs.length > 0`
   - Verify: `artifacts.1_plan.completed_at != null`
   - If gate fails: set `current.status = "blocked"`, display missing artifacts, STOP

3. Update `state.json`:
   - `current.stage` = `2`
   - `current.stage_name` = `"DO"`
   - `current.status` = `"in_progress"`
   - `progress.current_stage_started_at` = current ISO 8601 timestamp

4. Read approved spec from `.lifecycle/features/{feature-name}/spec.md`
   - Verify YAML frontmatter `status: agreed`
   - If status is not `agreed`: STOP, inform user spec must be approved first

5. Update spec YAML frontmatter:
   - `status: agreed` -> `status: implementing`
```

### DO Stage Checklist Display Format
```
DO Stage -- user-login
======================

Spec: .lifecycle/features/user-login/spec.md
Status: implementing (2/5 steps done)

Checklist:
  [x] e2e-login-001: Login Form Display (Screen)
  [x] e2e-login-002: Login Submission (Connection)
  [ ] e2e-login-003: Credential Validation (Processing)
  [ ] e2e-login-004: Successful Login Response (Response)
  [ ] e2e-login-005: Login Error Handling (Error)

Deviations: 0 recorded
ADRs created: 0

Next step: e2e-login-003 (Processing)
```

### Deviation User Confirmation Prompt
```
⚠️ Spec Deviation Detected
============================

Step: e2e-login-003 (Credential Validation)

Spec says:
  Storage: PostgreSQL users table -- READ

Implementing as:
  Storage: MongoDB users collection -- READ

Reason: 프로젝트가 MongoDB를 사용하므로 PostgreSQL 대신 MongoDB로 구현

이대로 진행할까요? (yes → deviation 기록 후 계속 / no → 원래 spec대로 구현)
```

### SKILL.md Stage 2 Section Update
```markdown
### Stage 2: DO

Purpose: Implement against approved spec. Track progress, log deviations, detect ADR-worthy decisions.
Primary: dev-lifecycle (spec-driven implementation tracking)
Supporting: ADR (auto-detect trade-offs), WHY+SEE code comments
Outputs: Code changes, updated spec (step statuses + deviations), ADR documents

Read: `$CLAUDE_SKILL_DIR/references/do-stage.md`

Pipeline:
1. Stage init (gate check, spec load, status update)
2. Per-step implementation (implement each chain step, add SPEC comments)
3. Deviation handling (detect, present to user, record with approval)
4. ADR detection (deviation + tradeoff triggers, suggest ADR creation)
5. Completion (artifact registration, state update)
```

## State of the Art

| Old Approach (SKILL.md current) | New Approach (after Phase 5) | Impact |
|--------------------------------|------------------------------|--------|
| Stage 2 says "Primary: PDCA (`/pdca do`)" | Primary: dev-lifecycle (do-stage.md adapter) | Consistent with Phase 4 decision -- dev-lifecycle is primary for inner loop stages |
| No deviation tracking | Deviation log in spec.md with user confirmation | Captures why implementation differs from plan |
| ADR suggested manually | Two automatic triggers (deviation + tradeoff) | Reduces forgotten architectural decisions |
| No per-step tracking | Status field per spec step (pending -> implemented) | Visible progress, step-level traceability |

**Important update needed:** The current SKILL.md Stage 2 section lists PDCA as primary skill. Following the Phase 4 decision pattern (where GSD was moved to supporting role), DO Stage should make dev-lifecycle primary and PDCA supporting. This aligns with how PLAN Stage works.

## Validation Architecture

### Test Framework

| Property | Value |
|----------|-------|
| Framework | Manual validation (Markdown document authoring, no executable code) |
| Config file | none |
| Quick run command | Visual review of do-stage.md structure vs plan-stage.md |
| Full suite command | Diff plan-stage.md structure against do-stage.md for completeness |

### Phase Requirements -> Test Map

| Req ID | Behavior | Test Type | Automated Command | File Exists? |
|--------|----------|-----------|-------------------|-------------|
| SPEC-03 | Per-step checklist tracking in spec | manual-only | Review do-stage.md Step 2 for status update instructions | N/A |
| SPEC-05 | Deviation log mechanism | manual-only | Review do-stage.md Step 3 for DEV-NNN format + user confirmation | N/A |
| PIPE-02 | Full DO Stage adapter | manual-only | Verify do-stage.md covers: spec load, implementation loop, deviation log, ADR detection, WHY+SEE | N/A |

**Justification for manual-only:** This phase produces Markdown reference documents, not executable code. Validation is structural review -- does the document cover all required workflow steps, does it follow plan-stage.md patterns, are anti-patterns documented.

### Sampling Rate
- **Per task commit:** Visual diff against plan-stage.md structure
- **Per wave merge:** Read-through of complete do-stage.md for workflow completeness
- **Phase gate:** All three requirements (SPEC-03, SPEC-05, PIPE-02) addressed in do-stage.md

### Wave 0 Gaps
None -- no test infrastructure needed for Markdown document authoring.

## Open Questions

1. **SKILL.md Stage 2 Primary Skill Update**
   - What we know: Phase 4 set precedent of moving dev-lifecycle to primary role (GSD became supporting for Stage 1). Current SKILL.md still lists PDCA as primary for Stage 2.
   - What's unclear: Should PDCA be fully removed or kept as "supporting"?
   - Recommendation: Follow Phase 4 pattern -- dev-lifecycle primary, PDCA supporting. Update both SKILL.md and role-matrix.md.

2. **Deviation Numbering Scope**
   - What we know: DEV-NNN format defined in e2e-spec.md. Sequential within a feature.
   - What's unclear: Does numbering reset per feature or continue globally?
   - Recommendation: Per-feature numbering (DEV-001, DEV-002 within each spec.md file), matching the spec ID pattern which is also per-feature.

## Sources

### Primary (HIGH confidence)
- `skill/references/plan-stage.md` -- Canonical pattern for stage adapter structure (311 lines, Step 0-4 + anti-patterns)
- `skill/references/e2e-spec.md` -- Deviation log format, spec status lifecycle, SPEC comment convention
- `skill/templates/spec-template.md` -- Deviation section template with DEV-NNN format
- `skill/references/stage-transitions.md` -- Gate conditions for 1_to_2 and 2_to_3
- `~/.claude/skills/adr/SKILL.md` -- WHY+SEE comment pattern, ADR 6-step workflow, proactive detection triggers
- `.planning/phases/05-do-stage/05-CONTEXT.md` -- Locked decisions for Phase 5

### Secondary (MEDIUM confidence)
- `skill/references/role-matrix.md` -- Stage 2 skill assignments (may need updating)
- `skill/references/skill-invocation.md` -- PDCA/ADR invocation patterns for Stage 2

### Tertiary (LOW confidence)
- None -- all findings sourced from project's own canonical documents

## Metadata

**Confidence breakdown:**
- Standard stack: HIGH -- all artifacts follow established plan-stage.md pattern
- Architecture: HIGH -- deviation log format already defined in Phase 2, ADR integration well-documented
- Pitfalls: HIGH -- derived from concrete plan-stage.md anti-patterns and spec format rules

**Research date:** 2026-03-22
**Valid until:** 2026-04-22 (stable -- internal project documentation)
