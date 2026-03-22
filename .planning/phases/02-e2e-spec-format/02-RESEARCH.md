# Phase 2: E2E Spec Format - Research

**Researched:** 2026-03-22
**Domain:** E2E feature specification format design (Markdown + YAML frontmatter)
**Confidence:** HIGH

## Summary

Phase 2 defines the E2E spec format -- the structured Markdown document that traces every feature interaction through a multi-step chain from user action to storage and back. This is a format design phase, not an implementation phase. The deliverable is a `.spec.md` template with frontmatter schema, a spec ID convention (`e2e-{feature}-{NNN}`), and a realistic example demonstrating the full round-trip flow.

The user has locked the decision on Markdown (.spec.md) with YAML frontmatter, rejecting the alternative of standalone YAML files. The location is `.lifecycle/features/{feature-name}/spec.md`. The format must capture the full round-trip: user action -> API/function -> processing with **mandatory storage destination** -> response -> user screen update. Each step gets a unique spec ID referenceable in code comments (`// SPEC: e2e-{feature}-{NNN}`), prototype HTML (`data-spec-id`), and verification reports.

**Primary recommendation:** Design a 6-step chain (Screen, Connection, Processing, Storage, Response, Error) rather than the original 5, with Storage as a mandatory explicit step inside Processing. Alternatively, keep 5 steps but make "storage destination" a required field within the Processing step. Both satisfy the user's "storage must be mandatory" constraint -- the choice is Claude's discretion per CONTEXT.md.

<user_constraints>
## User Constraints (from CONTEXT.md)

### Locked Decisions
- Markdown (.spec.md) format -- user readability priority, consistent with GSD patterns
- YAML frontmatter for metadata (ID, feature, status, etc.)
- Location: `.lifecycle/features/{feature-name}/spec.md` -- project-scoped document
- Full round-trip flow: user action -> API/function call -> server/logic processing -> final storage (DB/LLM/file) -> response -> user screen update
- Storage destination is MANDATORY in Processing step
- Spec ID format: `e2e-{feature}-{NNN}` (e.g., e2e-login-001, e2e-chat-002)
- Code comments: `// SPEC: e2e-{feature}-{NNN}` -- similar to WHY+SEE pattern
- DO stage uses spec as step-by-step checklist (Screen done, Connection done, ...)
- TEST stage verifies per-step pass/fail and generates report

### Claude's Discretion
- 5 steps vs 7 steps (+State Change, +Side Effects) -- final decision
- Frontmatter field schema details
- Spec template section structure
- Deviation log format (when DO diverges from spec)

### Deferred Ideas (OUT OF SCOPE)
None -- discussion stayed within phase scope
</user_constraints>

<phase_requirements>
## Phase Requirements

| ID | Description | Research Support |
|----|-------------|-----------------|
| SPEC-01 | Define format for specifying each feature as a Screen->Connection->Processing->Response->Error 5-step chain | Full chain design with Markdown structure, frontmatter schema, step template, ID convention, and realistic example. Research covers step granularity (5 vs 6 vs 7), required fields per step, status lifecycle, and cross-artifact linking. |
</phase_requirements>

## Standard Stack

This phase produces documentation artifacts only -- no code libraries are involved. The "stack" is the file format and tooling conventions.

### Core
| Tool | Version | Purpose | Why Standard |
|------|---------|---------|--------------|
| Markdown (.spec.md) | N/A | Spec document format | User-locked decision; readable in any editor, renders in GitHub/VSCode, consistent with GSD .planning/ patterns |
| YAML frontmatter | N/A | Machine-parseable metadata | Standard in Markdown tooling (Jekyll, Hugo, Obsidian, etc.); parsed by any YAML library |

### Supporting
| Tool | Purpose | When to Use |
|------|---------|-------------|
| `data-spec-id` HTML attribute | Link prototype elements to spec steps | When Phase 3 prototype references spec steps |
| `// SPEC: e2e-{feature}-{NNN}` code comments | Link implementation code to spec steps | During DO stage implementation |

### Alternatives Considered
| Instead of | Could Use | Tradeoff |
|------------|-----------|----------|
| Markdown (.spec.md) | Standalone YAML (.e2e.yaml) | **User rejected.** YAML is more machine-parseable but less human-readable. FEATURES.md research originally recommended YAML, but user chose Markdown for readability and GSD consistency. |
| YAML frontmatter | JSON frontmatter | YAML is more readable for complex nested data; JSON requires more quoting. YAML is the established standard for Markdown frontmatter. |

## Architecture Patterns

### Recommended Project Structure
```
.lifecycle/
├── state.json                    # Current stage, timestamps, mode
├── manifest.json                 # Artifact registry
├── history/                      # Stage transition log
└── features/                     # Per-feature artifacts
    └── {feature-name}/
        ├── spec.md               # E2E feature specification (THIS PHASE)
        ├── prototype.html        # Clickable prototype (Phase 3)
        └── verification.json     # TEST results (Phase 6)
```

### Pattern 1: 5-Step Chain with Mandatory Storage Field

**What:** Each spec step maps to one link in the chain: Screen -> Connection -> Processing -> Response -> Error. The Processing step has a required `storage` field that explicitly names where data is persisted (DB table, LLM API, file path, etc.).

**When to use:** Every feature interaction that involves data persistence (which is nearly all of them, per user's emphasis on round-trip flow).

**Recommendation (Claude's Discretion):** Use 5 steps, not 7. The user emphasized the round-trip flow and mandatory storage -- these are satisfied by making `storage` a required field within Processing. Adding State Change and Side Effects as separate steps (7-step) adds overhead without proportional value for v1. The 7-step expansion is already captured as v2 requirement FMT-02.

**Rationale for 5 over 7:**
- The user's core problem is "done but broken" -- features that miss wiring. 5 steps already cover the critical chain links (Screen, Connection, Processing, Response, Error).
- State Change is implicitly covered: the Response step describes what the user sees after processing, which IS the state change from the user's perspective.
- Side Effects (sending email, triggering webhook) are edge cases better handled as sub-items within Processing than as a mandatory step for every feature.
- Keeping the format lean means lower overhead per spec, which means higher adoption.

**Step definitions:**

| Step | What It Covers | Required Fields | Example |
|------|---------------|-----------------|---------|
| **Screen** | What the user sees/does to trigger the interaction | element, user_action, initial_state | "Login form with email + password inputs" |
| **Connection** | How frontend communicates with backend | method, endpoint/function, request_shape, auth | "POST /api/auth/login with {email, password}" |
| **Processing** | What backend does, including storage | steps[], **storage** (mandatory: destination + operation) | "Validate credentials, query users table, generate JWT" |
| **Response** | What comes back and how UI updates | success_status, response_shape, ui_updates[] | "200 + JWT token, redirect to /dashboard" |
| **Error** | Every failure mode with specific behavior | conditions[] (each with: condition, behavior, status) | "401 -> show error msg; 500 -> show retry toast" |

### Pattern 2: Spec ID as Cross-Artifact Thread

**What:** Every spec step gets a unique ID in the format `e2e-{feature}-{NNN}`. This ID appears in:
1. `spec.md` -- definition (heading + frontmatter)
2. `prototype.html` -- `data-spec-id="e2e-{feature}-{NNN}"` on interactive elements
3. Implementation code -- `// SPEC: e2e-{feature}-{NNN}` comment near implementation
4. `verification.json` -- per-step pass/fail keyed by spec ID

**When to use:** Always. This is the traceability mechanism.

**ID naming convention:**
- `{feature}` = kebab-case feature name (e.g., `login`, `chat-send`, `file-upload`)
- `{NNN}` = zero-padded sequential number within the feature (001, 002, ...)
- Example: `e2e-login-001` (login form display), `e2e-login-002` (login submission)
- Each ID maps to exactly one step in the chain. A feature with 5 chain steps = 5 IDs.

### Pattern 3: Frontmatter Schema

**What:** YAML frontmatter at the top of spec.md contains machine-parseable metadata.

**Recommended schema:**
```yaml
---
feature: {kebab-case-name}        # e.g., user-login
title: {human-readable title}     # e.g., User Login Flow
created: {ISO-8601 date}          # e.g., 2026-03-22
updated: {ISO-8601 date}          # updated on each change
status: draft | agreed | implementing | verified | failed
depends_on: []                    # list of feature names this depends on
steps: {number}                   # total step count
tags: []                          # optional categorization
---
```

**Status lifecycle:**
```
draft ──(user agrees)──> agreed ──(DO starts)──> implementing ──(TEST passes)──> verified
                                                       │
                                                       └──(TEST fails)──> failed ──(DO fixes)──> implementing
```

### Pattern 4: Step Template Structure

**What:** Each step in the spec body follows a consistent Markdown structure.

**Template:**
```markdown
## e2e-{feature}-{NNN}: {Step Title}

**Chain:** {Screen | Connection | Processing | Response | Error}
**Status:** pending | implemented | verified | failed

### What
{Human-readable description of this step}

### Verification Criteria
- [ ] {Concrete, checkable assertion 1}
- [ ] {Concrete, checkable assertion 2}

### Details
{Step-specific structured data -- varies by chain type}
```

**Chain-specific Details sections:**

For **Screen:**
```markdown
### Details
- **Element:** {what the user sees}
- **User Action:** {what the user does}
- **Initial State:** {what the screen looks like before action}
```

For **Connection:**
```markdown
### Details
- **Method:** {HTTP method or function call}
- **Endpoint:** {URL path or function signature}
- **Request:** {body/parameters shape}
- **Auth:** {authentication requirement}
```

For **Processing:**
```markdown
### Details
- **Steps:**
  1. {processing step 1}
  2. {processing step 2}
- **Storage:** {DB table / LLM API / file path} -- {read | write | read+write}
```

For **Response:**
```markdown
### Details
- **Success Status:** {HTTP status or return value}
- **Response Shape:** {what comes back}
- **UI Updates:**
  - {what changes on screen 1}
  - {what changes on screen 2}
```

For **Error:**
```markdown
### Details
| Condition | Behavior | Status |
|-----------|----------|--------|
| {error condition 1} | {what user sees} | {HTTP status or null} |
| {error condition 2} | {what user sees} | {HTTP status or null} |
```

### Pattern 5: Deviation Log Format (Claude's Discretion)

**What:** During DO, if implementation diverges from the spec, the deviation is recorded rather than silently ignored.

**Recommended format:** Add a `## Deviations` section at the bottom of spec.md (not a separate file -- keeps everything co-located).

```markdown
## Deviations

### DEV-001: {Brief description}
- **Spec step:** e2e-{feature}-{NNN}
- **Original:** {what the spec said}
- **Actual:** {what was implemented instead}
- **Reason:** {why the deviation was necessary}
- **Approved:** {yes/no -- user confirmed?}
```

### Anti-Patterns to Avoid

- **Vague assertions:** "Sends data to server" is not checkable. Use "POST /api/auth/login with {email: string, password: string}" -- concrete enough for YES/NO verification.
- **Skipping chain links:** Every feature interaction must have all 5 steps, even if a step is trivial. Explicitly state "Processing: None -- static page render" rather than omitting Processing. Omission is where bugs hide.
- **Spec without round-trip:** The user explicitly requires the full round-trip (user -> storage -> user). A spec that ends at "save to DB" without describing what the user sees afterward is incomplete.
- **Over-granular specs:** Target 5 steps per feature interaction (one per chain link). If a feature has multiple distinct interactions (e.g., "create chat" + "send message" + "delete message"), each interaction gets its own 5-step chain with its own ID range. Do NOT create 20+ steps for a single interaction.

## Don't Hand-Roll

| Problem | Don't Build | Use Instead | Why |
|---------|-------------|-------------|-----|
| YAML frontmatter parsing | Custom regex parser | Standard YAML libraries (js-yaml, PyYAML, gray-matter) | Frontmatter parsing has edge cases (multiline strings, special characters). Standard libraries handle them all. |
| Spec ID generation | Manual numbering | Sequential counter based on existing IDs in the spec file | Manual numbering leads to collisions and gaps. Count existing `e2e-{feature}-*` headings and increment. |
| Markdown rendering | Custom renderer | GitHub/VSCode built-in Markdown preview | The spec format uses standard Markdown. No custom rendering needed. |

**Key insight:** Phase 2 is format definition only. The "Don't Hand-Roll" items become relevant in Phase 4-6 when the format is consumed programmatically.

## Common Pitfalls

### Pitfall 1: Storage Destination Ambiguity
**What goes wrong:** Processing step says "store the data" without specifying WHERE. During DO, the AI picks an arbitrary storage mechanism. During TEST, there is no way to verify the correct storage was used.
**Why it happens:** Storage is "obvious" to the spec writer but not to the implementer (especially an AI).
**How to avoid:** Make `Storage` a required field in Processing with format: `{destination} -- {operation}`. Examples: "PostgreSQL users table -- INSERT", "OpenAI gpt-4 API -- POST", "~/.config/app/settings.json -- WRITE".
**Warning signs:** Processing step lists logic steps but no storage field.

### Pitfall 2: Error Step as Afterthought
**What goes wrong:** The Error step has one or two generic entries ("show error message") instead of covering specific failure modes.
**Why it happens:** Spec writers (both human and AI) focus on the happy path. Error handling is mentally deferred.
**How to avoid:** Require minimum 3 error conditions per interaction: (1) validation/client-side error, (2) server/processing error, (3) infrastructure/network error. Each with specific user-visible behavior.
**Warning signs:** Error step has fewer entries than other steps.

### Pitfall 3: Spec-Prototype ID Mismatch
**What goes wrong:** Spec uses `e2e-login-001` but prototype uses `INT-001` or `FEAT-001` (different ID schemes from earlier research).
**Why it happens:** FEATURES.md research used `FEAT-NNN` and `INT-NNN` conventions. CONTEXT.md locked `e2e-{feature}-{NNN}`. If earlier research is consulted without checking CONTEXT.md, wrong IDs get used.
**How to avoid:** The spec template must use the locked ID format. Previous research ID formats (FEAT-NNN, INT-NNN) are superseded.
**Warning signs:** Any ID in spec or prototype that does not match `e2e-{feature}-{NNN}` pattern.

### Pitfall 4: Confusing "Feature" with "Interaction"
**What goes wrong:** A spec file tries to cover an entire feature (e.g., "Chat System") with dozens of steps, making it unwieldy.
**Why it happens:** Unclear granularity guidance.
**How to avoid:** One spec file per feature, but within the spec, each distinct user interaction (send message, delete message, edit message) gets its own 5-step chain. The `{NNN}` counter is per-feature, so `e2e-chat-001` through `e2e-chat-005` might cover "send message" and `e2e-chat-006` through `e2e-chat-010` might cover "delete message".
**Warning signs:** A single 5-step chain tries to cover multiple distinct user actions.

### Pitfall 5: Status Field Not Updated During DO/TEST
**What goes wrong:** Implementation is complete but spec still shows "Status: pending" on all steps.
**Why it happens:** Updating the spec file during implementation feels like overhead.
**How to avoid:** The DO adapter (Phase 5) must update status as part of each step completion. For Phase 2, document the status lifecycle clearly so the adapter knows what to update.
**Warning signs:** Spec steps stuck at "pending" after DO stage.

## Code Examples

### Example 1: Complete Spec for "User Login" Feature

```markdown
---
feature: user-login
title: User Login Flow
created: 2026-03-22
updated: 2026-03-22
status: draft
depends_on: []
steps: 5
tags: [auth, core]
---

# E2E Spec: User Login Flow

Full round-trip: User enters credentials -> Server validates against DB -> JWT returned -> User sees dashboard.

## e2e-login-001: Login Form Display

**Chain:** Screen
**Status:** pending

### What
User navigates to /login and sees a login form with email input, password input, and submit button.

### Verification Criteria
- [ ] Login form renders at /login route
- [ ] Email input exists with type="email" and placeholder
- [ ] Password input exists with type="password"
- [ ] Submit button exists and is initially enabled

### Details
- **Element:** Login form with 2 inputs + 1 button
- **User Action:** Navigate to /login
- **Initial State:** Empty form, no error messages visible

## e2e-login-002: Login Submission

**Chain:** Connection
**Status:** pending

### What
User fills email + password and clicks submit. Frontend sends POST request to auth endpoint.

### Verification Criteria
- [ ] Submit triggers POST /api/auth/login
- [ ] Request body contains {email: string, password: string}
- [ ] Authorization header is not required (login is pre-auth)
- [ ] Loading state shown while request is in-flight

### Details
- **Method:** POST
- **Endpoint:** /api/auth/login
- **Request:** `{ email: string, password: string }`
- **Auth:** None (pre-authentication endpoint)

## e2e-login-003: Credential Validation

**Chain:** Processing
**Status:** pending

### What
Server receives login request, validates email exists in database, compares password hash, generates JWT on success.

### Verification Criteria
- [ ] Email lookup queries the users table
- [ ] Password comparison uses bcrypt (not plain text)
- [ ] JWT is generated with user ID payload
- [ ] JWT expiration is set (not infinite)

### Details
- **Steps:**
  1. Validate request body (email format, password non-empty)
  2. Query users table by email
  3. Compare submitted password with stored hash (bcrypt)
  4. Generate JWT with {userId, exp} payload
- **Storage:** PostgreSQL `users` table -- READ (SELECT by email)

## e2e-login-004: Successful Login Response

**Chain:** Response
**Status:** pending

### What
Server returns JWT token. Frontend stores token and redirects user to dashboard.

### Verification Criteria
- [ ] Response status is 200
- [ ] Response body contains {token: string, user: {id, email, name}}
- [ ] Token is stored in localStorage or httpOnly cookie
- [ ] Browser redirects to /dashboard
- [ ] Dashboard shows user's name

### Details
- **Success Status:** 200 OK
- **Response Shape:** `{ token: string, user: { id: string, email: string, name: string } }`
- **UI Updates:**
  - JWT token stored in client (localStorage or cookie)
  - Browser navigates to /dashboard
  - Dashboard header shows "Welcome, {name}"

## e2e-login-005: Login Error Handling

**Chain:** Error
**Status:** pending

### What
All failure modes for the login flow, with specific user-visible behavior for each.

### Verification Criteria
- [ ] Empty email/password shows inline validation error (no API call)
- [ ] Invalid credentials show "Invalid email or password" error message
- [ ] Server error shows "Something went wrong. Please try again." toast
- [ ] Network timeout shows "Connection failed. Check your internet." message
- [ ] Rate limit (429) shows "Too many attempts. Try again in X minutes."

### Details
| Condition | Behavior | Status |
|-----------|----------|--------|
| Empty email or password | Inline validation: "Email is required" / "Password is required" | null (client-side) |
| Invalid credentials | Error message below form: "Invalid email or password". Clear password field. | 401 |
| Server error | Toast notification: "Something went wrong. Please try again." | 500 |
| Network timeout | Error message: "Connection failed. Check your internet." | null (network) |
| Rate limited | Error message: "Too many attempts. Try again in {minutes} minutes." | 429 |
```

### Example 2: Spec ID in Code Comments

```typescript
// SPEC: e2e-login-001 -- Login Form Display (Screen)
export function LoginForm() {
  return (
    <form onSubmit={handleSubmit}> {/* SPEC: e2e-login-002 */}
      <input type="email" name="email" placeholder="Email" />
      <input type="password" name="password" />
      <button type="submit">Log In</button>
      {error && <p className="error">{error}</p>} {/* SPEC: e2e-login-005 */}
    </form>
  );
}

// SPEC: e2e-login-003 -- Credential Validation (Processing)
// Storage: PostgreSQL users table -- READ
export async function validateCredentials(email: string, password: string) {
  const user = await db.users.findByEmail(email);
  if (!user || !await bcrypt.compare(password, user.passwordHash)) {
    throw new AuthError('Invalid credentials');
  }
  return user;
}
```

### Example 3: Frontmatter for Manifest Registration

When spec.md is created, it is registered in `.lifecycle/manifest.json`:
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

## State of the Art

| Old Approach | Current Approach | When Changed | Impact |
|--------------|------------------|--------------|--------|
| YAML-based spec (.e2e.yaml) | Markdown spec (.spec.md) with YAML frontmatter | CONTEXT.md decision (2026-03-22) | Human readability prioritized; renders natively in GitHub/VSCode |
| FEAT-NNN / INT-NNN IDs | e2e-{feature}-{NNN} IDs | CONTEXT.md decision (2026-03-22) | Feature name in ID gives immediate context without lookup |
| Spec in .planning/specs/ | Spec in .lifecycle/features/{name}/ | Phase 1 + CONTEXT.md decisions | Co-located with prototype and verification artifacts |
| 5-layer without storage emphasis | 5-layer with mandatory storage field | CONTEXT.md decision (2026-03-22) | Forces explicit storage destination, prevents "processed but not saved" bugs |

**Deprecated/outdated:**
- `.planning/specs/{feature}.e2e.yaml` path from FEATURES.md research -- replaced by `.lifecycle/features/{feature-name}/spec.md`
- `FEAT-NNN` ID format from FEATURES.md research -- replaced by `e2e-{feature}-{NNN}`
- Standalone YAML format -- replaced by Markdown with frontmatter

## Open Questions

1. **Step numbering within multi-interaction features**
   - What we know: A feature like "Chat" has multiple interactions (send, delete, edit). Each interaction is a 5-step chain.
   - What's unclear: Should `{NNN}` be sequential across all interactions (e2e-chat-001 to e2e-chat-015) or reset per interaction (requires sub-ID)?
   - Recommendation: Sequential across all interactions within a feature. Simpler, no sub-ID complexity. Grouping is implicit from the heading structure.

2. **Spec template distribution**
   - What we know: The spec template needs to be accessible when PLAN creates a new spec.
   - What's unclear: Should the template live in `~/.claude/skills/dev-lifecycle/templates/spec-template.md` or as inline guidance in a reference file?
   - Recommendation: Template file in `templates/spec-template.md`. Reference file `references/e2e-spec.md` explains the format and points to the template. Consistent with existing `templates/state.json` and `templates/manifest.json` pattern.

3. **Maximum spec steps per feature**
   - What we know: Each interaction = 5 steps. A complex feature might have 5+ interactions = 25+ steps.
   - What's unclear: At what point should a feature be split into sub-features?
   - Recommendation: Soft limit of 25 steps (5 interactions). If exceeded, consider splitting into sub-features (e.g., "chat-messaging" and "chat-management").

## Validation Architecture

### Test Framework
| Property | Value |
|----------|-------|
| Framework | Manual validation (format definition phase -- no runtime code) |
| Config file | none |
| Quick run command | Visual inspection of generated spec.md against template |
| Full suite command | N/A |

### Phase Requirements -> Test Map
| Req ID | Behavior | Test Type | Automated Command | File Exists? |
|--------|----------|-----------|-------------------|-------------|
| SPEC-01 | 5-step chain format defined with realistic example | manual-only | Visual verification: spec has 5 chain steps, each with required fields, storage is mandatory, IDs follow convention | N/A |

**Justification for manual-only:** Phase 2 produces a Markdown template and example document. There is no runtime code to test. Validation = "does the template have all required sections and does the example follow the template."

### Sampling Rate
- **Per task commit:** Visual inspection of created files
- **Per wave merge:** Review spec template against CONTEXT.md decisions
- **Phase gate:** Template exists + example follows template + all CONTEXT.md constraints satisfied

### Wave 0 Gaps
None -- no test infrastructure needed for a format definition phase.

## Sources

### Primary (HIGH confidence)
- `.planning/phases/02-e2e-spec-format/02-CONTEXT.md` -- User's locked decisions on format, ID scheme, location, and usage pattern
- `.planning/research/FEATURES.md` -- Prior research on E2E spec format, 5-layer chain design, YAML vs Markdown analysis, Gherkin rejection rationale
- `.planning/research/ARCHITECTURE.md` -- Spec-prototype integration architecture, `.lifecycle/features/` structure, data flow diagrams
- `~/.claude/skills/dev-lifecycle/SKILL.md` -- Current skill structure, stage descriptions, state management
- `~/.claude/skills/dev-lifecycle/templates/manifest.json` -- Artifact registration pattern
- `~/.claude/skills/dev-lifecycle/references/stage-transitions.md` -- Gate rules, status values, artifact output format

### Secondary (MEDIUM confidence)
- `.planning/PROJECT.md` -- Core problem statement ("100% failure rate on feature completeness")
- `.planning/REQUIREMENTS.md` -- SPEC-01 definition and traceability matrix

### Tertiary (LOW confidence)
- None -- all findings based on project-internal documents and locked decisions

## Metadata

**Confidence breakdown:**
- Standard stack: HIGH -- User locked all format decisions (Markdown, frontmatter, location, ID scheme)
- Architecture: HIGH -- Project structure and spec template design follow directly from CONTEXT.md decisions + Phase 1 foundation
- Pitfalls: HIGH -- Based on prior research (FEATURES.md) analysis of common spec-writing mistakes and the user's explicit emphasis on mandatory storage

**Research date:** 2026-03-22
**Valid until:** 2026-04-22 (stable -- format definition unlikely to change rapidly)
