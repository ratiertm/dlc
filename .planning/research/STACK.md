# Stack Research

**Domain:** Claude Code skill-based development lifecycle orchestrator
**Researched:** 2026-03-22
**Confidence:** HIGH

## Recommended Stack

This project builds Claude Code skills (SKILL.md files), not a traditional software application. The "stack" is the set of patterns, file formats, and structural conventions that make Claude Code skills effective orchestrators. No npm packages or frameworks are needed -- everything runs as markdown instructions interpreted by Claude Code, with optional shell scripts and a Node.js CLI utility.

### Core Technologies

| Technology | Version | Purpose | Why Recommended |
|------------|---------|---------|-----------------|
| Claude Code Skills (SKILL.md) | Agent Skills 1.0 spec | Primary orchestration mechanism | Official, cross-platform standard (works in Claude Code, Codex, Copilot, Cursor). YAML frontmatter + markdown body = zero infrastructure. Auto-invocation via description matching eliminates need for explicit triggers. Confidence: HIGH (verified via official docs) |
| Claude Code Hooks | Settings-based | Deterministic lifecycle automation | Hooks fire every time (not probabilistic like skills). Essential for enforcing stage gates, auto-formatting, context re-injection after compaction. 20+ lifecycle events available. Confidence: HIGH (verified via official docs) |
| Claude Code Subagents | .claude/agents/ | Parallel task execution and delegation | Wave-based execution pattern proven by GSD. Each subagent gets fresh context (200k). Supports model routing (`haiku` for exploration, `sonnet` for execution, `opus` for planning). Confidence: HIGH (verified via official docs) |
| Node.js CLI (gsd-tools.cjs) | Node 18+ | State management, config parsing, git operations | GSD pattern: centralizes 50+ repetitive bash patterns into one CLI. Single-file CJS format for zero-dependency distribution. Handles: state load/save, phase operations, frontmatter CRUD, roadmap parsing, progress tracking. Confidence: HIGH (verified via existing GSD codebase) |
| Markdown with YAML Frontmatter | CommonMark + YAML | Structured artifact format | Every planning artifact (STATE.md, ROADMAP.md, PLAN.md, SUMMARY.md) uses this pattern. Frontmatter = machine-readable metadata, body = human-readable content. Claude Code natively parses both. Confidence: HIGH |
| JSON Config | .planning/config.json | Project-specific settings | GSD pattern: mode, granularity, parallelization, model_profile, workflow toggles. Read by CLI, consumed by all workflows. Simple, no schema validation needed -- Claude interprets flexibly. Confidence: HIGH |

### Supporting Libraries

| Library | Version | Purpose | When to Use |
|---------|---------|---------|-------------|
| jq | System | JSON parsing in hook scripts | When hooks need to extract fields from stdin JSON (tool_input.file_path, command, etc.). Required for PreToolUse/PostToolUse hooks. |
| Git | System | Version control + state persistence | Commits = checkpoints. GSD pattern: atomic commits per task, planning docs tracked in git, branch strategies (none/phase/milestone). |
| Bash/Zsh | System | Hook command execution, dynamic context injection | Shell scripts for file validation, formatting, notifications. `!`backtick`` syntax in SKILL.md for pre-processing. |

### Development Tools

| Tool | Purpose | Notes |
|------|---------|-------|
| `/hooks` browser | View/debug configured hooks | Read-only in Claude Code. Edit settings.json directly to modify. |
| `/agents` command | Manage subagent configurations | Interactive creation, but manual file editing preferred for version control. |
| `$CLAUDE_SKILL_DIR` | Reference skill-bundled files | Use in SKILL.md to reference scripts/templates regardless of cwd. |
| `/context` command | Check skill loading budget | Skills share 2% of context window (16K chars fallback). Monitor when skills multiply. |

---

## User Interaction Prototype Stack

This section covers the technology approach for generating clickable HTML+JS prototypes during the PLAN stage. This is one of two [핵심] features identified in PROJECT.md.

### Decision: Vanilla HTML + CSS + JS in a Single File

**Use vanilla HTML with inline CSS and JavaScript. No frameworks. No external dependencies. One `.html` file per feature.**

#### Why Vanilla, Not a Framework

| Criterion | Vanilla HTML+JS | Lightweight Framework (Alpine.js, HTMX, Petite-Vue) |
|-----------|-----------------|------------------------------------------------------|
| Zero dependencies | YES -- opens in any browser with `file://` | NO -- needs CDN link or bundled script |
| Claude generation reliability | HIGH -- Claude generates HTML/CSS/JS fluently | MEDIUM -- framework-specific syntax errors are common |
| File size | Small (5-30KB typical) | Larger, and adds framework weight |
| Browser compatibility | Universal -- works in file:// protocol | Most require served content or CDN access |
| Offline capability | Full -- no network needed | Requires cached CDN or bundled framework |
| Maintenance burden | None -- standard web APIs | Framework version churn, API changes |
| Machine parseability | Standard DOM -- easy to scrape/verify | Framework-specific templates harder to parse |

**The decisive factor:** Prototypes must open with `file:///path/to/prototype.html` in any browser, with no server, no npm, no build step. CDN-dependent frameworks fail when offline or behind corporate firewalls. Vanilla HTML is the only zero-assumption choice.

#### Why Single File, Not Multi-File

A single `.html` file with inline `<style>` and `<script>` ensures:
1. **Atomic artifact** -- One file = one feature prototype. Move, share, archive trivially.
2. **No broken references** -- External CSS/JS links break when files move. Inline never breaks.
3. **Claude generation simplicity** -- Claude writes one file, not three coordinated files.
4. **Git-friendly** -- One file diff per prototype. Easy to review changes.
5. **Size is fine** -- Prototypes target 5-30KB. The 14KB inline/external threshold is irrelevant for files opened via `file://`.

### Prototype Architecture: Hash-Based SPA

Each prototype is a single-page application using hash routing to simulate multi-screen navigation.

**Why hash routing:**
- Works with `file://` protocol (History API requires a server)
- URL changes are visible, making screen state shareable ("open the prototype, go to #login-error")
- `hashchange` event is native, simple, and reliable
- No server configuration needed

**Core structure of every generated prototype:**

```html
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title>[Feature Name] - Prototype</title>
  <style>
    /* Wireframe-level styling: functional, not pretty */
    /* Screen containers, form layouts, button states, error displays */
  </style>
</head>
<body>
  <!-- ========================================
       PROTOTYPE MANIFEST (machine-readable)
       ======================================== -->
  <script type="application/json" id="prototype-manifest">
  {
    "feature": "user-auth",
    "version": "1",
    "screens": ["login", "login-error", "login-success", "register", "forgot-password"],
    "interactions": [
      {"screen": "login", "element": "[data-action='submit-login']", "action": "click", "result": "navigate", "target": "#login-success"},
      {"screen": "login", "element": "[data-action='submit-login']", "action": "click", "condition": "invalid", "result": "navigate", "target": "#login-error"},
      {"screen": "login", "element": "[data-action='go-register']", "action": "click", "result": "navigate", "target": "#register"}
    ],
    "fields": [
      {"screen": "login", "element": "[data-field='email']", "type": "email", "required": true, "validation": "email-format"},
      {"screen": "login", "element": "[data-field='password']", "type": "password", "required": true, "validation": "min-length-8"}
    ],
    "error_states": [
      {"screen": "login-error", "trigger": "invalid-credentials", "message": "Invalid email or password"}
    ]
  }
  </script>

  <!-- ========================================
       SCREENS
       ======================================== -->
  <div id="app">
    <div data-screen="login" class="screen">
      <h2>Login</h2>
      <form data-form="login-form">
        <input data-field="email" type="email" placeholder="Email" required>
        <input data-field="password" type="password" placeholder="Password" required>
        <button data-action="submit-login" type="submit">Login</button>
        <a data-action="go-register" href="#register">Create account</a>
        <a data-action="go-forgot" href="#forgot-password">Forgot password?</a>
      </form>
    </div>

    <div data-screen="login-error" class="screen" style="display:none">
      <h2>Login</h2>
      <div data-error="invalid-credentials" class="error">Invalid email or password</div>
      <!-- ... same form ... -->
    </div>

    <div data-screen="login-success" class="screen" style="display:none">
      <h2>Welcome back!</h2>
      <p>Redirecting to dashboard...</p>
    </div>

    <!-- More screens... -->
  </div>

  <!-- ========================================
       INTERACTION ENGINE
       ======================================== -->
  <script>
    // Hash-based router
    function showScreen(screenId) {
      document.querySelectorAll('[data-screen]').forEach(s => s.style.display = 'none');
      const target = document.querySelector(`[data-screen="${screenId}"]`);
      if (target) target.style.display = 'block';
    }

    window.addEventListener('hashchange', () => {
      const screen = location.hash.slice(1) || 'login';
      showScreen(screen);
    });

    // Form validation simulation
    document.querySelectorAll('[data-action]').forEach(el => {
      el.addEventListener('click', (e) => {
        const action = el.dataset.action;
        // Action handlers defined per-prototype
        handleAction(action, e);
      });
    });

    function handleAction(action, event) {
      // Per-feature interaction logic
    }

    // Initialize
    showScreen(location.hash.slice(1) || 'login');
  </script>
</body>
</html>
```

### Data Attribute Convention: The Prototype-to-Spec Bridge

The most critical design decision. Data attributes on prototype HTML elements create a machine-readable contract that connects the prototype to the E2E feature spec and later to verification.

**Convention:**

| Attribute | Purpose | Example |
|-----------|---------|---------|
| `data-screen="name"` | Identifies a screen/view in the flow | `data-screen="login"` |
| `data-action="name"` | Identifies a clickable interaction point | `data-action="submit-login"` |
| `data-field="name"` | Identifies a form input | `data-field="email"` |
| `data-error="name"` | Identifies an error state display | `data-error="invalid-credentials"` |
| `data-nav="target"` | Identifies navigation target (screen name) | `data-nav="dashboard"` |
| `data-state="name"` | Identifies a dynamic state container | `data-state="loading-spinner"` |
| `data-role="name"` | Identifies a semantic UI component | `data-role="sidebar"`, `data-role="header"` |

**Why data attributes, not classes or IDs:**
- Classes change for styling reasons. Data attributes are stable semantic markers.
- IDs must be unique. Data attributes can repeat (multiple `data-action="delete"` on different screens).
- Industry standard for E2E testing (`data-testid` pattern), adopted here for specification.
- Machine-parseable: `document.querySelectorAll('[data-screen]')` extracts all screens.

**Why NOT `data-testid`:** We use domain-specific attribute names (`data-screen`, `data-action`, `data-field`) instead of generic `data-testid` because:
1. Each attribute type carries semantic meaning (screen vs action vs field).
2. The prototype manifest can reference them by type, not just by name.
3. During verification, the checker knows WHAT to expect: "screen X should have action Y" not just "element with testid Z should exist."

### Prototype Manifest: The Machine-Readable Specification

The embedded `<script type="application/json" id="prototype-manifest">` block is the key innovation. It serves three purposes:

1. **For the user:** Describes what they should see and click during review (can be extracted as a clickthrough guide).
2. **For the E2E spec:** Maps directly to the 5-step chain (Screen > Connection > Processing > Response > Error). Each manifest entry IS a spec assertion.
3. **For verification:** During TEST stage, a verifier script can parse the manifest and check that the actual implementation contains matching elements, routes, and behaviors.

**Manifest schema:**

```json
{
  "feature": "string -- feature identifier, matches spec filename",
  "version": "string -- increment on prototype revision",
  "screens": ["string[] -- all screen names in this feature"],
  "interactions": [
    {
      "screen": "string -- which screen this interaction lives on",
      "element": "string -- CSS selector using data attributes",
      "action": "click|submit|input|hover",
      "condition": "string? -- optional precondition (e.g., 'invalid', 'empty')",
      "result": "navigate|show|hide|alert|validate|submit",
      "target": "string? -- target screen or element",
      "description": "string? -- human-readable description of what happens"
    }
  ],
  "fields": [
    {
      "screen": "string -- which screen",
      "element": "string -- CSS selector",
      "type": "text|email|password|number|select|checkbox|radio|textarea",
      "required": "boolean",
      "validation": "string? -- validation rule name"
    }
  ],
  "error_states": [
    {
      "screen": "string -- which screen shows the error",
      "trigger": "string -- what causes this error",
      "message": "string -- error text shown to user"
    }
  ]
}
```

### Prototype File Structure

```
.lifecycle/prototypes/
  user-auth.html           # Login/register/forgot-password flow
  chat-messaging.html      # Send/receive/edit/delete messages
  settings-profile.html    # Profile editing, preferences

.lifecycle/specs/
  user-auth.spec.md        # E2E spec referencing prototype manifest
  chat-messaging.spec.md
  settings-profile.spec.md
```

**One prototype per feature, not per screen.** A feature prototype contains ALL screens for that feature's flow. This keeps the interaction chain complete in one file and prevents cross-file navigation complexity.

### How Prototypes Connect to E2E Specs

The E2E spec (화면 > 연결 > 처리 > 응답 > 에러) references prototype manifest entries:

```markdown
## Feature: User Authentication

### Flow 1: Successful Login

| Step | Spec | Prototype Reference |
|------|------|---------------------|
| Screen | Login form with email + password fields | `data-screen="login"`, fields: `data-field="email"`, `data-field="password"` |
| Connection | POST /api/auth/login with {email, password} | `data-action="submit-login"` triggers form submission |
| Processing | Validate credentials against user store | (server-side, not in prototype) |
| Response | JWT token + redirect to dashboard | `data-screen="login-success"` shown after action |
| Error | "Invalid email or password" message | `data-screen="login-error"`, `data-error="invalid-credentials"` |

### Verification Checklist (from prototype manifest)

- [ ] Screen "login" exists with fields "email" (required, email-format) and "password" (required, min-length-8)
- [ ] Action "submit-login" navigates to "login-success" on valid input
- [ ] Action "submit-login" navigates to "login-error" on invalid input
- [ ] Error "invalid-credentials" shows message "Invalid email or password"
- [ ] Action "go-register" navigates to "register"
- [ ] Action "go-forgot" navigates to "forgot-password"
```

This creates a traceable chain: Prototype manifest entry --> E2E spec line --> Verification checklist item --> Actual implementation test.

### Verification Script Pattern

During TEST stage, a lightweight script can extract the manifest from the prototype and verify the implementation:

```javascript
// Conceptual -- would be part of gsd-tools.cjs or a separate verify script
function extractManifest(htmlPath) {
  const html = fs.readFileSync(htmlPath, 'utf8');
  const match = html.match(/<script type="application\/json" id="prototype-manifest">([\s\S]*?)<\/script>/);
  return JSON.parse(match[1]);
}

function verifyImplementation(manifest, projectRoot) {
  const results = [];
  for (const interaction of manifest.interactions) {
    // Check: does the implementation have a route/handler for this action?
    // Check: does the UI have the expected data attributes?
    // Check: does the error handling exist for listed error_states?
    results.push({ interaction, status: 'pass|fail|missing' });
  }
  return results;
}
```

This is not auto-generated test code (which FEATURES.md explicitly flags as an anti-feature). It is a structural verification that checks "does the implementation have the elements the spec promised?" -- not "does clicking produce the right result."

### CSS Approach: Wireframe-Level, Not Design-Level

Prototypes use minimal, functional CSS:

```css
/* Base: system font, neutral palette, obvious clickable elements */
* { box-sizing: border-box; margin: 0; padding: 0; }
body { font-family: system-ui, sans-serif; padding: 20px; max-width: 480px; margin: 0 auto; background: #f5f5f5; }
.screen { background: white; border: 2px solid #ddd; border-radius: 8px; padding: 24px; }
button, [data-action] { background: #2563eb; color: white; border: none; padding: 10px 20px; border-radius: 4px; cursor: pointer; }
button:hover { background: #1d4ed8; }
input, textarea, select { width: 100%; padding: 10px; border: 1px solid #ccc; border-radius: 4px; margin-bottom: 12px; }
.error { background: #fef2f2; border: 1px solid #fca5a5; color: #dc2626; padding: 12px; border-radius: 4px; margin-bottom: 12px; }
.success { background: #f0fdf4; border: 1px solid #86efac; color: #16a34a; padding: 12px; border-radius: 4px; }
```

**Why wireframe-level:** PROJECT.md and FEATURES.md both flag "Pixel-Perfect UI Mockups" as an anti-feature. The prototype validates FLOW (can I log in? what happens on error?), not DESIGN (what shade of blue is the button?). Wireframe styling keeps the focus on interactions.

## Architecture Patterns (Stack-Level)

### Pattern 1: Progressive Disclosure Architecture

**The most important pattern for skill development.**

Skills should NOT load all instructions at once. Instead:

1. **Frontmatter** (always loaded): name, description -- minimal, for routing decisions only
2. **SKILL.md body** (loaded on invocation): core instructions, navigation to supporting files
3. **Supporting files** (loaded on demand): reference.md, templates/, scripts/, examples/

```
my-skill/
  SKILL.md           # < 500 lines. Overview + navigation
  reference.md       # Detailed API docs, loaded when needed
  templates/         # Output templates
  scripts/           # Executable helpers
  examples/          # Expected output samples
```

**Why:** Claude Code's skill description budget is 2% of context window. Bloated descriptions crowd out other skills. Full content loads only on invocation.

### Pattern 2: Orchestrator-Executor Separation

The orchestrator skill stays lean (~10-15% context). It coordinates, never executes directly.

```
Orchestrator (SKILL.md)
  - Reads STATE.md for current position
  - Determines which stage to run
  - Spawns subagent with stage-specific instructions
  - Collects results, updates state
  - Routes to next stage

Executor (subagent)
  - Gets fresh 200k context
  - Reads plan files
  - Executes tasks
  - Creates artifacts
  - Returns structured result
```

GSD proves this: execute-phase.md orchestrates, execute-plan.md executes. The orchestrator never touches code directly.

### Pattern 3: Markdown-as-Memory

Every artifact produced during development serves as external memory for future sessions:

| Artifact | Memory Function |
|----------|----------------|
| STATE.md | Where am I? What's next? |
| ROADMAP.md | What's the full plan? What's done? |
| PROJECT.md | Why are we building this? What are the constraints? |
| SUMMARY.md (per plan) | What was built? What decisions were made? |
| ADR docs | Why was X chosen over Y? |
| Retrospective docs | What went wrong? What to remember next time? |

**Key insight from PROJECT.md:** "Stage artifacts themselves serve as structured memory -- no separate 'remember this' mechanism needed."

### Pattern 4: State Machine via Filesystem

Stage transitions are tracked through file existence and frontmatter status fields, not in-memory state:

```
.planning/
  STATE.md              # Current position (phase, plan, status)
  ROADMAP.md            # Phase definitions + progress table
  config.json           # Project settings
  phases/
    01-plan/
      01-01-PLAN.md     # Plan exists = planned
      01-01-SUMMARY.md  # Summary exists = executed
      01-VERIFICATION.md # Verification exists = verified
```

**Why filesystem, not database:** Claude Code sessions are ephemeral. Context compaction loses in-memory state. Files persist across sessions, compactions, and crashes.

### Pattern 5: Prototype-as-Contract

**New pattern specific to the User Interaction Prototype feature.**

The prototype HTML file serves as a living specification contract:

```
PLAN stage:
  1. Claude generates feature-name.html with embedded manifest
  2. User opens file://.../feature-name.html in browser
  3. User clicks through, identifies missing screens/interactions
  4. Claude updates prototype until user approves
  5. Manifest is extracted into feature-name.spec.md (E2E spec)

DO stage:
  6. Implementation follows the spec (screens, actions, fields, errors)
  7. Implementation includes matching data attributes on real UI elements

TEST stage:
  8. Verification script extracts manifest from prototype
  9. Checks implementation for matching data attributes and routes
  10. Reports: "spec says data-action='submit-login' exists on screen 'login' -- FOUND/MISSING"
```

**The key constraint:** The prototype is NOT throwaway. It persists as the reference specification. Changes to the prototype during PLAN require re-agreement. The manifest is the source of truth for what "done" means.

## Invocation Control Matrix

Critical for the dev-lifecycle orchestrator -- some skills should auto-trigger, others should be user-only:

| Skill | `disable-model-invocation` | `user-invocable` | Rationale |
|-------|---------------------------|-------------------|-----------|
| dev-lifecycle (main) | false | true | Auto-detects phase transitions AND user can invoke directly |
| Stage executors | true | false | Only the orchestrator should trigger these |
| ADR | false | true | Auto-detects decision moments AND user can invoke |
| gsd-retrospective | true | true | User-triggered only (no false positives) |
| work-log | true | true | User-triggered only |

## Hook Strategy for Dev-Lifecycle

| Hook Event | Matcher | Purpose | Confidence |
|------------|---------|---------|------------|
| `SessionStart` | `startup` | Re-inject STATE.md context, surface pre-flight checks | HIGH |
| `SessionStart` | `compact` | Re-inject critical state after compaction (prevents context loss) | HIGH |
| `PreToolUse` | `Edit\|Write` | Validate against stage gates (no code edits during PLAN stage) | MEDIUM |
| `PostToolUse` | `Edit\|Write` | Track modified files for COMMIT stage artifact | HIGH |
| `Stop` | (none) | Check if current stage deliverables are complete before allowing stop | MEDIUM |
| `SubagentStop` | (any) | Collect executor results, update STATE.md | HIGH |

## Alternatives Considered

| Recommended | Alternative | When to Use Alternative |
|-------------|-------------|-------------------------|
| Vanilla HTML+CSS+JS (single file) | Alpine.js via CDN | Never for prototypes. CDN dependency violates zero-assumption constraint. Alpine's `x-data`/`x-show` syntax adds learning curve for spec parsing without proportional benefit. |
| Vanilla HTML+CSS+JS (single file) | HTMX via CDN | Never for prototypes. HTMX is designed for server interactions (`hx-get`, `hx-post`). Prototypes have no server. Using HTMX for client-only interactions is misuse. |
| Hash-based routing (`#screen-name`) | History API (`pushState`) | Never for file:// prototypes. History API requires a web server. Hash routing works universally including file:// protocol. |
| Inline CSS+JS in single HTML | Separate .css and .js files | Only if a prototype exceeds ~50KB (unlikely). Separation adds file management complexity with no benefit for small prototypes. |
| Embedded JSON manifest | Separate manifest.json file | Never. Keeping the manifest inside the HTML ensures they cannot drift apart. One file = one source of truth. |
| Domain-specific data attributes (`data-screen`, `data-action`) | Generic `data-testid` on everything | Never. Semantic attributes carry meaning ("this is a screen" vs "this is an action"). Generic testids require an external mapping to understand what they represent. |
| SKILL.md (Agent Skills standard) | .claude/commands/ (legacy) | Never for new work. Commands still work but lack: supporting files, invocation control, subagent execution. Skills supersede commands entirely. |
| Node.js CLI (gsd-tools.cjs) | Pure bash scripts | When operations are trivial (< 10 lines). For anything involving JSON parsing, file manipulation, or state management, the CLI is more reliable. Bash is fragile with JSON. |
| Subagents for execution | Inline execution in skill | Never for complex tasks. Inline execution bleeds into orchestrator context. Always fork to subagent for tasks that produce output or modify files. |
| `context: fork` on skills | Manual Task() spawning | When the skill IS the task definition. Use `context: fork` + `agent: [type]` for self-contained skills. Use Task() (now Agent tool) when orchestrator needs to compose the task dynamically. |
| Filesystem state (STATE.md) | In-memory state / environment vars | Never for cross-session state. Env vars die with the session. In-memory state dies with compaction. Files survive everything. |
| Progressive disclosure | Monolithic SKILL.md | Never. A 2000-line SKILL.md wastes context budget. Split into SKILL.md (< 500 lines) + supporting files. |

## What NOT to Use

| Avoid | Why | Use Instead |
|-------|-----|-------------|
| React/Vue/Svelte for prototypes | Framework overhead destroys the zero-dependency constraint. Build step required. Claude generates framework code less reliably than vanilla HTML. | Vanilla HTML+CSS+JS, single file |
| Tailwind CSS via CDN | CDN dependency. Also, utility-class-heavy HTML is harder to machine-parse ("what does this element DO?" gets buried in `class="flex items-center p-4 bg-blue-500"`). | Minimal inline CSS with semantic data attributes |
| Canvas/SVG-based wireframing | Not clickable in the same way as HTML forms and buttons. Cannot validate form interactions, navigation flows, or error states. Hard to machine-parse. | Standard HTML form elements |
| Figma/external design tool export | Requires external tool. Generated HTML is bloated and non-semantic. Cannot embed machine-readable manifest. | Hand-crafted (Claude-generated) HTML with data attributes |
| iframe-based multi-page approach | Adds complexity (cross-frame communication, relative paths). Hash routing achieves the same multi-screen effect in a single file. | Hash-based SPA routing within single file |
| MCP servers for state management | Overkill for file-based state. Adds infrastructure dependency. Skills should be zero-infrastructure. | Filesystem state (STATE.md, config.json) read/written by CLI tool |
| Database (SQLite, etc.) for planning artifacts | Claude Code reads/writes markdown natively. SQL adds complexity with zero benefit for this use case. | Markdown files with YAML frontmatter |
| Auto-generated E2E test code (Playwright/Cypress) | FEATURES.md flags this as an anti-feature. Generated tests are brittle and give false confidence. | Structural verification via manifest parsing (checks element existence, not behavior) |

## Stack Patterns by Variant

**If the feature has form-heavy interactions (login, settings, data entry):**
- Prototype emphasizes: `data-field` attributes, validation messages, error states
- CSS includes: form styling, input states (focus, error, disabled), inline validation
- Manifest emphasizes: `fields` array with types and validation rules

**If the feature has navigation-heavy interactions (dashboard, multi-step wizard, tab panels):**
- Prototype emphasizes: `data-screen` and `data-nav` attributes, breadcrumb/tab state
- CSS includes: active tab indicators, progress bars, back/forward navigation
- Manifest emphasizes: `interactions` array with navigate results

**If the feature has state-heavy interactions (toggle, expand/collapse, drag-drop):**
- Prototype emphasizes: `data-state` and `data-action` attributes, toggle visibility
- JS includes: state toggle functions, CSS class toggling
- Manifest emphasizes: `interactions` array with show/hide results

**If building a simple skill (reference/conventions):**
- Single SKILL.md file, no supporting files
- No subagents needed
- Use `user-invocable: false` if it's background knowledge

**If building an orchestrator skill (this project):**
- SKILL.md + reference files + templates + CLI tool
- Multiple subagents for stage execution
- Hooks for deterministic enforcement
- Filesystem state management
- Progressive disclosure is mandatory

**If building a generator skill (creates artifacts):**
- SKILL.md + templates/ + scripts/
- Use `context: fork` to isolate generation
- Bundle scripts in skill directory, reference via `$CLAUDE_SKILL_DIR`

## Version Compatibility

| Component | Compatible With | Notes |
|-----------|-----------------|-------|
| Agent Skills spec 1.0 | Claude Code, OpenAI Codex, GitHub Copilot, Cursor, VS Code, Gemini CLI | Cross-platform standard. Core features (SKILL.md, frontmatter, supporting files) work everywhere. Claude Code extensions (hooks, subagents, `context: fork`) are Claude-specific. |
| gsd-tools.cjs (Node.js) | Node 18+ | Single-file CJS format. No package.json, no dependencies. Runs on any system with Node installed. |
| Hooks system | Claude Code 1.0+ | Settings-based configuration. 20+ lifecycle events. Prompt-based and agent-based hooks available for judgment calls. |
| Subagents | Claude Code 1.0+ | Up to 10 simultaneous subagents (2026). `.claude/agents/` for project-level, `~/.claude/agents/` for user-level. |
| Prototype HTML files | Any modern browser | Uses standard HTML5, CSS3, ES6+. No polyfills needed. Works with file:// protocol. Tested: Chrome, Firefox, Safari, Edge. |
| Prototype manifest (JSON) | Any JSON parser | Standard JSON embedded in `<script type="application/json">`. Extractable via regex or DOM parsing. |

## Key Constraints from PROJECT.md

These constraints directly shape stack choices:

1. **Skill format only** -- No servers, no infrastructure. Everything runs as SKILL.md + supporting files inside Claude Code.
2. **Orchestration over modification** -- Don't modify GSD/PDCA. Build on top of them. The dev-lifecycle skill calls GSD/PDCA skills, not replaces them.
3. **Universal applicability** -- Must work for web, mobile, API, desktop. No framework-specific code in the orchestrator. Project-specific details go in config.json.
4. **Hardcoding removal** -- Current SKILL.md has muse-specific paths, commands, and flows. All must become configurable or templated.
5. **Prototype as contract** -- Prototype is not throwaway wireframe. It is the specification artifact that drives DO and TEST stages.

## Sources

- [Claude Code Skills Documentation](https://code.claude.com/docs/en/skills) -- SKILL.md format, frontmatter reference, progressive disclosure, invocation control. Verified 2026-03-22. Confidence: HIGH
- [Claude Code Hooks Guide](https://code.claude.com/docs/en/hooks-guide) -- Hook events, matchers, input/output, prompt-based and agent-based hooks. Verified 2026-03-22. Confidence: HIGH
- [Claude Code Subagents Documentation](https://code.claude.com/docs/en/sub-agents) -- Subagent configuration, frontmatter fields, tool restrictions, memory, isolation. Verified 2026-03-22. Confidence: HIGH
- [Agent Skills Open Standard](https://agentskills.io/home) -- Cross-platform specification. Adopted by Anthropic, Microsoft, OpenAI, Atlassian, Figma, Cursor, GitHub. Confidence: HIGH
- [Anthropic Skills Repository](https://github.com/anthropics/skills) -- Official skill specification and SDK. Confidence: HIGH
- GSD codebase (`~/.claude/get-shit-done/`) -- Proven orchestration patterns: CLI tool, workflows, state management, subagent spawning. Directly examined. Confidence: HIGH
- Existing skills (`~/.claude/skills/`) -- dev-lifecycle, adr, gsd-retrospective, work-log, canvas-design. Directly examined. Confidence: HIGH
- [FreeCodeCamp: Interactive HTML Prototypes](https://www.freecodecamp.org/news/how-to-create-interactive-html-prototypes/) -- Vanilla HTML prototype patterns. Confidence: MEDIUM
- [DEV.to: SPA Routing Using Hash](https://dev.to/thedevdrawer/single-page-application-routing-using-hash-or-url-9jh) -- Hash routing for file:// protocol compatibility. Confidence: MEDIUM
- [DEV.to: Data Attributes for E2E Tests](https://dev.to/marcostreng/how-to-add-data-attributes-to-your-component-library-and-benefit-from-them-in-your-e2e-tests-148l) -- data-testid patterns adapted for prototype specification. Confidence: MEDIUM
- [Testing Library: ByTestId](https://testing-library.com/docs/queries/bytestid/) -- Data attribute conventions for element selection in testing. Confidence: HIGH

---
*Stack research for: Claude Code skill-based development lifecycle orchestrator (with User Interaction Prototype stack)*
*Researched: 2026-03-22*
