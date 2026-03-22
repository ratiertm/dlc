# Feature Research: User Interaction Prototype & E2E Feature Spec

**Domain:** AI Development Lifecycle Orchestration (Claude Code Skill)
**Researched:** 2026-03-22
**Confidence:** HIGH (core problem well-understood; format design is opinionated based on research)
**Focus:** The two PRIMARY value proposition features and their integration into PLAN/DO/TEST

## The Core Problem These Features Solve

Claude says "done" but 100% of the time features do not actually work. Buttons missing, API calls not connected, error handling absent. Root cause: there is no shared, verifiable definition of "done" between human and AI. The AI checks "does the function exist?" not "does the feature work end-to-end?"

These two features attack the problem from complementary directions:
- **User Interaction Prototype**: Visual, clickable agreement on WHAT the user experiences
- **E2E Feature Spec**: Structured, verifiable chain of HOW it works under the hood

---

## Feature 1: User Interaction Prototype (UIP)

### What It Is

A standalone HTML+JS file generated during PLAN that simulates the user's interaction flow. Not a design mockup -- a functional flow validator. User clicks Button A, sees Panel B appear, fills Form C, gets Response D (mocked). Every interaction point becomes a checkpoint the implementation must satisfy.

### Table Stakes for UIP

| Sub-Feature | Why Expected | Complexity | Notes |
|-------------|--------------|------------|-------|
| **Standalone single-file HTML** | Must open in any browser with zero dependencies. No build step, no server, no npm install. Double-click to run. | LOW | Single .html file with embedded CSS+JS. Inline everything. |
| **All user-facing interactions represented** | Every button, form, navigation, modal that the feature requires must exist in the prototype. If a user can click it in the final product, they must be able to click it in the prototype. | MEDIUM | This is the whole point -- missing interactions in prototype = missing interactions in code. |
| **Mock responses for each action** | Clicking "Submit" must show a mock response (success message, error state, loading indicator). Without mocked responses, the prototype is just a wireframe. | MEDIUM | Use setTimeout to simulate async. Show realistic mock data, not "lorem ipsum." |
| **Error state representation** | Each interaction must show what happens on failure, not just success. "What if the API returns 500?" must be clickable. | MEDIUM | Toggle between success/error states via a small control panel or keyboard shortcut in the prototype. |
| **Interaction checklist embedded** | The prototype itself must contain a visible checklist of all interactions it demonstrates. This becomes the verification contract. | LOW | Rendered as a sidebar or footer in the HTML. Each item links to the interaction it describes. |

### Differentiators for UIP

| Sub-Feature | Value Proposition | Complexity | Notes |
|-------------|-------------------|------------|-------|
| **State machine visualization** | Show which states the feature can be in (idle, loading, success, error, empty) and transitions between them. Makes implicit states explicit. | HIGH | Could be a simple state diagram rendered in the prototype, or interactive state toggles. |
| **Data flow annotations** | Overlay showing "this button sends POST /api/chat" and "this panel receives WebSocket message." Connects the visible UI to the invisible plumbing. | MEDIUM | Togglable overlay. Off by default (clean UX), on for developer review. |
| **Diff-against-implementation** | After DO stage, compare prototype interactions against actual implementation. Highlight which interactions were implemented vs. missed. | HIGH | Requires the interaction checklist to be in a parseable format that TEST can compare. |

### UIP Integration into PLAN / DO / TEST

**PLAN Stage -- Generation and Agreement:**
```
1. User describes the feature they want
2. Orchestrator generates E2E spec (see Feature 2 below)
3. Orchestrator generates UIP from the E2E spec
4. Output: .planning/prototypes/{feature-name}.html
5. User opens prototype in browser, clicks through every flow
6. User confirms: "Yes, this is what I want" OR gives corrections
7. Corrections loop: regenerate until user agrees
8. Agreement = the prototype IS the acceptance criteria
```

**DO Stage -- Implementation Target:**
```
1. DO stage receives the agreed prototype as input artifact
2. The interaction checklist from the prototype becomes the implementation TODO list
3. Each checklist item maps to a concrete implementation task:
   - "Click 'Send' button" -> implement onClick handler
   - "Show loading spinner" -> implement loading state
   - "Display response in chat panel" -> implement response rendering
   - "Show error toast on failure" -> implement error handling
4. Implementation is NOT complete until every checklist item has corresponding code
5. The orchestrator can reference the prototype checklist to prevent premature "done"
```

**TEST Stage -- Verification Against:**
```
1. TEST receives both the prototype and the implementation
2. Verification is checklist-based (not automated browser testing):
   - For each interaction in the prototype checklist:
     a. Does the UI element exist in the implementation?
     b. Does it trigger the expected behavior?
     c. Does the success response render correctly?
     d. Does the error response render correctly?
3. Each checklist item gets PASS/FAIL
4. Any FAIL = back to DO with specific gap identified
5. All PASS = feature verified as matching the agreed prototype
```

### UIP Format Specification

```html
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <title>UIP: {Feature Name}</title>
  <style>
    /* Minimal wireframe styling -- NOT production CSS */
    /* Focus on layout and interaction, not aesthetics */
  </style>
</head>
<body>
  <header id="uip-meta">
    <h1>User Interaction Prototype: {Feature Name}</h1>
    <p>Generated: {date} | Status: {draft|agreed|verified}</p>
  </header>

  <!-- === INTERACTION CHECKLIST === -->
  <!-- This section is the verification contract -->
  <aside id="uip-checklist">
    <h2>Interaction Checklist</h2>
    <ol>
      <li data-uip-id="INT-001" data-status="pending">
        User clicks "Send" button -> POST request fires
      </li>
      <li data-uip-id="INT-002" data-status="pending">
        Loading spinner appears while waiting
      </li>
      <li data-uip-id="INT-003" data-status="pending">
        Response renders in chat panel on success
      </li>
      <li data-uip-id="INT-004" data-status="pending">
        Error toast appears on API failure
      </li>
      <li data-uip-id="INT-005" data-status="pending">
        Empty state shows when no messages exist
      </li>
    </ol>
  </aside>

  <!-- === PROTOTYPE UI === -->
  <main id="uip-prototype">
    <!-- Functional wireframe UI here -->
  </main>

  <script>
    // Mock interactions with realistic delays and data
    // Each interaction corresponds to a checklist item via data-uip-id
  </script>
</body>
</html>
```

Key format decisions:
- **data-uip-id attributes** tie UI elements to checklist items, making verification traceable
- **data-status** attribute tracks PASS/FAIL during TEST stage
- **Embedded checklist** means the spec and the prototype are one artifact, not two files that drift apart

---

## Feature 2: End-to-End Feature Spec (E2E Spec)

### What It Is

A structured specification that traces every feature through five layers: Screen (what the user sees) -> Connection (how frontend talks to backend) -> Processing (what the backend does) -> Response (what comes back) -> Error (what happens when it fails). Each layer is a testable assertion. When any layer fails, the exact break point is identified.

### Why Not Gherkin/BDD

Gherkin (Given/When/Then) was evaluated and rejected for this use case:
- Gherkin describes BEHAVIOR but not the CHAIN. "Given I am on the login page, When I submit, Then I am logged in" hides all the plumbing (what API? what validation? what error?).
- Gherkin is designed for human-readable test automation, but we need a format that is a CHECKLIST for the AI during implementation, not a test runner input.
- Gherkin scales poorly: as test suites grow, maintenance becomes a bottleneck (source: [TestQuality Gherkin Guide](https://testquality.com/gherkin-bdd-cucumber-guide-to-behavior-driven-development/)).
- Our format needs to be YAML-based for machine parseability while remaining readable in markdown rendering.

### Table Stakes for E2E Spec

| Sub-Feature | Why Expected | Complexity | Notes |
|-------------|--------------|------------|-------|
| **5-layer chain for every feature** | Screen->Connection->Processing->Response->Error must all be specified. Missing any layer = the gap where "done but broken" lives. | MEDIUM | Most "incomplete" features fail at Connection (API not called) or Error (no error handling). These are the layers AI skips. |
| **Each layer is a checkable assertion** | "POST /api/chat with body {message: string}" is checkable. "Sends data to server" is not. Every layer must be specific enough to verify YES/NO. | MEDIUM | Use concrete values: HTTP method, endpoint path, request body shape, response shape, error codes. |
| **YAML format with markdown rendering** | Must be parseable by the orchestrator (YAML) AND readable by humans (renders nicely in markdown preview). | LOW | YAML in a fenced code block inside a .md file. Or standalone .yaml file with comment annotations. |
| **Unique ID per chain** | Each feature chain gets an ID (e.g., FEAT-001) that traces through PLAN->DO->TEST. Enables "FEAT-001 failed at Connection layer" reporting. | LOW | Simple sequential numbering. Referenced in prototype checklist, commit messages, and test results. |
| **Dependency declaration** | If FEAT-002 depends on FEAT-001 (e.g., chat requires auth), this must be explicit. Implementation order follows dependency order. | LOW | `depends_on: [FEAT-001]` field in the spec. |

### Differentiators for E2E Spec

| Sub-Feature | Value Proposition | Complexity | Notes |
|-------------|-------------------|------------|-------|
| **Layer-level progress tracking** | During DO, track which layers of which features are implemented. "FEAT-001: Screen DONE, Connection DONE, Processing TODO, Response TODO, Error TODO" gives real progress, not "50% done." | MEDIUM | Update spec file in-place during DO. Each layer gets a status field. |
| **Cross-feature connection mapping** | Show which features share endpoints, databases, or UI components. Prevents "I changed the /api/chat response shape and broke 3 features." | HIGH | Computed from the Connection and Processing layers across all feature specs. |
| **Regression chain** | When a bug is found, trace which layer broke and which upstream change caused it. "Error layer of FEAT-003 broke because Processing layer of FEAT-001 changed response shape." | HIGH | Requires version tracking of spec changes. Future consideration. |

### E2E Spec Format (Concrete)

This is the recommended format. It is YAML-based, human-readable, and machine-parseable.

```yaml
# File: .planning/specs/{feature-name}.e2e.yaml
# Rendered in markdown docs, parsed by orchestrator at TEST

feature: Chat Message Send
id: FEAT-001
depends_on: [AUTH-001]  # Must be authenticated
prototype: prototypes/chat.html  # Links to UIP
status: planned  # planned | in-progress | implemented | verified | failed

chain:
  - layer: screen
    description: "Chat input field and send button"
    assertions:
      - element: "text input with placeholder 'Type a message...'"
        selector_hint: "input#chat-input"  # Hint, not binding
      - element: "Send button, enabled only when input is non-empty"
        selector_hint: "button#send-btn"
      - element: "Message list showing sent and received messages"
        selector_hint: "div#message-list"
    status: pending  # pending | pass | fail

  - layer: connection
    description: "Frontend sends message to backend API"
    assertions:
      - method: POST
        endpoint: /api/v1/chat/messages
        request_body:
          type: object
          required: [message, conversation_id]
          properties:
            message: { type: string, min_length: 1 }
            conversation_id: { type: string, format: uuid }
        headers:
          Authorization: "Bearer {token}"
    status: pending

  - layer: processing
    description: "Backend validates, stores, and generates response"
    assertions:
      - step: "Validate message is non-empty string"
      - step: "Verify conversation_id exists and belongs to user"
      - step: "Store message in database with timestamp"
      - step: "Call LLM API with conversation context"
      - step: "Store LLM response in database"
    status: pending

  - layer: response
    description: "Backend returns response, frontend renders it"
    assertions:
      - http_status: 200
        response_body:
          type: object
          required: [id, message, created_at, ai_response]
          properties:
            id: { type: string, format: uuid }
            message: { type: string }
            created_at: { type: string, format: iso8601 }
            ai_response:
              type: object
              required: [id, content, created_at]
              properties:
                id: { type: string, format: uuid }
                content: { type: string }
                created_at: { type: string, format: iso8601 }
      - ui_update: "New message appears in message list"
      - ui_update: "AI response appears below user message"
      - ui_update: "Input field is cleared after send"
    status: pending

  - layer: error
    description: "Error handling for all failure modes"
    assertions:
      - condition: "Empty message submitted"
        behavior: "Send button disabled, no API call"
        http_status: null  # Client-side only
      - condition: "Network timeout"
        behavior: "Toast notification 'Failed to send. Retry?'"
        http_status: null
      - condition: "Invalid conversation_id"
        behavior: "Redirect to conversation list"
        http_status: 404
        response_body: { error: "Conversation not found" }
      - condition: "Authentication expired"
        behavior: "Redirect to login"
        http_status: 401
      - condition: "LLM API failure"
        behavior: "Toast 'AI is temporarily unavailable. Message saved.'"
        http_status: 503
        response_body: { error: "Service unavailable", message_saved: true }
    status: pending
```

### Why This Format

1. **YAML, not Gherkin**: Structured data that any programming language can parse. No custom parser needed. The orchestrator reads it as a data structure, not a DSL.

2. **YAML, not plain Markdown**: Markdown checklists (`- [ ] thing`) are human-readable but not machine-parseable without fragile regex. YAML gives typed fields, nesting, and standard parsers.

3. **YAML in a standalone file, not embedded in .md**: The spec must be the single source of truth. Embedding it in a larger planning document risks it drifting from the authoritative version. The `.e2e.yaml` extension makes it discoverable by the orchestrator.

4. **Assertions are concrete, not abstract**: "POST /api/v1/chat/messages" not "sends data to server." "http_status: 404" not "returns an error." Concrete assertions have one interpretation; abstract ones have many.

5. **Status field per layer**: Enables layer-level progress tracking during DO and layer-level pass/fail during TEST.

6. **selector_hint, not selector**: The spec hints at expected selectors but does not bind to them. Implementation may use different selectors -- the assertion is "does this element exist and work?" not "does it have this exact CSS selector."

### E2E Spec Integration into PLAN / DO / TEST

**PLAN Stage -- Specification Writing:**
```
1. User describes the feature
2. Orchestrator generates E2E spec as .e2e.yaml file
3. Output: .planning/specs/{feature-name}.e2e.yaml
4. User reviews each layer:
   - Screen: "Are these all the UI elements?"
   - Connection: "Is this the right API shape?"
   - Processing: "Are these the right business rules?"
   - Response: "Is this what success looks like?"
   - Error: "Are these all the failure modes?"
5. User confirms OR requests changes
6. Agreement = the spec IS the implementation contract
7. Spec feeds into UIP generation (prototype visualizes the spec)
```

**DO Stage -- Implementation Checklist:**
```
1. DO stage receives the .e2e.yaml as input artifact
2. Implementation follows the chain order:
   a. Screen layer first: build all UI elements listed in assertions
   b. Connection layer: wire up API calls with correct method/endpoint/body
   c. Processing layer: implement each step in backend
   d. Response layer: handle response and update UI
   e. Error layer: implement EVERY error condition listed
3. As each layer is implemented, update status: pending -> pass
4. The orchestrator checks: are all layers marked as implemented?
5. Implementation is NOT "done" until all 5 layers have status != pending
6. Key insight: the AI cannot skip the error layer because it is an
   explicit, required section of the spec -- not an afterthought
```

**TEST Stage -- Layer-by-Layer Verification:**
```
1. TEST stage reads the .e2e.yaml
2. For each feature chain, verify layer by layer:
   a. Screen: Do all listed elements exist in the UI?
   b. Connection: Does the API call use the correct method/endpoint/body?
   c. Processing: Do the backend steps execute correctly?
   d. Response: Does the response match the specified shape?
   e. Error: Does each error condition produce the specified behavior?
3. Each layer gets: pass | fail
4. If any layer fails:
   - Report: "FEAT-001 FAILED at Connection layer"
   - Detail: "Expected POST /api/v1/chat/messages, found: endpoint not called"
   - Action: Return to DO with specific layer to fix
5. All layers pass = feature verified end-to-end
```

---

## How UIP and E2E Spec Work Together

```
PLAN Stage:
  1. User describes feature
  2. E2E spec generated (.e2e.yaml)     -- the CONTRACT
  3. UIP generated (.html)               -- the VISUAL of the contract
  4. User reviews BOTH:
     - Clicks through UIP to verify flow
     - Reviews spec to verify technical chain
  5. Agreement on both = "done" is now defined

DO Stage:
  1. E2E spec = implementation checklist (5 layers, each with assertions)
  2. UIP = visual reference (what should it look like when working?)
  3. Each UIP interaction maps to E2E spec chain via shared IDs:
     - UIP checklist item INT-001 -> E2E spec FEAT-001 screen+connection layers
  4. "Done" = all spec layers implemented + all UIP interactions functional

TEST Stage:
  1. E2E spec = verification criteria (layer-by-layer pass/fail)
  2. UIP = comparison target (does implementation match prototype?)
  3. Verification output:
     Feature: Chat Message Send (FEAT-001)
     Screen:     PASS (all 3 elements present)
     Connection: PASS (POST /api/v1/chat/messages confirmed)
     Processing: PASS (5/5 steps verified)
     Response:   PASS (response shape matches)
     Error:      FAIL (2/5 conditions handled)
       - MISSING: Network timeout handling
       - MISSING: LLM API failure handling
     Result: FAIL -> return to DO with specific gaps
```

---

## Anti-Features

| Feature | Why Requested | Why Problematic | Alternative |
|---------|---------------|-----------------|-------------|
| **Pixel-perfect UI mockups** | "Show me exactly what it looks like" | Creates arguments about colors/fonts instead of validating interaction flows. Claude generates mediocre CSS. False precision. | UIP focuses on FLOW (wireframe-level). Use existing UI-SPEC.md for visual design contracts. |
| **Auto-generated Playwright/Cypress tests from spec** | "Turn the spec into real test code" | Generated test code is brittle, selector-dependent, and gives false confidence. Maintaining generated tests becomes its own project. | E2E spec IS the test contract. Human verifies against spec. If project has existing test framework, spec informs what to test -- orchestrator does not generate test code. |
| **Gherkin/BDD format for specs** | "Gherkin is the standard for behavior specs" | Gherkin hides the connection chain. "Then I am logged in" does not specify the API call, the token format, or the error cases. Also requires a Cucumber-like runner. | YAML-based 5-layer spec. More structured, more machine-parseable, explicitly covers the layers that Gherkin abstracts away. |
| **UIP with real API calls** | "Make the prototype hit real endpoints" | Prototype becomes a mini-app that needs its own debugging. Breaks the "open HTML file, click around" simplicity. Adds auth, CORS, and deployment concerns to PLAN stage. | All API calls in UIP are mocked with setTimeout + realistic fake data. The spec documents real endpoints; the prototype simulates them. |
| **Combined spec+prototype in one file** | "Keep everything in one place" | HTML is a poor container for structured YAML data. YAML is a poor container for interactive UI. Editing one breaks the other. | Two files linked by shared IDs: `.e2e.yaml` (spec) + `.html` (prototype). The spec references `prototype: prototypes/chat.html`. The prototype references `data-spec-id: FEAT-001`. |
| **AI self-verification without human** | "Let the AI verify its own work" | The entire problem is that AI cannot reliably verify its own completeness. Self-verification is the current broken pattern. | Human verifies against prototype and spec. AI assists by presenting structured pass/fail against each spec layer, but HUMAN confirms. |

## Feature Dependencies

```
[E2E Feature Spec]
    |
    |--feeds into--> [User Interaction Prototype]
    |                    (prototype visualizes the spec)
    |
    |--consumed by--> [Stage Artifact Chaining]
    |                    (spec flows PLAN->DO->TEST)
    |
    |--verified by--> [Cross-Stage Verification Chain]
                         (TEST checks spec layer by layer)

[User Interaction Prototype]
    |
    |--requires--> [E2E Feature Spec]
    |                 (prototype needs spec to know what to simulate)
    |
    |--compared against--> [Implementation at TEST]
                              (does code match prototype?)

[E2E Feature Spec] --conflicts--> [Gherkin/BDD Specs]
    (pick one format; mixing creates confusion about which is authoritative)

[User Interaction Prototype] --conflicts--> [Pixel-Perfect Mockups]
    (pick one prototype style; mixing creates "which one is right?" debates)
```

### Dependency Notes

- **E2E Spec feeds UIP**: The spec is written first. The prototype is generated FROM the spec. This order matters -- if the prototype comes first, it lacks the technical chain detail.
- **E2E Spec requires Artifact Chaining**: Without artifact chaining, the spec file does not automatically flow from PLAN to DO to TEST. Someone must manually pass it.
- **UIP requires E2E Spec**: A prototype without a spec is just a wireframe. The spec defines WHAT the prototype simulates. The shared ID system (INT-xxx mapped to FEAT-xxx) enables traceability.

## MVP Definition

### Launch With (v1) -- Both Features as Core

Both UIP and E2E Spec are v1 requirements because they solve the PRIMARY problem (the "done but broken" problem). Without both, the value proposition is incomplete.

- [ ] **E2E Feature Spec format** -- YAML-based 5-layer chain. Parser in orchestrator that reads .e2e.yaml files. Status tracking per layer.
- [ ] **E2E Spec generation in PLAN** -- Orchestrator generates spec from user description. User reviews and agrees.
- [ ] **E2E Spec as DO checklist** -- DO stage presents spec layers as implementation TODO. Tracks layer completion.
- [ ] **E2E Spec as TEST criteria** -- TEST stage reads spec and reports pass/fail per layer with specific gap identification.
- [ ] **User Interaction Prototype generation** -- HTML+JS file generated from agreed E2E spec. Embeds interaction checklist.
- [ ] **UIP user agreement flow** -- User opens prototype, clicks through, confirms or requests changes. Loop until agreed.
- [ ] **UIP-to-spec ID linking** -- Prototype checklist items reference spec feature IDs. Traceability from visual to technical.

### Add After Validation (v1.x)

- [ ] **State machine visualization in UIP** -- Add when: basic UIP proves valuable but users request explicit state diagrams
- [ ] **Layer-level progress dashboard** -- Add when: spec tracking works but users want a visual summary of implementation progress
- [ ] **Cross-feature connection mapping** -- Add when: multi-feature projects reveal dependency blindness
- [ ] **Data flow annotations in UIP** -- Add when: UIP proves useful for non-technical stakeholders and they request technical overlay

### Future Consideration (v2+)

- [ ] **Regression chain tracking** -- Track spec version history to trace breaking changes across features
- [ ] **Diff-against-implementation tool** -- Automated comparison of prototype vs. actual implementation DOM
- [ ] **Spec template library** -- Pre-built specs for common patterns (CRUD, auth, file upload, real-time chat)

## Feature Prioritization Matrix

| Feature | User Value | Implementation Cost | Priority |
|---------|------------|---------------------|----------|
| E2E Spec format (.e2e.yaml) | HIGH | MEDIUM | P1 |
| E2E Spec PLAN generation | HIGH | MEDIUM | P1 |
| E2E Spec DO checklist | HIGH | LOW | P1 |
| E2E Spec TEST verification | HIGH | MEDIUM | P1 |
| UIP generation from spec | HIGH | MEDIUM | P1 |
| UIP agreement flow | HIGH | LOW | P1 |
| UIP-to-spec ID linking | MEDIUM | LOW | P1 |
| State machine in UIP | MEDIUM | HIGH | P2 |
| Layer progress dashboard | MEDIUM | MEDIUM | P2 |
| Cross-feature mapping | MEDIUM | HIGH | P2 |
| Data flow annotations | LOW | MEDIUM | P3 |
| Regression chain | LOW | HIGH | P3 |
| Diff-against-implementation | MEDIUM | HIGH | P3 |
| Spec template library | MEDIUM | MEDIUM | P3 |

## Competitor / Prior Art Analysis

| Approach | How It Works | What It Misses | Our Improvement |
|----------|-------------|----------------|-----------------|
| **GSD verify-work (current)** | UAT-style pass/fail on broad criteria | No structured chain. "Does login work?" not "Does login POST to /auth with correct body?" | E2E spec breaks every feature into 5 verifiable layers |
| **Gherkin/Cucumber BDD** | Given/When/Then scenarios | Hides connection and processing layers. No error chain. Requires runner infrastructure. | YAML 5-layer chain. No runner needed. Error layer is mandatory, not optional. |
| **OpenAPI/Swagger** | API contract specification | Only covers Connection layer. No Screen, Processing, or Error UX layers. | E2E spec covers all 5 layers including frontend and error UX |
| **Figma/design tool prototypes** | Pixel-perfect visual mockups | Cannot validate interaction logic. No mock responses. No error states. Require external tools. | UIP is a standalone HTML file. Validates flow, not pixels. Includes mock responses and error states. |
| **Claude Artifacts** | HTML preview in Claude conversation | Ephemeral -- disappears after session. Not version-controlled. No checklist embedded. | UIP is a file in the project. Version-controlled. Checklist embedded. Persists across sessions. |
| **Addy Osmani's spec approach** | Structured PRD with acceptance criteria | Focused on giving AI instructions, not on verifying AI output layer-by-layer. | E2E spec serves as both instruction (DO) and verification (TEST). Bidirectional. |
| **taskmd format** | YAML frontmatter + markdown body for AI tasks | Task-level, not feature-chain-level. No 5-layer breakdown. | E2E spec is feature-chain-level with explicit layers that prevent skipping. |

## Sources

- [Addy Osmani: How to Write a Good Spec for AI Agents](https://addyosmani.com/blog/good-spec/) -- Conformance testing, structured specs, self-audit patterns (HIGH confidence)
- [TestQuality: Gherkin BDD Guide](https://testquality.com/gherkin-bdd-cucumber-guide-to-behavior-driven-development/) -- Gherkin pros/cons analysis (MEDIUM confidence)
- [CodeRabbit: 2026 Year of AI Quality](https://www.coderabbit.ai/blog/2025-was-the-year-of-ai-speed-2026-will-be-the-year-of-ai-quality) -- AI code 1.7x more issues, need for verification (MEDIUM confidence)
- [Anthropic: Demystifying Evals for AI Agents](https://www.anthropic.com/engineering/demystifying-evals-for-ai-agents) -- Agent evaluation approaches (HIGH confidence)
- [taskmd: Task Management for AI Era](https://medium.com/@driangle/taskmd-task-management-for-the-ai-era-92d8b476e24e) -- YAML+Markdown format for AI tasks (LOW confidence, single source)
- [BDD Cucumber Gherkin Scaling Issues](https://testquality.com/scaling-gherkin-software-testing-strategies/) -- Gherkin maintenance challenges at scale (MEDIUM confidence)
- [UX Planet: Claude for Product Design](https://uxplanet.org/claude-for-code-how-to-use-claude-to-streamline-product-design-process-97d4e4c43ca4) -- Claude HTML prototype generation workflow (MEDIUM confidence)

---
*Feature research for: User Interaction Prototype & E2E Feature Spec*
*Researched: 2026-03-22*
