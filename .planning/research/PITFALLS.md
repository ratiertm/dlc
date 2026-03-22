# Pitfalls Research

**Domain:** User Interaction Prototype + E2E Feature Spec as Verification Mechanisms
**Researched:** 2026-03-22
**Confidence:** HIGH (user's lived experience + verified community patterns + spec-driven development literature)

## Critical Pitfalls

### Pitfall 1: Self-Verification Blindness -- Same AI Writes Spec AND Implements Code

**What goes wrong:**
When Claude generates the HTML+JS prototype AND implements the production code, both artifacts share the same blind spots. If Claude misunderstands the user's intent, the prototype will look "correct" (to Claude) and the implementation will match the prototype -- but both are wrong in the same way. The prototype confirms the implementation, but neither reflects what the user actually wanted.

This is the AI equivalent of a student grading their own exam. The prototype becomes a mirror of Claude's assumptions, not a mirror of user intent. Verification debt accumulates silently: tests are green, prototype matches implementation, but the customer gets something they never asked for.

**Why it happens:**
1. **Shared mental model.** Claude's prototype and Claude's implementation come from the same training data and the same context window. Systematic gaps (e.g., never considering keyboard navigation, always defaulting to modal dialogs) appear in both artifacts identically.
2. **Confirmation bias in test generation.** When Claude writes tests for code it also wrote, it tests the behavior it implemented -- not the behavior the user intended. Tests reinforce existing code rather than verifying correctness.
3. **No adversarial perspective.** Good verification requires questioning assumptions. Claude optimizes for coherent, plausible output -- not for challenging its own prior output.

**How to avoid:**
1. **User clicks the prototype, not Claude.** The prototype is ONLY useful if a human interacts with it and provides feedback. The prototype is a communication device, not a verification device. If the user does not click through it, it has zero value.
2. **Explicit "what did I miss?" prompt in PLAN.** After generating the prototype, force a structured review: "List 5 interactions this prototype does NOT cover. List 3 error states not shown. List the keyboard/accessibility paths not demonstrated."
3. **Separate the verification role.** TEST stage should re-read the original user request (not the prototype) and verify implementation against user words, not against Claude's interpretation of user words.
4. **Require user confirmation as a gate.** Prototype approval is a human gate, not an AI gate. PLAN cannot transition to DO without explicit user sign-off on the prototype.

**Warning signs:**
- Prototype is generated and immediately accepted without user interaction
- Claude says "the prototype matches the implementation" as if that proves correctness
- No user feedback recorded between prototype generation and implementation start
- Prototype covers only the happy path

**Phase to address:**
PLAN stage -- the prototype generation step must include mandatory user review and a "what's missing" self-audit. DO stage must reference the original user request alongside the prototype.

---

### Pitfall 2: Prototype Fidelity Trap -- Prototype Is Too Simple to Catch Real Problems

**What goes wrong:**
The HTML+JS prototype shows that a button exists and clicking it shows a success message. But it cannot demonstrate: real data loading latency, concurrent state conflicts, validation against actual data schemas, API error responses, permission-denied flows, or cross-component side effects. The user approves the prototype thinking "this is what I'll get," but the production implementation has entirely different behavior at the boundaries.

The prototype catches "is the button there?" (which was the original 100% failure problem) but misses "does the button do the right thing when the server is slow, the data is malformed, or the user is unauthorized?"

**Why it happens:**
1. **Prototypes are inherently shallow.** A clickable HTML+JS mockup can show UI flow but cannot simulate backend state, race conditions, or integration failures. It answers "what does the user see?" but not "what happens underneath?"
2. **False confidence from visual confirmation.** The user sees the prototype working and assumes the hard part is done. In reality, the hard part (wiring, error handling, state management) is entirely unrepresented.
3. **Prototype scope creep resistance.** Making prototypes more realistic means adding backend simulation, state management, error injection -- at which point you are building the app, not prototyping it.

**How to avoid:**
1. **Separate concerns between prototype and E2E spec.** Prototype answers: "What does the user see and click?" E2E spec answers: "What happens at each layer when they click it?" These are complementary, not redundant. Neither alone is sufficient.
2. **Prototype shows ONLY UI flow + interactions.** Do not try to make the prototype "realistic." Its job is to be a visual contract for layout, navigation, and interaction sequence. Nothing more.
3. **E2E spec explicitly covers what prototype cannot.** For each prototype interaction, the E2E spec must list: what API call fires, what the backend does, what happens on 400/500 error, what happens on timeout, what the loading state looks like, what happens if data is empty.
4. **Name the gap explicitly.** Every prototype should include a "NOT DEMONSTRATED" section listing what it cannot show: error states, loading states, edge cases, permission variations.

**Warning signs:**
- User approves prototype without seeing any error states
- E2E spec is written AFTER implementation (should be written WITH prototype)
- Prototype includes fake "loading..." animations that give false impression of realism
- No "NOT DEMONSTRATED" section in prototype documentation

**Phase to address:**
PLAN stage -- prototype and E2E spec must be generated together as a pair. The E2E spec template must have explicit sections for what the prototype does NOT cover.

---

### Pitfall 3: Prototype-Implementation Drift -- Prototype Says X, Implementation Becomes Y

**What goes wrong:**
The prototype shows a specific interaction flow. During DO stage, Claude encounters technical constraints (framework limitations, state management complexity, API shape differences) and silently adapts the implementation. The adaptation may be reasonable, but it was never validated against user expectations. The prototype becomes a historical artifact rather than a living specification.

Example: Prototype shows inline editing. During implementation, Claude discovers it is complex with the chosen state management approach, switches to a modal dialog. User expected inline editing. Nobody catches the drift until testing.

**Why it happens:**
1. **Implementation reveals unknown constraints.** Prototypes are constraint-free. Real code has framework opinions, library limitations, and performance requirements that force design changes.
2. **AI optimizes for "working code" not "matching spec."** During DO, Claude's priority shifts from "match the prototype" to "make it compile and run." Deviations happen silently because Claude does not flag them -- it just builds what works.
3. **No diff mechanism.** There is no tooling to compare "what the prototype promised" against "what was actually built." The drift is invisible until a human manually compares them.
4. **Spec drift in AI is well-documented.** Each AI iteration accumulates undocumented choices. Specifications become post-hoc documentation rather than guiding documents.

**How to avoid:**
1. **E2E spec as the binding contract, not the prototype.** The prototype is a visual aid. The E2E spec (screen -> connection -> processing -> response -> error) is the implementation contract. DO stage checks off E2E spec items, not prototype screenshots.
2. **Deviation log.** When Claude deviates from the spec during DO, it must record: what was specified, what was implemented instead, and why. This is not optional -- any unlogged deviation is a bug.
3. **TEST stage compares against E2E spec line-by-line.** Each E2E spec item gets a PASS/FAIL/DEVIATED status. DEVIATED requires user approval.
4. **Small increments.** Implement one E2E spec item at a time. Verify before moving to next. This prevents compounding drift where deviation A forces deviation B forces deviation C.

**Warning signs:**
- DO stage output does not reference the prototype or E2E spec
- Claude says "I had to change the approach because..." without recording it formally
- TEST stage verifies "it works" but not "it matches what was planned"
- Multiple deviations discovered at TEST stage instead of during DO

**Phase to address:**
DO stage -- must include mandatory spec-reference and deviation logging. TEST stage -- must compare against E2E spec items, not just "does it work."

---

### Pitfall 4: E2E Spec Format Gaps -- What screen->connection->processing->response->error Misses

**What goes wrong:**
The 5-step E2E format (screen -> connection -> processing -> response -> error) covers the request-response cycle well but has systematic blind spots:

1. **No state management spec.** What changes in the application state? What other components re-render? What gets cached/invalidated?
2. **No concurrent interaction spec.** What happens when the user clicks another button while this request is in-flight? What about two browser tabs?
3. **No undo/rollback spec.** If the operation partially succeeds, what state is the user left in?
4. **No accessibility spec.** Keyboard navigation, screen reader announcements, focus management.
5. **No performance spec.** Expected response time, what happens if it exceeds threshold.
6. **No sequence/ordering spec.** What must happen before this feature works? (Auth? Onboarding? Data existence?)
7. **No "between states" spec.** The loading state between screen and response. The optimistic update. The skeleton UI.

These gaps are precisely the things that cause "100% of features don't work" -- the happy path works, but the transitions, edge cases, and state interactions are missing.

**Why it happens:**
1. **Format covers the vertical slice, misses horizontal concerns.** screen->connection->processing->response->error is a single request path. Real features involve multiple requests, shared state, and cross-cutting concerns.
2. **Error section is too narrow.** "Error" in the E2E spec means "API returns error." But errors also include: timeout, offline, partial success, validation failure before request, stale data, concurrent modification.
3. **Linear format for non-linear interactions.** Real UIs have parallel paths, conditional flows, and state that persists across multiple E2E chains.

**How to avoid:**
1. **Extend the format to 7 steps.** screen -> connection -> processing -> response -> error -> **state change** -> **side effects**. "State change" covers what updates in the app. "Side effects" covers what else needs to happen (cache invalidation, other component updates, notifications).
2. **Add a "preconditions" section.** Before the E2E chain, list what must be true: user is authenticated, data X exists, previous step Y completed.
3. **Add an "edge cases" section per feature.** Not part of the chain, but a list: what if offline, what if timeout, what if concurrent edit, what if empty data, what if permission denied mid-flow.
4. **Accept that not all gaps need filling at PLAN.** Some edge cases are discovered during DO. The key is having a place to record them and a TEST mechanism to verify them.

**Warning signs:**
- E2E spec has no mention of loading states
- Error section only lists one error type (usually "show error message")
- No preconditions documented
- State management is implicit ("it just updates")
- Features work individually but break when used together

**Phase to address:**
PLAN stage -- E2E spec template must include extended sections. PLAN should not require all sections filled, but must make gaps visible so DO and TEST can address them.

---

### Pitfall 5: Over-Specification -- Prototype Becomes the App, Building Everything Twice

**What goes wrong:**
The prototype starts as a simple clickable mockup. Then the user wants to see error states. Then loading states. Then real data shapes. Then validation. At some point, the prototype is a working application -- and the actual implementation is a second version of the same application. Development time doubles. Worse, the second version inevitably differs from the first, creating confusion about which is "correct."

Research shows this is measurable: one evaluation found spec-driven development taking 33 minutes to produce 2,577 lines of spec for 689 lines of code -- approximately 10x slower than iterative approaches with no quality gain.

**Why it happens:**
1. **Incremental scope creep feels reasonable.** Each addition to the prototype seems small ("just add the error state"). Cumulatively, the prototype becomes a full implementation.
2. **Psychological attachment to working code.** Once the prototype "works," there is pressure to evolve it rather than rebuild from scratch. This pulls the prototype toward production quality.
3. **Unclear boundary between "enough specification" and "actual implementation."** There is no clear line separating "this is a spec" from "this is code."
4. **AI makes generation cheap.** Because Claude can generate both prototype and implementation quickly, the cost of over-specifying feels low. But the cost is not in generation -- it is in maintenance, synchronization, and confusion about source of truth.

**How to avoid:**
1. **Hard constraint: prototype is static HTML+JS with no build step.** Single file, no framework, no bundler, no backend calls. The moment it requires npm install, it has crossed the line.
2. **Time-box prototype creation.** Prototype generation should take < 15 minutes. If it takes longer, the prototype is too detailed.
3. **Prototype is disposable.** Explicitly label it as throwaway. It exists to communicate, not to evolve. After user approval, the prototype is archived and never updated.
4. **E2E spec carries the detail, not the prototype.** Details about error handling, state management, and edge cases live in the E2E spec (text), not in the prototype (code). The prototype shows the happy path visually. The spec covers everything else textually.
5. **No prototype iteration beyond initial feedback.** One round of user feedback, one revision, done. Further refinement happens in the E2E spec, not the prototype.

**Warning signs:**
- Prototype has more than one HTML file
- Prototype includes a package.json or build configuration
- Time spent on prototype exceeds time spent on E2E spec
- User refers to the prototype as "the app"
- Prototype gets updated after DO stage begins

**Phase to address:**
PLAN stage -- skill must enforce prototype constraints (single file, no build step, disposable after approval). The skill template should make these constraints visible.

---

### Pitfall 6: Verification Theater -- Green Checks That Miss Real Failures

**What goes wrong:**
The E2E spec exists. The prototype was approved. TEST stage runs verification. Everything passes. The feature still does not work for the user. This happens because verification checks for structural compliance ("does a handler exist for this button?") but not behavioral correctness ("does clicking this button actually save the data the user entered?").

The user's experience -- "100% of features don't work" -- persists even with prototypes and E2E specs because the verification layer checks the artifacts, not the actual running application.

**Why it happens:**
1. **Claude cannot interact with a running application.** Claude can verify that code exists, that functions are wired, that API routes match. It cannot click a button and see what happens. Verification is static analysis, not dynamic testing.
2. **Wiring checks are necessary but insufficient.** Checking "button has onClick -> onClick calls API -> API exists" proves the chain exists. It does not prove the chain works. The API might return the wrong data shape. The state update might not trigger re-render.
3. **E2E spec verification stops at existence.** "Screen: login button exists. Connection: calls /api/auth. Processing: validates credentials." Each item can be verified as structurally present while being functionally broken.

**How to avoid:**
1. **Generate runnable test commands, not just checklists.** TEST stage should produce actual test commands (curl, playwright, etc.) that the user can execute, not just confirmations that code looks right.
2. **User verification is the real gate.** After Claude's structural verification, the user must manually test the feature. Claude's verification narrows the scope of manual testing (user does not need to check if the button exists -- Claude verified that). But the user must verify the feature works end-to-end.
3. **Distinguish structural verification (Claude can do) from behavioral verification (user must do).** Make this distinction explicit in the TEST stage:
   - **Claude verifies:** Code exists, wired correctly, error handlers present, types match
   - **User verifies:** Feature works as expected when used, UX matches intent, edge cases handled
4. **Include specific manual test steps.** TEST stage should output: "Open the app, go to X page, click Y button, type Z, submit. Expected result: [specific outcome]." This gives the user a script, not just a status.

**Warning signs:**
- TEST stage passes without the user touching the application
- Verification output is all "PASS" with no manual test steps
- No distinction between "verified by Claude" and "verified by user"
- TEST stage takes less than 2 minutes (too fast for real verification)

**Phase to address:**
TEST stage -- must include both automated structural verification (Claude) and manual behavioral verification script (for user). Gate to COMMIT requires both.

---

### Pitfall 7: Incomplete Prototype -- AI's Prototype Itself Has the Same Gaps as AI's Code

**What goes wrong:**
The prototype is supposed to prevent the "done but not done" problem. But if Claude generates the prototype, it may omit the same things it would omit from implementation: error states, empty states, loading states, edge case flows. The prototype looks complete because Claude thinks it is complete -- the same completion bias that causes the original problem.

The user sees a prototype with a form, a submit button, and a success message. The prototype does NOT show: what happens with invalid input, what happens if the form is submitted twice, what the empty state looks like before any data exists, what happens on a slow connection. These omissions are the exact same omissions that cause "features don't work."

**Why it happens:**
1. **Same training, same biases.** Claude's model of "complete UI" is the same whether generating a prototype or production code. If it does not think to include an empty state in code, it will not think to include it in a prototype.
2. **Prototype inherits happy-path bias.** AI systems consistently generate the happy path first and most thoroughly. Error states, edge cases, and boundary conditions are afterthoughts in both prototype and implementation.
3. **No external input on completeness.** Without a checklist of required states/flows, Claude decides what "complete" means -- and its definition is consistently insufficient.

**How to avoid:**
1. **Mandatory state checklist for every prototype screen.** Before generating the prototype, require Claude to enumerate states: empty, loading, error, success, partial, permission-denied, offline. Each state must either be shown in the prototype or explicitly listed as "NOT DEMONSTRATED" with coverage in the E2E spec.
2. **User reviews the state list before the prototype.** Before seeing the prototype, the user reviews: "For this feature, I will show these states: [list]. These states are NOT shown but covered in E2E spec: [list]." User can add missing states before the prototype is even generated.
3. **Prototype review checklist.** After generating prototype, run a structured check: "Does this prototype show error state? Loading state? Empty state? Multi-item state? Disabled state? Unauthorized state?" Force explicit YES/NO for each.

**Warning signs:**
- Prototype has only 1-2 screens (likely missing states)
- No error or empty state shown
- Prototype shows only the "golden path" with ideal data
- User approves quickly without asking "what about when X goes wrong?"

**Phase to address:**
PLAN stage -- prototype generation template must include mandatory state enumeration before generation and a completeness checklist after generation.

---

## Technical Debt Patterns

| Shortcut | Immediate Benefit | Long-term Cost | When Acceptable |
|----------|-------------------|----------------|-----------------|
| Skip prototype, go straight to E2E spec | Saves time on visual artifact | Misalignment on UI expectations; rework at TEST stage | Only for backend-only features with no UI component |
| Skip E2E spec, rely only on prototype | Prototype is more fun to review | Missing error/state/edge case coverage; same "100% failure" problem | Never -- prototype is visual, E2E spec is functional |
| Allow prototype to evolve into implementation | No "rebuild" feeling | Two sources of truth; confusion about which is canonical; maintenance burden | Never -- prototype must be disposable |
| E2E spec without preconditions section | Shorter specs, faster to write | Features that work in isolation but fail when composed (missing auth, missing data, wrong sequence) | MVP only, with plan to add preconditions in next pass |
| Skip user verification in TEST | Faster pipeline | Structural checks pass but feature is behaviorally broken; the original "100% failure" problem persists | Never -- user verification is the entire point |
| Single-round prototype (no user feedback) | Faster PLAN stage | Prototype reflects Claude's assumptions, not user intent; defeats the purpose | Only for features the user has explicitly described in extreme detail |

## Integration Gotchas

| Integration | Common Mistake | Correct Approach |
|-------------|----------------|------------------|
| Prototype + E2E spec | Treating them as alternatives (use one or the other) | They are complementary: prototype = visual contract, E2E spec = functional contract. Both required. |
| Prototype + DO stage | Prototype left open in a tab as "reference" but not formally linked | DO stage must explicitly read the E2E spec file. Prototype is for user alignment, E2E spec is for implementation guidance. |
| E2E spec + TEST stage | TEST checks "does the feature work?" without referencing the spec | TEST must iterate through E2E spec items and mark each PASS/FAIL/DEVIATED |
| User feedback + prototype iteration | Multiple rounds of prototype revision (scope creep) | One feedback round, one revision. Further details go into E2E spec amendments. |
| Deviation log + COMMIT | Deviations recorded during DO but not included in commit context | COMMIT message must reference any deviations from spec with rationale |

## "Looks Done But Isn't" Checklist

Specific to the prototype + E2E spec approach:

- [ ] **Prototype:** Shows error state for at least one interaction -- verify not just the happy path
- [ ] **Prototype:** Shows empty/zero-data state -- verify not just the populated state
- [ ] **Prototype:** Has "NOT DEMONSTRATED" section listing what it does not cover
- [ ] **E2E spec:** Has preconditions for each feature chain -- verify auth, data, and sequence requirements
- [ ] **E2E spec:** Error section lists more than one error type -- verify timeout, validation, permission, and server errors
- [ ] **E2E spec:** Includes state change and side effects -- verify what updates beyond the immediate response
- [ ] **E2E spec:** Loading/transition states documented -- verify what the user sees between action and response
- [ ] **DO stage:** Deviation log exists if implementation differs from spec -- verify no silent changes
- [ ] **TEST stage:** Includes manual test steps for user -- verify not just automated structural checks
- [ ] **TEST stage:** Each E2E spec item has explicit PASS/FAIL/DEVIATED status -- verify line-by-line comparison
- [ ] **User verification:** User has actually used the running feature -- verify not just code review

## Recovery Strategies

| Pitfall | Recovery Cost | Recovery Steps |
|---------|---------------|----------------|
| Self-verification blindness (Claude confirms Claude) | MEDIUM | Re-read original user request (not prototype). Compare user words against implementation. Generate a "misalignment report" listing any interpretation gaps. |
| Prototype too shallow | LOW | Generate the missing state list. Update E2E spec (not prototype) to cover gaps. Add edge cases to TEST verification. |
| Prototype-implementation drift | MEDIUM | Run E2E spec comparison against implementation. For each DEVIATED item, get user approval or fix implementation. |
| E2E format gaps | LOW | Extend affected E2E specs with state-change, side-effects, and preconditions sections. Update TEST checklist. |
| Over-specification (built app twice) | HIGH | Declare prototype frozen. Archive it. Use only E2E spec going forward. Accept the sunk cost. Enforce single-file constraint for future prototypes. |
| Verification theater | MEDIUM | Generate manual test script from E2E spec. User executes test script. Record PASS/FAIL for each step. Fix failures. |
| Incomplete prototype | LOW | Run state checklist (empty/loading/error/success/partial/denied). Add missing states to E2E spec. Prototype stays as-is (disposable). |

## Pitfall-to-Phase Mapping

| Pitfall | Prevention Stage | Verification |
|---------|-----------------|--------------|
| Self-verification blindness | PLAN | User has clicked prototype AND provided feedback (not just "looks good") |
| Prototype too shallow | PLAN | "NOT DEMONSTRATED" section exists; E2E spec covers listed gaps |
| Prototype-implementation drift | DO | Deviation log has entries for any spec departures; no silent changes |
| E2E format gaps | PLAN | E2E spec template includes 7 steps + preconditions + edge cases |
| Over-specification | PLAN | Prototype is single HTML file, no build step, generated in < 15 minutes |
| Verification theater | TEST | Manual test script exists; user has executed it; PASS/FAIL recorded |
| Incomplete prototype | PLAN | State checklist completed before AND after prototype generation |

## What This Approach CANNOT Solve

Honest acknowledgment of limitations:

1. **Performance issues.** Neither prototype nor E2E spec can predict performance bottlenecks. A feature can match the spec perfectly and still be unusably slow.
2. **Cross-feature interactions.** E2E specs cover individual features. When Feature A's state change breaks Feature B, neither spec anticipated it. Integration testing is a separate concern.
3. **Visual/aesthetic quality.** Prototype shows layout and flow but not polish. Production quality CSS, animation timing, responsive behavior -- these are unspecifiable at PLAN stage.
4. **Non-deterministic AI behavior.** The same E2E spec given to Claude twice will produce different implementations. Spec-driven development with AI is inherently non-deterministic. Small increments and frequent verification mitigate this but cannot eliminate it.
5. **Unknown unknowns.** E2E specs cover what we think of. They cannot cover what we do not think of. The prototype+spec approach reduces the gap but cannot close it. The user's "100% failure" rate may improve to 20-30% -- not 0%.
6. **Accumulated context loss.** Even with perfect specs, Claude's implementation quality degrades as the codebase grows and the context window fills. Spec quality does not compensate for context limitations.

## Sources

- [Verification debt: the hidden cost of AI-generated code (Lars Janssen)](https://fazy.medium.com/agentic-coding-ais-adolescence-b0d13452f981) -- verification debt concept, confirmation bias in AI self-verification
- [Why Spec-Driven Development Fails (DEV Community)](https://dev.to/casamia918/why-spec-driven-development-fails-and-what-we-can-learn-from-it-2pec) -- over-specification costs (33 min / 2,577 lines spec for 689 lines code), spec drift
- [Spec-driven development with AI (GitHub Blog)](https://github.blog/ai-and-ml/generative-ai/spec-driven-development-with-ai-get-started-with-a-new-open-source-toolkit/) -- spec-driven development patterns and tools
- [How spec-driven development improves AI coding quality (Red Hat)](https://developers.redhat.com/articles/2025/10/22/how-spec-driven-development-improves-ai-coding-quality) -- SDD benefits and limitations
- [Spec-Driven Development (Thoughtworks)](https://thoughtworks.medium.com/spec-driven-development-d85995a81387) -- industry analysis of SDD patterns
- [Spec-driven development: From Code to Contract (arxiv)](https://arxiv.org/html/2602.00180v1) -- academic analysis of SDD
- [How to write a good spec for AI agents (Addy Osmani)](https://addyosmani.com/blog/good-spec/) -- spec-writing best practices
- [The Limits of Spec-Driven Development (Isoform)](https://isoform.ai/blog/the-limits-of-spec-driven-development) -- SDD limitations
- [Martin Fowler: Understanding Spec-Driven-Development tools](https://martinfowler.com/articles/exploring-gen-ai/sdd-3-tools.html) -- SDD tooling analysis
- [When AI writes the software, who verifies it? (Leo de Moura)](https://leodemoura.github.io/blog/2026/02/28/when-ai-writes-the-worlds-software.html) -- formal verification for AI-generated code
- User experience documented in PROJECT.md: "100% failure rate" on feature completeness

---
*Pitfalls research for: User Interaction Prototype + E2E Feature Spec as Verification Mechanisms*
*Researched: 2026-03-22*
