# Architecture Research

**Domain:** Claude Code skill-based development lifecycle orchestrator
**Researched:** 2026-03-22
**Confidence:** HIGH (based on direct analysis of existing skill implementations + PROJECT.md constraints)

**Focus:** How to architect the pipeline so that User Interaction Prototype and E2E Spec are integral parts of PLAN/DO/TEST stages, not separate components.

## System Overview

```
┌─────────────────────────────────────────────────────────────────────┐
│                     PLAN STAGE (produces 2 core artifacts)          │
│                                                                     │
│  ┌──────────────────────────────┐  ┌──────────────────────────────┐ │
│  │     E2E Feature Spec         │  │  User Interaction Prototype  │ │
│  │     (.spec.md)               │  │  (.prototype.html)           │ │
│  │                              │  │                              │ │
│  │  Screen ──────────────── ◄───┼──┤  [clickable HTML page]       │ │
│  │  Connection ─────────── ◄───┼──┤  [simulated API calls]       │ │
│  │  Processing ────────────    │  │  [state transitions]         │ │
│  │  Response ──────────── ◄───┼──┤  [success/error displays]    │ │
│  │  Error ────────────── ◄───┼──┤  [error state mockups]       │ │
│  │                              │  │                              │ │
│  │  Each step has:              │  │  Each interaction has:       │ │
│  │  - id (e2e-001)              │  │  - data-spec-id="e2e-001"   │ │
│  │  - verification criteria     │  │  - visual representation    │ │
│  │  - status: pending           │  │  - user can click through   │ │
│  └──────────────┬───────────────┘  └──────────────┬───────────────┘ │
│                 │  LINKED BY spec-id               │                │
├─────────────────┴──────────────────────────────────┴────────────────┤
│                     DO STAGE (consumes spec as checklist)            │
│                                                                     │
│  For each spec step (e2e-001, e2e-002, ...):                       │
│  ┌──────────────────────────────────────────────────────────────┐   │
│  │  [ ] Screen done?      — UI element exists, renders          │   │
│  │  [ ] Connection done?   — Event handler wired, API called    │   │
│  │  [ ] Processing done?   — Backend logic implemented          │   │
│  │  [ ] Response done?     — Success path returns correct data  │   │
│  │  [ ] Error done?        — Error cases handled, UI shows msg  │   │
│  └──────────────────────────────────────────────────────────────┘   │
│                                                                     │
│  DO updates spec status: pending -> implemented                     │
│  DO CANNOT advance to TEST until all steps = implemented            │
├─────────────────────────────────────────────────────────────────────┤
│                     TEST STAGE (verifies spec + diffs prototype)     │
│                                                                     │
│  ┌──────────────────────────┐  ┌──────────────────────────────────┐ │
│  │  Spec Verification       │  │  Prototype Comparison            │ │
│  │                          │  │                                  │ │
│  │  For each e2e-NNN:       │  │  For each data-spec-id:         │ │
│  │  - Screen: exists? ✓/✗   │  │  - Prototype has element? ✓     │ │
│  │  - Connection: wired? ✓/✗│  │  - Implementation has it? ✓/✗   │ │
│  │  - Processing: works? ✓/✗│  │  - Behavior matches? ✓/✗       │ │
│  │  - Response: correct? ✓/✗│  │  - States match? ✓/✗           │ │
│  │  - Error: handled? ✓/✗   │  │                                  │ │
│  │                          │  │  STRUCTURAL DIFF:                │ │
│  │  Status: implemented ->  │  │  - Missing elements             │ │
│  │          verified/failed │  │  - Extra elements               │ │
│  └──────────────────────────┘  └──────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────────────┘
```

### Component Responsibilities

| Component | Responsibility | Implementation |
|-----------|----------------|----------------|
| E2E Spec Generator | Produce structured feature spec with 5-step chain per interaction | Markdown file with YAML frontmatter; each step has unique ID, verification criteria, status field |
| Prototype Generator | Produce clickable HTML+JS mockup that visualizes the spec | Single HTML file with embedded JS; each interactive element tagged with `data-spec-id` linking to spec |
| Spec Checklist Engine | Track implementation progress per spec step during DO | Read spec file, present as checklist, update status field from `pending` to `implemented` |
| Prototype Differ | Compare prototype structure against actual implementation at TEST | Parse prototype HTML for `data-spec-id` elements, verify corresponding elements exist in implementation |
| Verification Gate | Block TEST->COMMIT transition until all spec steps = verified | Read spec, count verified vs failed, block if any failed |

## Recommended Project Structure

```
.lifecycle/                              # Orchestrator state (separate from .planning/)
├── state.json                           # Current stage, timestamps, mode
├── manifest.json                        # Artifact registry: stage -> [file paths]
├── history/                             # Stage transition log
│   └── YYYY-MM-DD-HH-MM.json
└── features/                            # Per-feature artifacts (spec + prototype together)
    └── {feature-name}/
        ├── spec.md                      # E2E feature specification
        ├── prototype.html               # Clickable interaction prototype
        └── verification.json            # TEST stage results (per spec step)
```

### Structure Rationale

- **`features/{name}/` groups spec + prototype together:** They are two views of the same thing. The spec is the machine-readable contract; the prototype is the human-clickable demo. Separating them into `specs/` and `prototypes/` creates artificial distance -- a change to one always requires checking the other.
- **`verification.json` lives with the feature:** TEST results are specific to the feature, not to the stage. When a feature is re-tested, the previous verification results are overwritten, not accumulated. Historical results live in `history/`.
- **No separate `prototypes/` and `specs/` directories:** The previous architecture had `prototypes/` and `specs/` as siblings. This creates a coordination problem: which spec goes with which prototype? Grouping by feature eliminates this.

## Architectural Patterns

### Pattern 1: Dual-Artifact PLAN Output (Spec + Prototype as Inseparable Pair)

**What:** PLAN stage ALWAYS produces two artifacts for every feature: an E2E spec (.spec.md) and a clickable prototype (.prototype.html). Neither is optional. Neither is produced without the other. They share a common identifier namespace (`e2e-NNN`).

**When to use:** Every time PLAN runs for a feature (not for hotfixes or config changes).

**Trade-offs:**
- PRO: User validates both the structural contract (spec) AND the interactive experience (prototype) before DO begins. Eliminates "done but broken" at the root.
- PRO: Machine (TEST) verifies the spec; Human (user) verifies the prototype. Both verification paths covered.
- CON: PLAN takes longer. Generating a clickable prototype is more work than writing a spec.
- MITIGATION: The prototype is wireframe-level, not pixel-perfect. Interaction flow, not visual polish. Claude generates adequate HTML+JS for flow validation.

**Spec format:**

```markdown
---
feature: user-authentication
created: 2026-03-22
status: draft | agreed | implementing | verified
steps: 5
---

# E2E Spec: User Authentication

## e2e-001: Login Form Display
- **Screen:** Login form with email input, password input, submit button
- **Verification:** Form renders with all 3 elements; email input has type="email"; password input has type="password"
- **Status:** pending
- **Prototype ref:** data-spec-id="e2e-001"

## e2e-002: Login Submission
- **Connection:** Submit button click -> POST /api/auth/login with {email, password}
- **Verification:** Network request fires with correct method, URL, and body shape
- **Status:** pending
- **Prototype ref:** data-spec-id="e2e-002"

## e2e-003: Credential Validation
- **Processing:** Server validates email exists, password matches hash
- **Verification:** Valid credentials return 200; invalid return 401 with error body
- **Status:** pending
- **Prototype ref:** data-spec-id="e2e-003"

## e2e-004: Successful Login Response
- **Response:** 200 returns JWT token; UI redirects to /dashboard; token stored
- **Verification:** Token present in response; redirect occurs; token in localStorage
- **Status:** pending
- **Prototype ref:** data-spec-id="e2e-004"

## e2e-005: Login Error Display
- **Error:** 401 -> show "Invalid email or password" below form; clear password; focus email
- **Verification:** Error message visible; password field empty; email field focused
- **Status:** pending
- **Prototype ref:** data-spec-id="e2e-005"
```

**Prototype format:**

```html
<!DOCTYPE html>
<html>
<head>
  <meta charset="utf-8">
  <title>Prototype: User Authentication</title>
  <meta name="spec-file" content="spec.md">
  <meta name="feature" content="user-authentication">
  <style>
    /* Minimal wireframe styling -- NOT design */
    * { font-family: system-ui; box-sizing: border-box; }
    .prototype-frame { max-width: 400px; margin: 40px auto; padding: 20px; border: 2px solid #333; }
    .spec-indicator { font-size: 10px; color: #999; position: absolute; top: -12px; left: 0; }
    [data-spec-id] { position: relative; outline: 1px dashed #ccc; margin: 4px 0; }
    [data-spec-id]:hover { outline: 2px solid #0066cc; }
    [data-spec-id]:hover .spec-indicator { color: #0066cc; font-weight: bold; }
    .error-msg { color: red; display: none; }
    .hidden { display: none; }
  </style>
</head>
<body>
  <div class="prototype-frame">
    <h2>Login</h2>

    <!-- e2e-001: Login Form Display -->
    <div data-spec-id="e2e-001" data-spec-step="Screen">
      <span class="spec-indicator">e2e-001: Screen</span>
      <input id="email" type="email" placeholder="Email" />
      <input id="password" type="password" placeholder="Password" />
      <button id="submit" onclick="handleLogin()">Log In</button>
    </div>

    <!-- e2e-005: Error Display -->
    <div data-spec-id="e2e-005" data-spec-step="Error">
      <span class="spec-indicator">e2e-005: Error</span>
      <p id="error-msg" class="error-msg">Invalid email or password</p>
    </div>
  </div>

  <!-- e2e-004: Success State (initially hidden) -->
  <div id="dashboard" class="hidden prototype-frame" data-spec-id="e2e-004" data-spec-step="Response">
    <span class="spec-indicator">e2e-004: Response</span>
    <h2>Dashboard</h2>
    <p>Welcome! You are logged in.</p>
  </div>

  <script>
    // Prototype simulation -- NOT real implementation
    // Each function simulates one spec step
    function handleLogin() { // e2e-002: Connection
      const email = document.getElementById('email').value;
      const password = document.getElementById('password').value;

      // e2e-003: Processing (simulated)
      if (email === 'test@example.com' && password === 'password') {
        // e2e-004: Response
        document.querySelector('.prototype-frame').classList.add('hidden');
        document.getElementById('dashboard').classList.remove('hidden');
      } else {
        // e2e-005: Error
        document.getElementById('error-msg').style.display = 'block';
        document.getElementById('password').value = '';
        document.getElementById('email').focus();
      }
    }
  </script>

  <!-- Spec coverage footer (machine-parseable) -->
  <script type="application/json" id="spec-coverage">
  {
    "spec_file": "spec.md",
    "feature": "user-authentication",
    "covered_steps": ["e2e-001", "e2e-002", "e2e-003", "e2e-004", "e2e-005"],
    "interaction_map": {
      "e2e-001": {"element": "[data-spec-id='e2e-001']", "action": "render", "type": "Screen"},
      "e2e-002": {"element": "#submit", "action": "click", "type": "Connection"},
      "e2e-003": {"element": null, "action": "validate", "type": "Processing"},
      "e2e-004": {"element": "#dashboard", "action": "show", "type": "Response"},
      "e2e-005": {"element": "#error-msg", "action": "show", "type": "Error"}
    }
  }
  </script>
</body>
</html>
```

**Key design decisions in prototype format:**
1. `data-spec-id` on every interactive element links it to the spec.
2. `data-spec-step` indicates which chain step (Screen/Connection/Processing/Response/Error) the element represents.
3. Hover shows `spec-indicator` labels so the user can see which spec step they are testing.
4. Embedded `<script type="application/json" id="spec-coverage">` provides machine-parseable coverage data for TEST stage.
5. Styling is deliberately wireframe-level. No design system, no colors, no polish.

### Pattern 2: Spec-as-Checklist in DO Stage

**What:** During DO, the spec is treated as an implementation checklist. Each spec step's status transitions from `pending` -> `implemented`. DO reads the spec at entry, presents it as a task list, and updates status after implementing each step.

**When to use:** Every DO stage for a feature.

**Trade-offs:**
- PRO: Implementation completeness is tracked per spec step, not per file. "Login form component created" is not enough -- "e2e-001 Screen implemented, e2e-002 Connection implemented" is required.
- PRO: Partially completed features have precise progress tracking. If compaction or session end interrupts DO, the next session knows exactly which steps remain.
- CON: Adds overhead to DO stage (must update spec after each step).
- MITIGATION: Update is trivial -- change `Status: pending` to `Status: implemented` in the spec file. One line edit per step.

**DO stage flow:**

```
DO STAGE ENTRY:
  1. Read spec.md from .lifecycle/features/{feature}/
  2. Read prototype.html for visual reference
  3. Present spec steps as implementation checklist:
     [ ] e2e-001: Login Form Display (Screen)
     [ ] e2e-002: Login Submission (Connection)
     [ ] e2e-003: Credential Validation (Processing)
     [ ] e2e-004: Successful Login Response (Response)
     [ ] e2e-005: Login Error Display (Error)
  4. For each step:
     a. Implement the code
     b. Update spec.md: Status: pending -> implemented
     c. Move to next step
  5. When all steps = implemented: suggest transition to TEST
```

**Critical constraint:** DO must NOT skip spec steps. If a step seems unnecessary, the right action is to go back to PLAN and remove it from the spec (with user agreement), not to silently skip it. This prevents the "connection exists but wiring is missing" class of bugs.

### Pattern 3: Structural Diff at TEST Stage (Prototype vs Implementation)

**What:** TEST stage performs two verification passes. Pass 1: verify each spec step against the actual code (programmatic). Pass 2: compare prototype structure against implementation structure (structural diff).

**When to use:** Every TEST stage for a feature.

**Trade-offs:**
- PRO: Catches the "prototype has 5 buttons, implementation has 3" class of bugs.
- PRO: Machine-parseable prototype metadata (`spec-coverage` JSON) enables automated structural comparison.
- CON: Structural diff is approximate, not pixel-perfect. The prototype is HTML; the implementation might be React/Vue/Flutter. Direct DOM comparison is not possible for non-web projects.
- MITIGATION: Structural diff compares CONCEPTS not DOM. "Does the implementation have a login form with email, password, and submit?" -- not "Does the implementation have an `<input type='email'>`."

**TEST stage flow:**

```
TEST STAGE ENTRY:
  1. Read spec.md
  2. Read prototype.html (parse spec-coverage JSON)
  3. Read implementation source code

  PASS 1 — Spec Verification (per step):
  For each spec step (e2e-NNN):
    Check verification criteria against implementation:
    - Screen: Does the UI element exist in the codebase?
    - Connection: Is there a handler that calls the specified endpoint?
    - Processing: Does backend logic exist for the endpoint?
    - Response: Does the success path return the specified data?
    - Error: Is the error case handled with the specified behavior?
    Update spec: Status: implemented -> verified OR implemented -> failed
    If failed: record WHY in verification.json

  PASS 2 — Prototype Structural Diff:
  For each data-spec-id in prototype:
    Check: Does implementation have a corresponding element/component?
    Check: Does the element handle the specified action (click, render, show)?
    Check: Does the interaction produce the expected state change?
    Record: { spec_id, prototype_has, implementation_has, match: true/false }

  OUTPUT:
  - verification.json with per-step results
  - Updated spec.md with verified/failed status
  - Gap list: spec steps that failed or prototype elements missing from implementation
  - If any gaps: BLOCK transition to COMMIT. Return to DO with specific gap list.
```

**verification.json format:**

```json
{
  "feature": "user-authentication",
  "tested_at": "2026-03-22T16:00:00",
  "spec_verification": {
    "e2e-001": {
      "status": "verified",
      "screen": { "pass": true, "evidence": "LoginForm.tsx renders email, password, submit" },
      "connection": null,
      "processing": null,
      "response": null,
      "error": null
    },
    "e2e-002": {
      "status": "failed",
      "screen": null,
      "connection": { "pass": false, "evidence": "onClick handler exists but calls console.log, not API", "gap": "Wiring incomplete: submit button -> API call missing" },
      "processing": null,
      "response": null,
      "error": null
    }
  },
  "prototype_diff": {
    "e2e-001": { "prototype": true, "implementation": true, "match": true },
    "e2e-002": { "prototype": true, "implementation": false, "match": false, "detail": "Prototype simulates API call; implementation has stub handler" },
    "e2e-005": { "prototype": true, "implementation": false, "match": false, "detail": "Prototype shows error message; implementation has no error handling" }
  },
  "summary": {
    "total_steps": 5,
    "verified": 1,
    "failed": 2,
    "pending": 2,
    "gate": "BLOCKED"
  }
}
```

### Pattern 4: Spec ID as Cross-Stage Thread

**What:** The `e2e-NNN` identifier is the primary key that connects PLAN, DO, and TEST artifacts. Every reference to a feature interaction uses this ID. The ID appears in: spec.md (definition), prototype.html (visualization), implementation code (comments), verification.json (results).

**When to use:** Always. This is the mechanism that prevents artifact fragmentation (Pitfall 6 from PITFALLS.md).

**Trade-offs:**
- PRO: Traceability. From any artifact, you can trace forward and backward: "e2e-003 was defined in spec, visualized in prototype, implemented in auth.service.ts:45, and verified in verification.json."
- PRO: Gap detection. If spec has e2e-005 but implementation has no comment or code referencing e2e-005, it is missing.
- CON: Requires discipline to add `// e2e-003` comments in code.
- MITIGATION: DO stage instructions include "add spec ID comment near implementation." TEST stage checks for presence of spec ID comments as part of wiring verification.

**Example in implementation code:**

```typescript
// e2e-001: Login Form Display (Screen)
export function LoginForm() {
  return (
    <form onSubmit={handleSubmit}> {/* e2e-002: Connection */}
      <input type="email" name="email" />
      <input type="password" name="password" />
      <button type="submit">Log In</button>
      {error && <p className="error">{error}</p>} {/* e2e-005: Error */}
    </form>
  );
}

// e2e-003: Credential Validation (Processing)
export async function validateCredentials(email: string, password: string) {
  const user = await db.users.findByEmail(email);
  if (!user || !await bcrypt.compare(password, user.passwordHash)) {
    throw new AuthError('Invalid credentials'); // e2e-005: Error source
  }
  return user;
}

// e2e-004: Successful Login Response
export async function loginHandler(req: Request, res: Response) {
  const { email, password } = req.body;
  const user = await validateCredentials(email, password); // e2e-003
  const token = jwt.sign({ userId: user.id }, SECRET);
  res.json({ token }); // e2e-004: Response
}
```

## Data Flow

### Artifact Flow: PLAN -> DO -> TEST (The Inner Loop)

```
PLAN STAGE
    │
    ├── Generates: spec.md (E2E feature specification)
    │   - 5-step chain per feature interaction
    │   - Each step has unique e2e-NNN ID
    │   - All steps start as Status: pending
    │
    ├── Generates: prototype.html (clickable mockup)
    │   - Each interactive element has data-spec-id="e2e-NNN"
    │   - Embedded spec-coverage JSON for machine parsing
    │   - User clicks through to validate flow
    │
    ├── User validates prototype + spec
    │   - Clicks through prototype: "Yes, this is what I want"
    │   - Reviews spec: "Yes, these are the success/error criteria"
    │   - spec.md status: draft -> agreed
    │
    ▼
DO STAGE
    │
    ├── Reads: spec.md (as implementation checklist)
    ├── Reads: prototype.html (as interaction reference)
    │
    ├── For each e2e-NNN step:
    │   ├── Implement code for this step
    │   ├── Add // e2e-NNN comment near implementation
    │   └── Update spec.md: Status: pending -> implemented
    │
    ├── All steps implemented?
    │   ├── NO: Continue implementing
    │   └── YES: Suggest TEST transition
    │
    ▼
TEST STAGE
    │
    ├── Reads: spec.md (verification criteria)
    ├── Reads: prototype.html (structural reference)
    ├── Reads: implementation source code
    │
    ├── Pass 1: Spec Verification
    │   - For each e2e-NNN: check criteria against code
    │   - Update: Status: implemented -> verified/failed
    │
    ├── Pass 2: Prototype Structural Diff
    │   - For each data-spec-id in prototype:
    │     compare against implementation structure
    │
    ├── Writes: verification.json (results)
    │
    ├── All steps verified?
    │   ├── NO: Return to DO with gap list (DO-TEST loop)
    │   └── YES: Allow COMMIT transition
    │
    ▼
COMMIT STAGE (gated by verification)
```

### The DO-TEST Rework Loop

```
DO produces implementation
    │
    ▼
TEST finds gaps (e.g., e2e-002 failed: handler is a stub)
    │
    ▼
Gap list generated:
  - e2e-002: Connection stub — onClick calls console.log, not POST /api/auth/login
  - e2e-005: Error handling missing — no catch block, no error display
    │
    ▼
Return to DO with SPECIFIC gaps (not "fix everything")
    │
    ▼
DO fixes only the gaps
    │
    ├── Fix e2e-002: Replace console.log with fetch() call
    ├── Fix e2e-005: Add try/catch, display error message
    ├── Update spec.md: Status for fixed steps -> implemented
    │
    ▼
TEST re-verifies (only failed steps + regression on verified steps)
    │
    ├── All verified? YES -> COMMIT
    └── Still gaps? -> Back to DO (max 3 loops before escalating to user)
```

### State Management: Spec Status Lifecycle

```
PLAN creates spec step
    │
    ▼
pending ──── (DO implements) ──── implemented ──── (TEST verifies) ──── verified
                                       │                                   │
                                       │                                   ▼
                                       │                              COMMIT gate opens
                                       │
                                       └── (TEST finds gap) ──── failed
                                                                   │
                                                                   ▼
                                                              return to DO
                                                                   │
                                                                   ▼
                                                              implemented (again)
                                                                   │
                                                                   ▼
                                                              TEST re-verifies
```

### Key Data Flows

1. **Spec-driven implementation:** PLAN creates the spec with all steps `pending`. DO reads the spec and treats it as a checklist, advancing each step to `implemented`. This flow ensures no spec step is forgotten -- the checklist is exhaustive by definition.

2. **Prototype-driven verification:** TEST reads the prototype's `spec-coverage` JSON to know what interactions to check. For each `data-spec-id` in the prototype, TEST verifies a corresponding element/behavior exists in the implementation. This flow catches "prototype showed 5 buttons, implementation has 3."

3. **Gap-driven rework:** TEST produces a specific gap list (not "test failed" but "e2e-002 Connection: handler is a stub"). DO re-enters with the gap list as its task scope, not the full spec. This flow prevents full-rework when only 1-2 steps failed.

## Scaling Considerations

| Scale | Architecture Adjustments |
|-------|--------------------------|
| 1 feature, simple | Single spec.md with 3-5 steps. Prototype is one HTML page. Verification is straightforward. |
| 1 feature, complex (e.g., multi-step wizard) | Spec has 15-20 steps grouped by sub-feature. Prototype has multiple "pages" (divs shown/hidden). Consider splitting into sub-features if > 20 steps. |
| Multiple features in parallel | Each feature gets its own `.lifecycle/features/{name}/` directory. Specs are independent. Prototypes are independent. No cross-feature coupling in the inner loop. |
| Non-web project (API-only, CLI, mobile) | Prototype adapts: API-only gets a "request/response simulator" HTML page instead of a UI mockup. CLI gets a "terminal session simulator." Mobile gets a responsive wireframe. The spec format (5-step chain) is universal. |

### Scaling Priorities

1. **First bottleneck: Prototype generation time.** Claude takes longer to generate well-structured HTML prototypes than to write specs. MITIGATION: Keep prototypes wireframe-level. No styling beyond layout. Use a standard template that Claude fills in (reduce generation variance).

2. **Second bottleneck: Spec step granularity.** Too fine-grained = 50 steps for a simple form (overhead). Too coarse = "implement login" with 1 step (useless). MITIGATION: Target 3-8 steps per feature interaction. One step per chain link (Screen, Connection, Processing, Response, Error) is the natural grain.

## Anti-Patterns

### Anti-Pattern 1: Prototype as Design Deliverable

**What people do:** Spend time making the prototype look polished with proper CSS, responsive layout, brand colors.
**Why it's wrong:** The prototype validates INTERACTION FLOW, not visual design. Polishing wastes time and sets wrong expectations ("I thought it would look exactly like this"). When the prototype is pretty, users give feedback on colors instead of flows.
**Do this instead:** Deliberately ugly wireframe. Borders and system fonts. No gradients, no rounded corners, no brand colors. The visual contract lives separately in UI-SPEC.md (if needed).

### Anti-Pattern 2: Prototype Without Spec (or Vice Versa)

**What people do:** Generate just a prototype ("let me show you a mockup") or just a spec ("here are the requirements") but not both together.
**Why it's wrong:** A prototype without spec has no verification criteria -- TEST cannot check it systematically. A spec without prototype has no user validation -- the user cannot click to confirm. The pair is inseparable because they serve different audiences: spec serves the machine (TEST), prototype serves the human (user).
**Do this instead:** PLAN always produces both. The SKILL.md instruction must say "PLAN is not complete until both spec.md AND prototype.html exist with matching spec IDs."

### Anti-Pattern 3: Spec Steps That Skip Chain Links

**What people do:** Write a spec step like "e2e-002: User can log in" that jumps from Screen to Response without specifying Connection or Processing.
**Why it's wrong:** The skipped links are where bugs hide. "User can log in" is true if the form submits and the response is OK -- but it does not verify that the handler calls the right API, that the API validates credentials, or that errors are caught. Every skipped link is a gap that will manifest as "done but broken."
**Do this instead:** Every feature interaction has ALL 5 chain links specified. Some may be trivial ("Processing: none, static page") but they must be explicit. Explicit "none" is better than implicit "assumed."

### Anti-Pattern 4: TEST Stage as Manual Review Only

**What people do:** TEST stage reads the spec and says "I've reviewed the code and it looks correct."
**Why it's wrong:** Claude's review is subject to the same existence bias that causes the "done but broken" problem (Pitfall 1). Claude sees a function called `handleLogin` and concludes login works, without checking whether `handleLogin` actually calls the API.
**Do this instead:** TEST must perform PROGRAMMATIC checks for each chain link. Screen: grep for the UI element. Connection: grep for the API call with the correct URL. Processing: grep for the validation logic. Response: grep for the response format. Error: grep for the error handler. Automated checks, not narrative review.

### Anti-Pattern 5: Treating Spec IDs as Optional Decoration

**What people do:** Generate a spec with IDs but do not reference them in prototype, do not comment them in code, do not check them in TEST.
**Why it's wrong:** Without consistent ID usage, the cross-stage thread breaks. The spec says e2e-003 exists. The prototype shows something that might be e2e-003. The code has a function that might implement e2e-003. But there is no linkage, so TEST cannot trace the chain.
**Do this instead:** ENFORCE spec ID presence at every stage. PLAN: spec has IDs, prototype has `data-spec-id`. DO: code has `// e2e-NNN` comments. TEST: verification.json keys are spec IDs. Missing an ID at any stage = incomplete artifact.

## Integration Points

### How Prototype Serves Dual Purpose (Demo AND Verification Target)

The prototype HTML file is designed to serve two audiences:

| Audience | How They Use It | What They See |
|----------|-----------------|---------------|
| **User (demo)** | Opens in browser, clicks through interactions | Wireframe UI with clickable elements, state changes, error displays |
| **TEST stage (verification)** | Parses `spec-coverage` JSON, reads `data-spec-id` attributes | Machine-readable map of what elements exist, what actions they perform, what spec steps they cover |

This dual-use is achieved through:
1. **Visual layer:** Standard HTML that renders in any browser. User sees forms, buttons, transitions.
2. **Metadata layer:** `data-spec-id` attributes and `spec-coverage` JSON that TEST parses. User ignores these; they are invisible in the browser.
3. **Hover labels:** `spec-indicator` CSS shows spec IDs on hover. Users can see the connection if curious; labels do not clutter the default view.

### How E2E Spec Adapts to Non-Web Projects

| Project Type | Screen | Connection | Processing | Response | Error |
|-------------|--------|------------|------------|----------|-------|
| **Web app** | UI component renders | Click/submit triggers API call | Backend processes request | Success data returned, UI updates | Error message displayed |
| **API-only** | API endpoint exists | Request reaches handler | Business logic executes | Response body correct | Error response with correct status code |
| **CLI tool** | Command accepted | Arguments parsed | Operation performed | Output printed | Error message + exit code |
| **Mobile app** | Screen/widget renders | Tap triggers action | Processing occurs | UI navigates/updates | Error dialog/snackbar shown |
| **Desktop app** | Window/panel renders | Menu/button triggers action | Processing occurs | UI updates | Dialog/notification shown |

The 5-step chain (Screen -> Connection -> Processing -> Response -> Error) is universal. Only the concrete manifestation changes per project type. The spec format remains identical.

### Internal Boundaries

| Boundary | Communication | Notes |
|----------|---------------|-------|
| Spec <-> Prototype | `e2e-NNN` IDs | Spec defines the contract; prototype visualizes it. Same IDs in both. |
| Spec <-> DO stage | Status field updates | DO reads spec steps, implements each, updates status to `implemented`. |
| Spec <-> TEST stage | Verification criteria | TEST reads criteria from spec, checks against code, updates to `verified`/`failed`. |
| Prototype <-> TEST stage | `spec-coverage` JSON | TEST parses coverage JSON to know what structural elements to verify in implementation. |
| TEST <-> DO stage (rework) | Gap list in verification.json | TEST produces specific gaps; DO reads and fixes only those gaps. |
| Verification gate <-> COMMIT stage | verification.json summary | If `gate: "BLOCKED"`, COMMIT is not allowed. If `gate: "PASSED"`, COMMIT proceeds. |

## Build Order (Dependency-Driven)

The previous ARCHITECTURE.md placed prototypes in Phase 4. This is wrong. Prototypes and E2E specs are the core value proposition (they solve Problem #1 from PROJECT.md, which has 100% failure rate). They must be in Phase 1 or early Phase 2.

```
Phase 1: Foundation + Inner Loop Core
├── State Engine (state.json schema + read/write)
├── Manifest schema (artifact registry)
├── E2E Spec format definition (the 5-step chain template)
├── Prototype template (standard HTML skeleton with data-spec-id pattern)
├── PLAN adapter: produces spec.md + prototype.html
├── DO adapter: reads spec as checklist, updates status
├── TEST adapter: verifies spec + prototype structural diff
└── Verification gate: blocks COMMIT if any step failed

Phase 2: Outer Loop + Polish
├── COMMIT / DEPLOY / DEPLOY TEST adapters
├── DOCUMENT / RETROSPECT / PROMOTE adapters
├── Prototype template variants (API-only, CLI, mobile)
├── Session resumption with spec-aware state display
└── Scope detection (skip prototype for hotfix/config modes)
```

**Why this revised order:**
- The prototype + spec system IS the inner loop. It is not an enhancement to the inner loop; it is the mechanism that makes the inner loop work. Without it, DO produces incomplete features (100% failure rate per PROJECT.md).
- Building the foundation (state engine) without the spec system means Phase 1 produces a pipeline that routes between stages but does not solve the core problem.
- The spec format and prototype template are simple to define. The complexity is in the TEST structural diff logic, but even a basic version (check that spec IDs have corresponding code comments) is valuable.

## Sources

- Direct analysis of existing skill files:
  - `~/.claude/skills/dev-lifecycle/SKILL.md` (current muse-hardcoded version)
  - `~/.claude/get-shit-done/references/verification-patterns.md` (wiring verification patterns)
  - `~/.claude/get-shit-done/workflows/execute-phase.md` (orchestration reference)
- PROJECT.md requirements:
  - "User Interaction Prototype -- PLAN stage core output, not optional"
  - "End-to-end feature spec -- Screen -> Connection -> Processing -> Response -> Error"
  - "100% failure rate on feature completeness without upfront validation"
- PITFALLS.md findings:
  - Pitfall 1: "Done But Not Done" illusion -- solved by spec verification gate
  - Pitfall 4: Specification Gap -- solved by clickable prototype
  - Pitfall 6: Artifact Fragmentation -- solved by spec ID threading

---
*Architecture research for: dev-lifecycle orchestrator (prototype + E2E spec integration)*
*Researched: 2026-03-22*
