---
feature: {kebab-case-name}
title: {human-readable title}
created: {ISO-8601 date}
updated: {ISO-8601 date}
status: draft
depends_on: []
steps: {number}
tags: []
---

<!-- Status values:
  pending        -- Not yet implemented
  implemented    -- Code written, not yet verified
  verified (attempt N) -- Mini-verify passed during DO (N = attempt number)
  implemented (override: {reason}) -- Mini-verify skipped with user justification
  failed (N attempts) -- Mini-verify failed after N retry attempts
  verified       -- TEST stage verification passed
  failed         -- TEST stage verification failed

  Step-level status lifecycle:
  pending --(DO implements)--> implemented --(mini-verify pass)--> verified (attempt N)
                                  |                                      |
                                  +--(mini-verify fail x3)--> failed (3 attempts)
                                  |                                      |
                                  +--(user override)--> implemented (override: {reason})
                                                                         |
                                             All three continue to TEST stage:
                                             --(TEST verifies)--> verified
                                             --(TEST fails)--> failed
-->

# E2E Spec: {Title}

{One-line round-trip summary: User does X -> stored in Y -> user sees Z}

## e2e-{feature}-{NNN}: {Screen Step Title}

**Chain:** Screen
**Status:** pending

### What
{What the user sees and does to trigger the interaction}

### Verification Criteria
- [ ] {Concrete, checkable assertion about UI element existence}
- [ ] {Concrete, checkable assertion about initial state}

### Details
- **Element:** {what the user sees -- form, button, page, etc.}
- **User Action:** {what the user does -- click, navigate, type, etc.}
- **Initial State:** {what the screen looks like before the action}

## e2e-{feature}-{NNN}: {Connection Step Title}

**Chain:** Connection
**Status:** pending

### What
{How the frontend communicates with the backend after user action}

### Verification Criteria
- [ ] {Concrete, checkable assertion about request method/endpoint}
- [ ] {Concrete, checkable assertion about request payload}

### Details
- **Method:** {HTTP method or function call}
- **Endpoint:** {URL path or function signature}
- **Request:** {body/parameters shape}
- **Auth:** {authentication requirement -- None, Bearer token, API key, etc.}

## e2e-{feature}-{NNN}: {Processing Step Title}

**Chain:** Processing
**Status:** pending

### What
{What the backend does -- validation, business logic, data transformation}

### Verification Criteria
- [ ] {Concrete, checkable assertion about processing logic}
- [ ] {Concrete, checkable assertion about storage operation}

### Details
- **Steps:**
  1. {processing step 1}
  2. {processing step 2}
- **Storage:** {destination} -- {operation}

> Storage is MANDATORY. Format: `{DB table / LLM API / file path} -- {READ | WRITE | READ+WRITE}`
> Examples: "PostgreSQL users table -- READ", "OpenAI gpt-4 API -- POST", "~/.config/app/settings.json -- WRITE"

## e2e-{feature}-{NNN}: {Response Step Title}

**Chain:** Response
**Status:** pending

### What
{What comes back from the server and how the UI updates}

### Verification Criteria
- [ ] {Concrete, checkable assertion about response status/shape}
- [ ] {Concrete, checkable assertion about UI update}

### Details
- **Success Status:** {HTTP status code or return value}
- **Response Shape:** {what comes back -- JSON structure, redirect, etc.}
- **UI Updates:**
  - {what changes on screen 1}
  - {what changes on screen 2}

## e2e-{feature}-{NNN}: {Error Step Title}

**Chain:** Error
**Status:** pending

### What
{All failure modes for this interaction, with specific user-visible behavior}

### Verification Criteria
- [ ] {Concrete, checkable assertion about client validation error}
- [ ] {Concrete, checkable assertion about server error handling}
- [ ] {Concrete, checkable assertion about network error handling}

### Details
| Condition | Behavior | Status |
|-----------|----------|--------|
| {client validation error} | {what user sees} | null (client-side) |
| {server/processing error} | {what user sees} | {HTTP status} |
| {infrastructure/network error} | {what user sees} | null (network) |

> Minimum 3 error conditions required: (1) client validation, (2) server/processing, (3) infrastructure/network.

## Deviations

_No deviations recorded yet._

<!-- Deviation template:
### DEV-NNN: {Brief description}
- **Spec step:** e2e-{feature}-{NNN}
- **Original:** {what spec said}
- **Actual:** {what was implemented}
- **Reason:** {why}
- **Approved:** {yes/no}
-->
