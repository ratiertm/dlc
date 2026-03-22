# Phase 3: Prototype Format - Research

**Researched:** 2026-03-22
**Domain:** Clickable single-file HTML prototypes with embedded manifest and semantic data attributes
**Confidence:** HIGH

## Summary

Phase 3 defines the prototype format -- the template, reference documentation, and example file for generating clickable HTML prototypes. This mirrors Phase 2's deliverable structure (template + reference + example) but for prototypes instead of specs. The prototype is a single vanilla HTML file with inline CSS/JS, hash-based SPA routing, semantic `data-*` attributes, and an embedded JSON manifest. It opens via `file://` protocol with no server needed.

The technical foundation is already well-researched in STACK.md and ARCHITECTURE.md. The key decisions are locked: vanilla HTML+CSS+JS, single file, hash routing, domain-specific data attributes (`data-screen`, `data-action`, `data-field`, `data-error`), embedded JSON manifest via `<script type="application/json">`, and dual linking to E2E specs via `data-spec-id` attributes. Phase 3's job is to codify these decisions into reusable skill artifacts (template, reference, example).

**Primary recommendation:** Create three files mirroring Phase 2's structure: `skill/templates/prototype-template.html` (boilerplate), `skill/references/prototype.md` (format reference with rules), and `skill/examples/user-login.prototype.html` (working example based on the existing user-login.spec.md).

<user_constraints>
## User Constraints (from CONTEXT.md)

### Locked Decisions
- Minimal UI with responsive design (mobile + desktop)
- Click triggers screen transition + mock data display (not simple alerts)
- Dual spec linking: data-spec-id attributes + JSON manifest both reference spec IDs
- file:// protocol must work (no server required)
- Each UI element gets `data-spec-id="e2e-{feature}-{NNN}"` attribute
- Embedded JSON manifest contains `{specId, element, type}` mapping

### Claude's Discretion
- Error state simulation inclusion (whether to include error UI in prototypes)
- CSS framework usage (vanilla vs lightweight library)
- Prototype boilerplate's concrete HTML structure
- Mock data format and injection method

### Deferred Ideas (OUT OF SCOPE)
None -- discussion stayed within phase scope.
</user_constraints>

<phase_requirements>
## Phase Requirements

| ID | Description | Research Support |
|----|-------------|-----------------|
| PROTO-01 | Single HTML file (vanilla HTML+CSS+JS), opens via file:// | STACK.md confirms: no frameworks, no CDN, inline CSS+JS, universal browser compatibility. Hash routing works with file:// (verified via web search -- History API requires server, hash does not). |
| PROTO-02 | Hash-based SPA routing for multi-screen navigation | hashchange event is native, no library needed. `location.hash.slice(1)` extracts screen name. Show/hide `[data-screen]` divs. Confirmed working with file:// protocol. |
| PROTO-03 | data-* attributes (data-screen, data-action, data-field, data-error) for semantic UI elements | HTML5 data-* attributes are stable standard. Domain-specific names carry meaning (screen vs action vs field). `document.querySelectorAll('[data-screen]')` enables machine parsing. |
| PROTO-04 | Embedded JSON manifest for machine-parseable prototype structure | `<script type="application/json" id="prototype-manifest">` is valid HTML5, not executed by browser, parseable by any JSON tool. Schema defined in STACK.md. |
</phase_requirements>

## Standard Stack

### Core
| Technology | Version | Purpose | Why Standard |
|------------|---------|---------|--------------|
| Vanilla HTML5 | Living Standard | Prototype structure | Zero dependencies, file:// compatible, Claude generates reliably |
| Inline CSS3 | Living Standard | Wireframe styling | No external files to break, responsive via media queries |
| Vanilla ES6+ JavaScript | ES2015+ | Hash routing + interaction simulation | Native hashchange event, querySelectorAll for data-* parsing |
| JSON (embedded) | RFC 8259 | Machine-readable manifest | `<script type="application/json">` standard pattern |

### Supporting
| Technology | Purpose | When to Use |
|------------|---------|-------------|
| system-ui font stack | Cross-platform consistent typography | Always -- avoids web font CDN dependency |
| CSS custom properties | Theming (optional) | If prototype needs light/dark mode simulation |
| `<meta name="viewport">` | Mobile responsive | Always -- locked decision: mobile + desktop |

### Alternatives Considered
| Instead of | Could Use | Tradeoff |
|------------|-----------|----------|
| Vanilla CSS | Lightweight CSS (e.g., MVP.css, Pico.css via CDN) | CDN breaks file:// offline. Could inline a micro-framework (~5KB) but adds maintenance burden. Recommendation: **stay vanilla** -- wireframe styling is ~30 lines of CSS. |
| Inline everything | Separate .css/.js files | Breaks atomic single-file constraint. Never do this. |
| Hash routing | History API (pushState) | History API requires web server. Fails with file://. Never use for prototypes. |

**No installation needed.** This phase produces Markdown reference docs and HTML template files. No npm packages.

## Architecture Patterns

### Recommended Deliverable Structure
```
skill/
  templates/
    prototype-template.html    # NEW: Boilerplate HTML with placeholders
  references/
    prototype.md               # NEW: Format rules, data attribute conventions
  examples/
    user-login.prototype.html  # NEW: Working example matching user-login.spec.md
```

Runtime deployment mirrors Phase 2 pattern:
```
~/.claude/skills/dev-lifecycle/
  templates/prototype-template.html
  references/prototype.md
  examples/user-login.prototype.html
```

Project-level output location:
```
.lifecycle/features/{feature-name}/
  spec.md              # From Phase 2
  prototype.html       # Generated from template during PLAN stage
```

### Pattern 1: Four-Section HTML Structure
**What:** Every prototype HTML file has four clearly delimited sections: HEAD (meta + style), MANIFEST (embedded JSON), SCREENS (data-screen divs), ENGINE (routing + interaction JS).
**When to use:** Every prototype file.

```html
<!DOCTYPE html>
<html lang="en">
<head>
  <!-- SECTION 1: HEAD — meta + responsive + inline CSS -->
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title>{Feature Name} - Prototype</title>
  <style>/* wireframe CSS */</style>
</head>
<body>
  <!-- SECTION 2: MANIFEST — machine-readable JSON -->
  <script type="application/json" id="prototype-manifest">
  { "feature": "...", "specFile": "spec.md", ... }
  </script>

  <!-- SECTION 3: SCREENS — data-screen divs -->
  <div id="app">
    <div data-screen="screen-name" class="screen">...</div>
  </div>

  <!-- SECTION 4: ENGINE — routing + interaction logic -->
  <script>
    // Hash router, action handlers, mock data
  </script>
</body>
</html>
```

### Pattern 2: Dual Spec Linking (data-spec-id + manifest)
**What:** Every interactive UI element has both a `data-spec-id` attribute in the HTML AND a corresponding entry in the JSON manifest. This dual linkage ensures neither human (browsing HTML) nor machine (parsing JSON) misses a connection.
**When to use:** Every prototype element that maps to a spec step.

```html
<!-- In HTML (human-readable): -->
<form data-spec-id="e2e-login-001" data-screen="login">
  <input data-spec-id="e2e-login-001" data-field="email" type="email">
  <button data-spec-id="e2e-login-002" data-action="submit-login">Login</button>
</form>

<!-- In manifest (machine-readable): -->
{
  "specMapping": [
    {"specId": "e2e-login-001", "element": "[data-screen='login']", "type": "Screen"},
    {"specId": "e2e-login-002", "element": "[data-action='submit-login']", "type": "Connection"}
  ]
}
```

### Pattern 3: Screen-as-Div with Hash Router
**What:** Each "screen" is a `<div data-screen="name">` hidden by default. The hash router shows/hides screens based on `location.hash`. Default screen is the first one.
**When to use:** Every multi-screen prototype.

```javascript
function showScreen(screenId) {
  document.querySelectorAll('[data-screen]').forEach(s => s.style.display = 'none');
  const target = document.querySelector(`[data-screen="${screenId}"]`);
  if (target) target.style.display = 'block';
}

window.addEventListener('hashchange', () => {
  showScreen(location.hash.slice(1) || getDefaultScreen());
});

// Initialize on load
showScreen(location.hash.slice(1) || getDefaultScreen());
```

### Pattern 4: Mock Data Injection
**What:** Prototypes show realistic-looking data (names, emails, counts) without real API calls. Mock data is defined inline in the JS section or within a `<script type="application/json" id="mock-data">` block.
**When to use:** When prototype needs to display dynamic content (user names, lists, counts).

```javascript
const MOCK = {
  user: { name: "Alice Kim", email: "alice@example.com" },
  items: [
    { id: 1, title: "First item", status: "active" },
    { id: 2, title: "Second item", status: "completed" }
  ]
};

function populateScreen(screenId) {
  const screen = document.querySelector(`[data-screen="${screenId}"]`);
  screen.querySelectorAll('[data-mock]').forEach(el => {
    const path = el.dataset.mock; // e.g., "user.name"
    el.textContent = resolvePath(MOCK, path);
  });
}
```

### Anti-Patterns to Avoid
- **Over-styling:** Prototype is for flow validation, not design. System fonts, border-box, neutral colors only. No gradients, shadows, or brand colors.
- **External dependencies:** No CDN links, no external CSS/JS files. Everything inline.
- **History API routing:** Breaks on file://. Always use hash routing.
- **Alert-based interactions:** User locked decision says "not simple alerts." Use screen transitions and mock data display instead.
- **Separate manifest file:** Manifest MUST be embedded in the HTML. Separate file can drift out of sync.

## Don't Hand-Roll

| Problem | Don't Build | Use Instead | Why |
|---------|-------------|-------------|-----|
| Responsive layout | Custom media query system | Simple CSS: `max-width: 480px; margin: 0 auto` + one `@media` breakpoint | Two viewports (mobile/desktop) need 3 lines, not a framework |
| Form validation UX | Custom validator | HTML5 `required`, `type="email"`, `pattern` attributes + simple JS check | Browser native validation covers 90% of prototype needs |
| Screen transitions | Animation library | `display: none/block` toggle | Prototypes validate flow, not motion design |
| JSON manifest parsing | Custom parser | `JSON.parse(document.getElementById('prototype-manifest').textContent)` | One-liner, standards-compliant |
| Spec ID extraction | Custom HTML parser | `document.querySelectorAll('[data-spec-id]')` | Native DOM API |

**Key insight:** Prototypes are deliberately simple. The complexity is in the FORMAT DEFINITION (what goes where, what attributes mean, how manifest maps to spec), not in the code.

## Common Pitfalls

### Pitfall 1: Prototype That Doesn't Match Spec
**What goes wrong:** Prototype shows 4 screens but spec defines 5 interactions. Or prototype has a button not mentioned in any spec step.
**Why it happens:** Prototype and spec are created separately without cross-checking.
**How to avoid:** The template and reference must enforce a rule: every `data-spec-id` in the prototype MUST have a matching `e2e-{feature}-{NNN}` in the spec file. The manifest's `specMapping` array is the enforcement mechanism.
**Warning signs:** Manifest `specMapping` count differs from spec step count.

### Pitfall 2: Hash Routing Not Working on First Load
**What goes wrong:** User opens prototype.html and sees blank page because no screen is shown.
**Why it happens:** Initial load has no hash (`location.hash` is empty). Router waits for `hashchange` event which never fires.
**How to avoid:** Always call `showScreen(defaultScreen)` on page load, not just on hashchange. Template must include initialization code.
**Warning signs:** Prototype requires user to click a link before anything shows.

### Pitfall 3: file:// CORS Restrictions
**What goes wrong:** Prototype tries to load external file (JSON, CSS) and browser blocks it.
**Why it happens:** file:// protocol has strict CORS. Fetch/XMLHttpRequest to local files is blocked in most browsers.
**How to avoid:** EVERYTHING inline. No `fetch()` calls. No `<link>` to external CSS. No `<script src>`. The single-file constraint solves this completely.
**Warning signs:** Console errors about CORS or cross-origin when opening with file://.

### Pitfall 4: data-spec-id on Wrong Granularity
**What goes wrong:** Putting `data-spec-id` on a container div that wraps multiple spec steps, or on individual characters.
**Why it happens:** Unclear granularity rules for data attribute placement.
**How to avoid:** One `data-spec-id` per spec step. Screen step -> `data-spec-id` on the screen container div. Connection step -> `data-spec-id` on the clickable element (button/link). Error step -> `data-spec-id` on the error display element.
**Warning signs:** Multiple spec IDs on one element, or one spec ID on a generic wrapper.

### Pitfall 5: Manifest Schema Drift
**What goes wrong:** Manifest has different field names across different prototypes (e.g., `specId` in one, `spec_id` in another, `spec-id` in a third).
**Why it happens:** No enforced schema; each prototype is hand-written.
**How to avoid:** Template defines the canonical manifest schema. Reference doc specifies exact field names. Example demonstrates the schema in practice.
**Warning signs:** Verification script breaks on some prototypes but not others.

## Code Examples

### Complete Minimal Prototype (verified pattern from STACK.md + ARCHITECTURE.md)

```html
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title>{Feature Name} - Prototype</title>
  <style>
    * { box-sizing: border-box; margin: 0; padding: 0; }
    body { font-family: system-ui, -apple-system, sans-serif; background: #f5f5f5; padding: 16px; }
    #app { max-width: 480px; margin: 0 auto; }
    .screen { display: none; background: #fff; border: 1px solid #ddd; border-radius: 8px; padding: 24px; }
    .screen.active { display: block; }
    h2 { margin-bottom: 16px; color: #333; }
    input, select, textarea { width: 100%; padding: 10px; border: 1px solid #ccc; border-radius: 4px; margin-bottom: 12px; font-size: 14px; }
    button, .btn { display: inline-block; padding: 10px 20px; background: #2563eb; color: #fff; border: none; border-radius: 4px; cursor: pointer; font-size: 14px; text-decoration: none; }
    button:hover, .btn:hover { background: #1d4ed8; }
    .error { background: #fef2f2; border: 1px solid #fca5a5; color: #dc2626; padding: 12px; border-radius: 4px; margin-bottom: 12px; display: none; }
    .success { background: #f0fdf4; border: 1px solid #86efac; color: #16a34a; padding: 12px; border-radius: 4px; }
    .field-group { margin-bottom: 12px; }
    .field-group label { display: block; margin-bottom: 4px; font-weight: 600; color: #555; font-size: 13px; }
    .nav-link { color: #2563eb; text-decoration: underline; cursor: pointer; font-size: 14px; }
    @media (min-width: 768px) { #app { max-width: 640px; } }
  </style>
</head>
<body>

  <!-- ============================================================
       PROTOTYPE MANIFEST (machine-readable, not rendered)
       ============================================================ -->
  <script type="application/json" id="prototype-manifest">
  {
    "feature": "{feature-name}",
    "specFile": "spec.md",
    "version": "1",
    "generatedFrom": "e2e-{feature} spec",
    "screens": ["{screen-1}", "{screen-2}"],
    "specMapping": [
      {"specId": "e2e-{feature}-001", "element": "[data-screen='{screen-1}']", "type": "Screen"},
      {"specId": "e2e-{feature}-002", "element": "[data-action='{action-1}']", "type": "Connection"}
    ],
    "interactions": [
      {"screen": "{screen-1}", "element": "[data-action='{action-1}']", "action": "click", "result": "navigate", "target": "#{screen-2}"}
    ],
    "fields": [
      {"screen": "{screen-1}", "element": "[data-field='{field-1}']", "type": "email", "required": true}
    ],
    "errorStates": [
      {"screen": "{screen-1}", "trigger": "{error-condition}", "element": "[data-error='{error-name}']", "message": "{error message}"}
    ]
  }
  </script>

  <!-- ============================================================
       SCREENS
       ============================================================ -->
  <div id="app">
    <div data-screen="{screen-1}" data-spec-id="e2e-{feature}-001" class="screen">
      <!-- Screen content with data-field, data-action attributes -->
    </div>
    <div data-screen="{screen-2}" class="screen">
      <!-- Next screen -->
    </div>
  </div>

  <!-- ============================================================
       INTERACTION ENGINE
       ============================================================ -->
  <script>
    // --- Hash Router ---
    function showScreen(id) {
      document.querySelectorAll('.screen').forEach(s => s.classList.remove('active'));
      const target = document.querySelector(`[data-screen="${id}"]`);
      if (target) target.classList.add('active');
    }

    function getDefaultScreen() {
      const manifest = JSON.parse(document.getElementById('prototype-manifest').textContent);
      return manifest.screens[0];
    }

    window.addEventListener('hashchange', () => {
      showScreen(location.hash.slice(1) || getDefaultScreen());
    });

    // --- Action Handlers ---
    document.addEventListener('click', (e) => {
      const actionEl = e.target.closest('[data-action]');
      if (actionEl) {
        e.preventDefault();
        handleAction(actionEl.dataset.action, actionEl);
      }
    });

    function handleAction(action, element) {
      // Feature-specific action logic goes here
    }

    // --- Initialize ---
    showScreen(location.hash.slice(1) || getDefaultScreen());
  </script>

</body>
</html>
```

### Data Attribute Convention Table (from STACK.md, verified)

| Attribute | Purpose | Placement | Example |
|-----------|---------|-----------|---------|
| `data-screen="name"` | Identifies a screen/view | Container div | `<div data-screen="login">` |
| `data-action="name"` | Identifies clickable interaction | Button, link, clickable element | `<button data-action="submit-login">` |
| `data-field="name"` | Identifies form input | Input, select, textarea | `<input data-field="email">` |
| `data-error="name"` | Identifies error display | Error message container | `<div data-error="invalid-credentials">` |
| `data-spec-id="e2e-..."` | Links element to spec step | Any element mapping to a spec step | `<form data-spec-id="e2e-login-001">` |
| `data-nav="target"` | Navigation target hint | Navigation links | `<a data-nav="dashboard">` |
| `data-state="name"` | Dynamic state container | Loading spinners, toggles | `<div data-state="loading">` |
| `data-mock="path"` | Mock data binding | Text display elements | `<span data-mock="user.name">` |

### Manifest Schema (canonical, from STACK.md)

```json
{
  "feature": "string -- matches spec feature name (kebab-case)",
  "specFile": "string -- relative path to spec.md",
  "version": "string -- increment on prototype revision",
  "generatedFrom": "string -- description of source spec",
  "screens": ["string[] -- all screen names, order = navigation order"],
  "specMapping": [
    {
      "specId": "string -- e2e-{feature}-{NNN}",
      "element": "string -- CSS selector using data attributes",
      "type": "Screen | Connection | Processing | Response | Error"
    }
  ],
  "interactions": [
    {
      "screen": "string -- source screen name",
      "element": "string -- CSS selector for interactive element",
      "action": "click | submit | input | hover",
      "condition": "string? -- optional precondition",
      "result": "navigate | show | hide | validate | submit",
      "target": "string? -- target screen or element"
    }
  ],
  "fields": [
    {
      "screen": "string -- which screen",
      "element": "string -- CSS selector",
      "type": "text | email | password | number | select | checkbox | radio | textarea",
      "required": "boolean",
      "validation": "string? -- validation rule name"
    }
  ],
  "errorStates": [
    {
      "screen": "string -- which screen shows error",
      "trigger": "string -- what causes this error",
      "element": "string -- CSS selector for error display",
      "message": "string -- error text shown to user"
    }
  ]
}
```

## Claude's Discretion Recommendations

### Error State Simulation: INCLUDE
**Recommendation:** Include error states in the prototype template and example. Rationale:
- The E2E spec's 5th step is explicitly "Error" -- every spec has error conditions
- The user-login.spec.md example has 6 error conditions (empty fields, invalid format, invalid credentials, server error, network timeout, rate limit)
- Error flows are where "done but broken" most frequently occurs
- Implementation: error screens are just additional `data-screen` divs or inline `data-error` elements shown/hidden via JS

### CSS Framework: STAY VANILLA
**Recommendation:** No CSS framework. Use inline wireframe CSS (~30 lines). Rationale:
- CDN dependency violates file:// constraint
- Inlining a micro-framework adds 5-15KB for minimal benefit
- Wireframe styling needs: box model, form inputs, buttons, error/success colors, responsive max-width
- These are ~30 lines of CSS, not worth a framework

### Mock Data Format: INLINE JS OBJECT
**Recommendation:** Define mock data as a `const MOCK = {...}` in the script section. Rationale:
- Simplest approach -- no parsing needed
- Can be referenced by `data-mock` attributes on display elements
- For the template, provide a pattern but leave data to be filled per-feature
- Alternative (separate `<script type="application/json" id="mock-data">`) adds unnecessary parsing step

### Prototype HTML Structure: FOUR-SECTION LAYOUT
**Recommendation:** As described in Pattern 1 above. HEAD / MANIFEST / SCREENS / ENGINE. Clear comment delimiters between sections. This structure is predictable for both Claude (generation) and verification scripts (parsing).

## State of the Art

| Old Approach | Current Approach | When Changed | Impact |
|--------------|------------------|--------------|--------|
| Multi-file prototypes (separate HTML/CSS/JS) | Single-file with inline everything | Industry trend toward self-contained artifacts | Eliminates broken references, simpler to share |
| Generic `data-testid` | Domain-specific data attributes (`data-screen`, `data-action`) | Testing Library pattern evolved to semantic naming | Each attribute type carries meaning, enables smarter machine parsing |
| Manual clickthrough documentation | Embedded JSON manifest | Innovation in this project | Machine-parseable structure enables automated verification at TEST stage |
| Figma/design-tool prototypes | Code-based clickable HTML | Growing trend for developer-centric teams | No external tools, version-controllable, machine-readable |

## Open Questions

1. **Processing step representation in prototype**
   - What we know: Screen, Connection, Response, Error steps map directly to visible UI elements. Processing is server-side.
   - What's unclear: Should the prototype show a "processing simulation" (loading state, spinner) or just skip from action to result?
   - Recommendation: Show a brief loading state (`data-state="loading"`) that auto-transitions to the result screen. This validates the user's mental model of "something is happening."

2. **Spec ID sharing between container and children**
   - What we know: `data-spec-id="e2e-login-001"` goes on the Screen container. But the fields inside that screen (email, password) are part of the same spec step.
   - What's unclear: Should child elements also carry the parent's spec-id, or rely on DOM hierarchy?
   - Recommendation: Only the primary element gets `data-spec-id`. Children inherit context via DOM hierarchy. The manifest `specMapping` maps the spec ID to the container selector. Avoids redundant attributes.

3. **Multiple interactions per screen**
   - What we know: A login screen has both "submit" (Connection) and "go to register" (navigation) as separate interactions.
   - What's unclear: How to handle screen reuse across multiple spec interaction chains.
   - Recommendation: Each interaction element gets its own `data-spec-id`. The screen container gets the Screen step's spec-id. Navigation links within a screen get their own spec IDs from other interaction chains. The manifest tracks all of them.

## Validation Architecture

### Test Framework
| Property | Value |
|----------|-------|
| Framework | Manual browser validation + manifest schema check |
| Config file | None -- prototypes are standalone HTML files |
| Quick run command | `open prototype.html` (macOS) / browser file:// open |
| Full suite command | Parse manifest JSON, validate against spec step IDs |

### Phase Requirements to Test Map
| Req ID | Behavior | Test Type | Automated Command | File Exists? |
|--------|----------|-----------|-------------------|-------------|
| PROTO-01 | Single HTML file opens in browser via file:// | manual | `open skill/examples/user-login.prototype.html` | Wave 0 |
| PROTO-02 | Hash routing switches between screens | manual | Open file, click links, verify hash changes | Wave 0 |
| PROTO-03 | data-* attributes present on semantic elements | smoke | `grep -c 'data-screen\|data-action\|data-field\|data-error' skill/examples/user-login.prototype.html` | Wave 0 |
| PROTO-04 | JSON manifest parseable and contains required fields | smoke | `node -e "const h=require('fs').readFileSync('skill/examples/user-login.prototype.html','utf8'); const m=h.match(/<script type=\"application\/json\" id=\"prototype-manifest\">([\s\S]*?)<\/script>/); const j=JSON.parse(m[1]); console.log(j.feature, j.screens.length, j.specMapping.length)"` | Wave 0 |

### Sampling Rate
- **Per task commit:** Visual check: open prototype in browser, verify screens switch
- **Per wave merge:** Parse manifest, cross-check specMapping IDs against spec.md step IDs
- **Phase gate:** All four PROTO requirements verified

### Wave 0 Gaps
- [ ] `skill/templates/prototype-template.html` -- boilerplate to be created
- [ ] `skill/references/prototype.md` -- format reference to be created
- [ ] `skill/examples/user-login.prototype.html` -- working example to be created

## Sources

### Primary (HIGH confidence)
- `.planning/research/STACK.md` -- Comprehensive stack decisions: vanilla HTML, hash routing, data attributes, manifest schema, CSS approach. Directly applicable.
- `.planning/research/ARCHITECTURE.md` -- Prototype-spec integration architecture: dual-artifact pattern, data-spec-id linking, verification flow. Directly applicable.
- `skill/references/e2e-spec.md` -- Cross-Artifact Linking table defining how spec IDs appear in prototype HTML (`data-spec-id` attribute). Phase 2 output.
- `skill/examples/user-login.spec.md` -- Complete 5-step spec example. Prototype example MUST map to this spec's steps (e2e-login-001 through e2e-login-005).
- `skill/templates/spec-template.md` -- Spec template structure. Prototype template mirrors this pattern.

### Secondary (MEDIUM confidence)
- [DEV.to: SPA Routing Using Hash](https://dev.to/thedevdrawer/single-page-application-routing-using-hash-or-url-9jh) -- Confirms hash routing works with file:// protocol, provides implementation patterns
- [DEV.to: Data Attributes for E2E Tests](https://dev.to/marcostreng/how-to-add-data-attributes-to-your-component-library-and-benefit-from-them-in-your-e2e-tests-148l) -- Confirms data-* attribute patterns for semantic element identification
- [FreeCodeCamp: Interactive HTML Prototypes](https://www.freecodecamp.org/news/how-to-create-interactive-html-prototypes/) -- Vanilla HTML prototype patterns
- [Best Practices for Custom Data Attributes](https://sqlpey.com/javascript/best-practices-custom-html-data/) -- Confirms data-* safety (reserved by spec, no conflicts)
- [22 HTML data-* Attributes Best Practices](https://freefrontend.com/html-data-attributes/) -- Comprehensive data attribute patterns

### Tertiary (LOW confidence)
None -- all findings verified with primary or secondary sources.

## Metadata

**Confidence breakdown:**
- Standard stack: HIGH -- All decisions locked in STACK.md with extensive rationale. No frameworks, no dependencies. Pure web standards.
- Architecture: HIGH -- Four-section HTML structure, dual linking pattern, and manifest schema are fully defined in STACK.md and ARCHITECTURE.md. Phase 3 codifies existing decisions.
- Pitfalls: HIGH -- Well-documented from STACK.md (file:// CORS, History API incompatibility) and ARCHITECTURE.md anti-patterns (prototype without spec, skipping chain links).
- Discretion items: MEDIUM -- Recommendations for error states, CSS, mock data format are reasoned but not user-validated.

**Research date:** 2026-03-22
**Valid until:** 2026-04-22 (stable -- web standards, no fast-moving dependencies)
