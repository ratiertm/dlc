# Prototype Format Reference

This document defines the single-file HTML prototype format used in the dev-lifecycle skill. Every prototype is a self-contained, clickable HTML file that validates feature flow before implementation begins.

## Purpose

Prototypes exist for **flow validation, not design**. They answer one question: "Does the user journey through this feature make sense?" A prototype is linked to its E2E spec via `data-spec-id` attributes, creating a traceable chain from spec step to clickable UI element.

**Relationship to E2E Spec:**
- The E2E spec defines *what* happens at each step (Screen, Connection, Processing, Response, Error)
- The prototype shows *how it looks and feels* to the user as they move through those steps
- Every prototype element that represents a spec step carries a `data-spec-id` attribute matching the spec's step ID

**When prototypes are created:** During the PLAN stage, after the E2E spec is agreed. The prototype is the second output of the PLAN stage.

**When prototypes are used:** During the TEST stage, to verify that the implementation matches the expected user flow. The embedded JSON manifest enables machine-parseable verification.

## Four-Section Layout

Every prototype HTML file has four clearly delimited sections, marked by HTML comments:

### SECTION 1: HEAD

Meta tags and inline CSS. Contains `<meta charset="UTF-8">`, `<meta name="viewport">`, a descriptive `<title>`, and a `<style>` block with wireframe CSS.

**CSS rules:**
- Use `system-ui, -apple-system, sans-serif` font stack (no web fonts)
- Mobile-first: `max-width: 480px` on `#app`, expanded to `640px` at `min-width: 768px`
- Neutral colors only: `#f5f5f5` background, `#fff` cards, `#333` text, `#2563eb` buttons
- No gradients, shadows, or brand colors
- Screens hidden by default (`.screen { display: none }`), shown via `.screen.active { display: block }`

### SECTION 2: MANIFEST

A `<script type="application/json" id="prototype-manifest">` block containing the machine-readable manifest. This JSON is not executed by the browser. It describes the prototype's structure for verification tools.

See "Manifest Schema" section below for field definitions.

### SECTION 3: SCREENS

A `<div id="app">` container holding one `<div data-screen="name" class="screen">` per screen. Each screen div contains the UI elements for that view: headings, forms, buttons, error messages, success messages, navigation links.

All interactive and semantic elements use `data-*` attributes (see "Data Attribute Conventions" below).

### SECTION 4: ENGINE

A `<script>` block containing:
- Hash router (`showScreen` function, `hashchange` listener)
- Delegated click handler for `[data-action]` elements
- `handleAction` function with feature-specific logic
- Mock data object and `populateScreen` function
- Initialization call (`showScreen` on page load)

## Data Attribute Conventions

| Attribute | Purpose | Placement | Example |
|-----------|---------|-----------|---------|
| `data-screen="name"` | Identifies a screen/view | Container div for each screen | `<div data-screen="login">` |
| `data-action="name"` | Identifies a clickable interaction | Button, link, clickable element | `<button data-action="submit-login">` |
| `data-field="name"` | Identifies a form input | Input, select, textarea | `<input data-field="email">` |
| `data-error="name"` | Identifies an error display | Error message container | `<div data-error="invalid-credentials">` |
| `data-spec-id="e2e-..."` | Links element to spec step | Any element mapping to a spec step | `<div data-spec-id="e2e-login-001">` |
| `data-nav="target"` | Navigation target hint | Navigation links | `<a data-nav="dashboard">` |
| `data-state="name"` | Dynamic state indicator | Loading spinners, toggles | `<div data-state="loading">` |
| `data-mock="path"` | Mock data binding | Text display elements | `<span data-mock="user.name">` |

**Naming convention:** All attribute values use kebab-case (e.g., `submit-login`, `invalid-credentials`, `user-login`).

## Manifest Schema

The embedded JSON manifest has this structure:

```json
{
  "feature": "string -- kebab-case feature name, matches spec",
  "specFile": "string -- filename of the linked spec (e.g., user-login.spec.md)",
  "version": "string -- increment on prototype revision",
  "generatedFrom": "string -- description of source spec",
  "screens": ["string[] -- screen names in navigation order"],
  "specMapping": [
    {
      "specId": "string -- e2e-{feature}-{NNN}",
      "element": "string -- CSS selector using data attributes",
      "type": "Screen | Connection | Processing | Response | Error"
    }
  ],
  "interactions": [
    {
      "screen": "string -- source screen",
      "element": "string -- CSS selector for interactive element",
      "action": "click | submit | input | hover",
      "result": "navigate | show | hide | validate | submit",
      "target": "string? -- target screen or element"
    }
  ],
  "fields": [
    {
      "screen": "string -- which screen",
      "element": "string -- CSS selector",
      "type": "text | email | password | number | select | checkbox | radio | textarea",
      "required": "boolean"
    }
  ],
  "errorStates": [
    {
      "screen": "string -- which screen shows the error",
      "trigger": "string -- what causes this error",
      "element": "string -- CSS selector for error display",
      "message": "string -- error text shown to user"
    }
  ]
}
```

**Required fields:** `feature`, `specFile`, `version`, `screens`, `specMapping`. All other fields are required when applicable (interactions if clickable elements exist, fields if form inputs exist, errorStates if error displays exist).

## Spec Linking Rules

Prototypes use **dual linking** to connect to E2E specs:

1. **HTML attribute:** `data-spec-id="e2e-{feature}-{NNN}"` on the relevant DOM element
2. **Manifest entry:** Corresponding object in the `specMapping` array

Both must be present for every spec step that has a visual representation in the prototype.

**Granularity rules for data-spec-id placement:**

| Spec Step Type | Place data-spec-id On | Example |
|---------------|----------------------|---------|
| Screen | Screen container div | `<div data-screen="login" data-spec-id="e2e-login-001">` |
| Connection | Clickable element (button/link) | `<button data-action="submit" data-spec-id="e2e-login-002">` |
| Processing | Loading/processing state element | `<div data-state="loading" data-spec-id="e2e-login-003">` |
| Response | Result screen container | `<div data-screen="dashboard" data-spec-id="e2e-login-004">` |
| Error | Error display element | `<div data-error="invalid-creds" data-spec-id="e2e-login-005">` |

**One data-spec-id per spec step.** Child elements of a spec-id container inherit context via DOM hierarchy -- do not repeat the spec-id on children.

**Processing step:** Even though processing is server-side, the prototype shows a brief loading state (`data-state="loading"`) that auto-transitions to the result. This validates the user's mental model that "something is happening."

## Hash Routing Rules

- Use `hashchange` event listener (native, no library)
- Extract screen name: `location.hash.slice(1)`
- Show/hide screens by toggling `.active` class on `[data-screen]` divs
- **Always call `showScreen(defaultScreen)` on page load** -- do not wait for `hashchange` (which never fires on first load)
- Default screen is the first entry in `manifest.screens[]`
- Navigation between screens: set `location.hash = '#screen-name'` or call `showScreen()` directly

## Anti-Patterns

| Anti-Pattern | Why It Is Wrong | Correct Approach |
|-------------|----------------|------------------|
| Over-styling (gradients, shadows, brand colors) | Prototype is for flow validation, not design | Neutral wireframe colors, system fonts |
| External dependencies (CDN links, `<link>`, `<script src>`) | Breaks file:// protocol, adds failure points | Everything inline in one file |
| History API routing (`pushState`) | Requires web server, fails with file:// | Hash routing (`hashchange` event) |
| Alert-based interactions (`alert()`, `confirm()`) | Does not validate real UI flow | Screen transitions and mock data display |
| Separate manifest file | Can drift out of sync with HTML | Manifest embedded in `<script type="application/json">` |
| data-spec-id on wrong granularity | Multiple spec IDs on one element, or ID on generic wrapper | One data-spec-id per spec step on the most specific element |
| Fetch/XMLHttpRequest calls | Blocked by CORS on file:// | Inline mock data, no network calls |
| CSS frameworks (Bootstrap, Tailwind CDN) | External dependency, over-styled | ~30 lines inline wireframe CSS |

## File Locations

| File | Path | Purpose |
|------|------|---------|
| Prototype template | `skill/templates/prototype-template.html` | Boilerplate with placeholders |
| This reference | `skill/references/prototype.md` | Format rules and conventions |
| Example prototype | `skill/examples/user-login.prototype.html` | Working login flow example |
| Project output | `.lifecycle/features/{name}/prototype.html` | Generated during PLAN stage |

Runtime deployment: All skill files are copied to `~/.claude/skills/dev-lifecycle/` for Claude access.
