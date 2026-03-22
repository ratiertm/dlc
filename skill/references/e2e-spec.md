# E2E Feature Spec Format Reference

This document defines the structured Markdown format for specifying feature interactions as end-to-end chains. Each spec traces a complete round-trip: user action -> storage -> response back to user.

## Purpose

The E2E spec solves the "done but broken" problem -- features that appear complete but fail because one link in the chain (UI wiring, API call, storage, error handling) was missed or misconnected.

By requiring every feature interaction to be traced through a 5-step chain with explicit storage destinations, the spec:
- Makes missing wiring visible before implementation
- Provides a step-by-step checklist for the DO stage
- Enables per-step pass/fail verification in the TEST stage
- Creates a traceability thread from spec to code to verification

## 5-Step Chain Overview

Every feature interaction follows this chain:

| Step | Chain Type | What It Covers | Required Fields |
|------|-----------|----------------|-----------------|
| 1 | **Screen** | What the user sees/does to trigger the interaction | Element, User Action, Initial State |
| 2 | **Connection** | How frontend communicates with backend | Method, Endpoint, Request, Auth |
| 3 | **Processing** | What backend does, including mandatory storage | Steps[], **Storage** (mandatory) |
| 4 | **Response** | What comes back and how UI updates | Success Status, Response Shape, UI Updates[] |
| 5 | **Error** | Every failure mode with specific behavior | Conditions table (min 3 rows) |

Each step has: a spec ID heading, Chain type, Status, What section, Verification Criteria (checkboxes), and a Details section with chain-specific fields.

## Spec ID Convention

**Format:** `e2e-{feature}-{NNN}`

- `{feature}` = kebab-case feature name (e.g., `login`, `chat-send`, `file-upload`)
- `{NNN}` = zero-padded sequential number within the feature (001, 002, ...)
- Each ID maps to exactly one step in the chain

**Examples:**
- `e2e-login-001` -- Login form display (Screen)
- `e2e-login-002` -- Login submission (Connection)
- `e2e-chat-006` -- Delete message processing (Processing)

**Multi-interaction features:** IDs are sequential across all interactions within a feature. A "Chat" feature with send (5 steps) and delete (5 steps) uses `e2e-chat-001` through `e2e-chat-010`.

## Cross-Artifact Linking

The spec ID is the traceability thread that connects all artifacts:

| Artifact | How Spec ID Appears | Example |
|----------|---------------------|---------|
| `spec.md` | Section heading | `## e2e-login-001: Login Form Display` |
| `prototype.html` | HTML data attribute | `<form data-spec-id="e2e-login-001">` |
| Implementation code | Code comment | `// SPEC: e2e-login-001 -- Login Form Display` |
| `verification.json` | Object key | `{ "e2e-login-001": { "status": "passed" } }` |

This linking ensures that every spec step can be traced to its prototype element, implementation code, and test result.

## Status Lifecycle

Spec-level status (in YAML frontmatter):

```
draft ──(user agrees)──> agreed ──(DO starts)──> implementing ──(TEST passes)──> verified
                                                       |
                                                       └──(TEST fails)──> failed ──(DO fixes)──> implementing
```

Step-level status (per chain step):

```
pending ──(DO implements)──> implemented ──(TEST verifies)──> verified
                                                |
                                                └──(TEST fails)──> failed
```

**Status values:**
- `draft` -- Spec written, not yet reviewed
- `agreed` -- User has confirmed the spec is correct
- `implementing` -- DO stage is actively implementing against this spec
- `verified` -- TEST stage confirms all steps pass
- `failed` -- TEST stage found failures (spec or step level)
- `pending` -- Step not yet implemented (step-level only)
- `implemented` -- Step code exists (step-level only)

## PLAN/DO/TEST Usage

### PLAN Stage
Creates the spec file. The spec is the primary output of the PLAN stage.
- Fill in the template for each feature interaction
- Ensure all 5 chain steps are present with concrete details
- Storage field must name specific destination and operation
- User reviews and agrees (status: `draft` -> `agreed`)

### DO Stage
Uses the spec as a step-by-step implementation checklist.
- Implement each chain step in order (Screen -> Connection -> Processing -> Response -> Error)
- Add `// SPEC: e2e-{feature}-{NNN}` comments in code near each implementation
- Update step status: `pending` -> `implemented`
- Update spec status: `agreed` -> `implementing`
- Record deviations if implementation differs from spec

### TEST Stage
Verifies each step individually and generates a report.
- Check each Verification Criteria checkbox
- Update step status: `implemented` -> `verified` or `failed`
- Generate `verification.json` with per-step results keyed by spec ID
- Update spec status: `implementing` -> `verified` (all pass) or `failed` (any fail)

## Storage Field Rules

The **Storage** field in the Processing step is **mandatory**. It specifies where data is persisted or retrieved.

**Format:** `{destination} -- {operation}`

**Operations:**
- `READ` -- Data is retrieved/queried
- `WRITE` -- Data is created/updated/deleted
- `READ+WRITE` -- Both operations occur

**Examples:**
- `PostgreSQL users table -- READ` (SELECT query)
- `PostgreSQL orders table -- WRITE` (INSERT/UPDATE)
- `Redis session store -- READ+WRITE` (get + set)
- `OpenAI gpt-4 API -- POST` (external API call)
- `~/.config/app/settings.json -- WRITE` (file write)
- `S3 uploads bucket -- WRITE` (object storage)

**Why mandatory:** The user's core requirement is tracing the full round-trip flow. Without explicit storage destinations, features can appear "processed" but data is never actually saved -- the most common "done but broken" failure mode.

## Deviation Log

When the DO stage implementation differs from the spec, record the deviation in the `## Deviations` section at the bottom of the spec file.

**Format:**
```markdown
### DEV-NNN: {Brief description}
- **Spec step:** e2e-{feature}-{NNN}
- **Original:** {what the spec said}
- **Actual:** {what was implemented instead}
- **Reason:** {why the deviation was necessary}
- **Approved:** {yes/no -- user confirmed?}
```

**When to record:**
- API endpoint changed from spec
- Storage destination differs
- Additional processing steps added
- Error conditions added/removed
- UI layout differs from spec

**When NOT to record:**
- Minor wording changes
- CSS/styling details not in spec
- Implementation details not covered by spec

## Granularity Guide

- **One spec file per feature** (e.g., `user-login`, `chat-messaging`)
- **Each distinct user interaction gets its own 5-step chain** within the spec
- **Soft limit: 25 steps (5 interactions) per feature**
- If a feature exceeds 25 steps, consider splitting into sub-features (e.g., `chat-messaging` and `chat-management`)

**What counts as a "distinct interaction":**
- User login (enter credentials -> see dashboard) = 1 interaction = 5 steps
- User logout (click logout -> see login page) = 1 interaction = 5 steps
- Send chat message (type message -> see it in chat) = 1 interaction = 5 steps

## Anti-Patterns

| Anti-Pattern | Problem | Fix |
|-------------|---------|-----|
| **Vague assertions** | "Sends data to server" is not checkable | Use concrete details: "POST /api/auth/login with {email, password}" |
| **Skipping chain links** | Omitting a step hides where bugs live | All 5 steps required, even if trivial: "Processing: None -- static page render" |
| **Spec without round-trip** | Ending at "save to DB" without describing user response | Always include Response step showing what user sees after processing |
| **Over-granular specs** | 20+ steps for one interaction buries the signal | One 5-step chain per distinct interaction; split features if too many interactions |

## Manifest Integration

When a spec is created during the PLAN stage, it registers in `.lifecycle/manifest.json` under `1_plan.outputs`:

```json
{
  "1_plan": {
    "outputs": [
      {
        "type": "spec",
        "path": ".lifecycle/features/user-login/spec.md",
        "created_at": "2026-03-22T10:00:00Z",
        "status": "complete"
      }
    ]
  }
}
```

The gate rule `1_to_2` (`artifacts.1_plan.outputs.length > 0`) checks that at least one spec exists before allowing transition to the DO stage. This ensures no implementation begins without a spec to implement against.

## File Locations

- **Spec template:** `$CLAUDE_SKILL_DIR/templates/spec-template.md`
- **This reference:** `$CLAUDE_SKILL_DIR/references/e2e-spec.md`
- **Example spec:** `$CLAUDE_SKILL_DIR/examples/user-login.spec.md`
- **Runtime spec location:** `.lifecycle/features/{feature-name}/spec.md` (per-project)

Where `$CLAUDE_SKILL_DIR` = `~/.claude/skills/dev-lifecycle` (or version-controlled `skill/` directory).
