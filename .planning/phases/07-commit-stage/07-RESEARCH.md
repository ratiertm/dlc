# Phase 7: COMMIT Stage - Research

**Researched:** 2026-03-22
**Domain:** Stage 4 COMMIT adapter -- verification gate + why-centric git commit + ADR references
**Confidence:** HIGH

## Summary

Phase 7 implements the COMMIT Stage (Stage 4) adapter for the dev-lifecycle pipeline. This is the final stage of the inner loop (PLAN -> DO -> TEST -> COMMIT). The COMMIT stage reads TEST outputs (verification.json), blocks if any spec step FAIL exists, and when all pass, creates a why-centric git commit with ADR references. It also performs a final artifact existence check per manifest.json gate rules.

The implementation follows the exact same adapter pattern established by plan-stage.md, do-stage.md, and test-stage.md: Step 0 initialization -> Step 1-N pipeline -> anti-patterns -> relationship to other skills -> file locations. All three prior adapters are complete and provide a proven template. The COMMIT stage is the simplest of the four inner loop stages -- it reads existing artifacts, validates them, and orchestrates a git commit. No new artifact formats need to be created; it consumes TEST outputs and produces a commit hash.

**Primary recommendation:** Follow the established Stage adapter pattern exactly. The commit-stage.md reference file should mirror test-stage.md structure. Focus on three core responsibilities: (1) verification gate enforcement, (2) why-centric commit message generation with ADR references, and (3) artifact registration in manifest.json.

<user_constraints>

## User Constraints (from CONTEXT.md)

### Locked Decisions
- TEST Stage의 verification 결과 확인 -- FAIL 항목이 있으면 COMMIT 차단
- 차단 시 구체적 gap list 표시 (어떤 spec step이 FAIL인지, 왜인지)
- 사용자가 방향 결정 (DO로 돌아가기 / 무시하고 진행 등)
- "why" 중심 커밋 메시지 -- 변경의 이유를 중심으로 작성
- 관련 ADR이 있으면 커밋 메시지에 `(ADR-NNN)` 참조 포함
- 관련 spec ID도 참조 가능
- manifest.json의 gate rules에 따라 필수 artifact 존재 확인
- spec.md, prototype.html, verification report가 모두 있어야 COMMIT 진입

### Claude's Discretion
- commit-stage.md 참조 파일의 구체적 구조
- 커밋 메시지 포맷의 세부사항
- gap list 표시 형식
- SKILL.md/role-matrix Stage 4 업데이트 방식

### Deferred Ideas (OUT OF SCOPE)
None -- discussion stayed within phase scope

</user_constraints>

<phase_requirements>

## Phase Requirements

| ID | Description | Research Support |
|----|-------------|-----------------|
| PIPE-04 | Stage 4 COMMIT -- why 중심 커밋 + ADR 참조 + artifact 존재 검증 | Full adapter pattern from Stages 1-3; verification.json schema from test-stage.md; gate_rules 3_to_4 from manifest.json |
| ARCH-04 | Stage 전환 게이트 -- 필수 artifact 없이 다음 Stage 진입 차단 | Gate condition `artifacts.3_test.outputs.length > 0` already defined; COMMIT must enforce this AND do deeper content-level checks (verification.json overall status) |

</phase_requirements>

## Standard Stack

### Core
| Library | Version | Purpose | Why Standard |
|---------|---------|---------|--------------|
| Git | system | Commit creation | Direct tool per role-matrix Stage 4 |
| verification.json | n/a | TEST stage output, machine-parseable results | Input consumed by COMMIT gate logic |
| manifest.json | n/a | Artifact registry with gate rules | Gate condition `3_to_4` already defined |
| state.json | n/a | Pipeline state tracking | Stage/status updates |

### Supporting
| Library | Version | Purpose | When to Use |
|---------|---------|---------|-------------|
| ADR skill | existing | Referenced in commit messages | When `docs/decisions/` contains ADRs created during DO stage |

### Alternatives Considered
| Instead of | Could Use | Tradeoff |
|------------|-----------|----------|
| Direct git commands | git hooks | Hooks add complexity; Stage 4 is Claude-orchestrated, not automated |
| Content-level verification check | Simple existence check only | Existence check is already in gate_rules; content check (PASS/FAIL) adds real value |

## Architecture Patterns

### Recommended File Structure
```
skill/
  references/
    commit-stage.md          # NEW: Stage 4 adapter (this phase creates it)
    plan-stage.md            # Existing: Stage 1 pattern reference
    do-stage.md              # Existing: Stage 2 pattern reference
    test-stage.md            # Existing: Stage 3 pattern reference
    stage-transitions.md     # Existing: Gate conditions
    role-matrix.md           # Existing: needs Stage 4 update
  templates/
    manifest.json            # Existing: gate_rules 3_to_4
    state.json               # Existing: state schema
```

### Pattern 1: Stage Adapter Structure (Canonical)
**What:** Every stage adapter follows the same document structure
**When to use:** Always -- all four inner loop stages use this pattern
**Example structure (derived from plan-stage.md, do-stage.md, test-stage.md):**

```markdown
# {STAGE} Stage Adapter

## When This Runs
## Prerequisites
## Step 0: Stage Initialization
## Step 1-N: Pipeline Steps
## Anti-Patterns
## Relationship to Other Skills
## File Locations
```

### Pattern 2: Verification Gate (Content-Level)
**What:** Beyond simple `outputs.length > 0` gate, COMMIT reads verification.json and checks `specVerification.overall` and `userVerification.status`
**When to use:** At Step 0 of COMMIT stage, after the basic gate passes
**Logic:**

```
1. Basic gate: artifacts.3_test.outputs.length > 0 (from manifest.json)
2. Content gate: Read verification.json
   - specVerification.overall === "passed"
   - userVerification.status === "passed"
   - If either is "failed" -> BLOCK with gap list
   - If userVerification.status === "pending" -> BLOCK, user behavioral not done
```

### Pattern 3: Why-Centric Commit Message Format
**What:** Commit message structure focusing on WHY, not WHAT
**When to use:** Step 2 of COMMIT stage, generating the commit message
**Recommended format:**

```
{why-summary}: {feature-name}

Implements {feature description} per E2E spec.

Spec: .lifecycle/features/{name}/spec.md
Steps: {N} verified ({M} deviations)
{ADR references if any}

(ADR-001) (ADR-002)
e2e-{feature}-001 through e2e-{feature}-{NNN}
```

### Pattern 4: Gap List Display (Blocked State)
**What:** When verification fails, display exactly which steps failed and why
**When to use:** When verification.json shows failures
**Format:**

```
COMMIT Stage -- {feature-name} -- BLOCKED
==========================================
Verification gate FAILED. Cannot commit.

Failed spec steps:
  [FAIL] e2e-{feature}-{NNN}: {title} -- {reason}
  [FAIL] e2e-{feature}-{NNN}: {title} -- {reason}

User behavioral verification: {PASSED/FAILED/PENDING}

Options:
  1. Return to DO stage to fix failures
  2. Return to TEST stage to re-verify
  3. Override and commit anyway (not recommended)
```

### Anti-Patterns to Avoid
- **Auto-committing without user confirmation:** COMMIT must present the commit message and get user approval before executing git commit
- **Skipping verification.json content check:** The basic `outputs.length > 0` gate is insufficient -- MUST read verification.json and check overall status
- **Including sensitive files:** Must exclude `.env`, credentials, `.lifecycle/` directory from the commit
- **Committing when userVerification is "pending":** Dual gate requires BOTH Claude structural AND user behavioral verification to pass

## Don't Hand-Roll

| Problem | Don't Build | Use Instead | Why |
|---------|-------------|-------------|-----|
| Commit message formatting | Custom parser | Simple template interpolation | Commit messages are short text; no parser needed |
| Verification result parsing | Custom verifier | Read verification.json directly | TEST stage already produces structured JSON |
| ADR reference detection | Scan all code | Read manifest.json `2_do` outputs for type "adr" | Manifest already tracks ADR artifacts |
| Git operations | Shell script wrapper | Direct git commands via Claude | Role-matrix says "Git (direct)" for Stage 4 |

**Key insight:** COMMIT is the simplest inner loop stage. TEST already did the hard verification work. COMMIT just reads those results, gates accordingly, and creates a commit. Do not over-engineer.

## Common Pitfalls

### Pitfall 1: Ignoring userVerification Status
**What goes wrong:** Committing when Claude structural verification passed but user behavioral verification is still "pending"
**Why it happens:** The basic gate `outputs.length > 0` passes as soon as TEST writes verification.json, even if user hasn't completed behavioral checks
**How to avoid:** Always read `verification.json.userVerification.status` -- must be "passed", not "pending" or "failed"
**Warning signs:** verification.json exists but `userVerification.status !== "passed"`

### Pitfall 2: Forgetting to Register COMMIT Outputs in Manifest
**What goes wrong:** Stage 4 outputs (commit-hash, optional tag) not registered, breaking gate 4_to_5 for DEPLOY
**Why it happens:** COMMIT feels like a "final" step; easy to forget it produces artifacts too
**How to avoid:** Always register commit hash (and optional tag) in `artifacts.4_commit.outputs`
**Warning signs:** `artifacts.4_commit.outputs` is empty after commit

### Pitfall 3: What vs Why in Commit Messages
**What goes wrong:** Commit message lists files changed instead of explaining why the changes matter
**Why it happens:** Default git behavior and developer habit
**How to avoid:** Template the message: start with the business/user value, reference spec and ADRs, let git diff handle the "what"
**Warning signs:** Commit message contains file paths or code details instead of purpose

### Pitfall 4: Not Handling the Override Path
**What goes wrong:** User wants to commit despite failures (e.g., known issue, will fix later), but there's no path forward
**Why it happens:** CONTEXT.md says "사용자가 방향 결정" -- user decides direction, including override
**How to avoid:** Provide explicit override option with warning. If overridden, record it in commit message and manifest
**Warning signs:** User says "just commit it" but adapter has no override flow

### Pitfall 5: Artifact Existence vs Content Validity
**What goes wrong:** Gate passes because files exist, but content is invalid (e.g., empty verification.json, corrupted spec.md)
**Why it happens:** gate_rules only check `outputs.length > 0`
**How to avoid:** COMMIT Step 0 should do content-level checks: verify spec.md has `status: verified`, verification.json parses correctly, prototype.html exists
**Warning signs:** Gate passes but downstream processing fails on malformed data

## Code Examples

### Example 1: Step 0 -- Reading and Validating verification.json
```markdown
# In commit-stage.md Step 0:

1. Read `.lifecycle/manifest.json`
2. Check gate: `artifacts.3_test.outputs.length > 0`
3. Read `.lifecycle/features/{feature-name}/verification.json`
4. Check:
   - specVerification.overall === "passed"
   - userVerification.status === "passed"
5. If any check fails -> display gap list, set status "blocked"
```

### Example 2: Commit Message Generation
```
feat(user-login): enable secure authentication flow

Users can now log in with email/password credentials.
Validated against 5 E2E spec steps, all verified.
1 approved deviation (DEV-001: OAuth deferred to v2).

(ADR-003) Chose bcrypt over argon2 for password hashing
Spec: .lifecycle/features/user-login/spec.md
```

### Example 3: Manifest Output Registration
```json
{
  "type": "commit-hash",
  "path": "{git-commit-sha}",
  "created_at": "2026-03-22T10:00:00Z",
  "status": "complete"
}
```

### Example 4: Git File Staging Guidance
```markdown
# Files to stage:
- All source code changes (from DO stage)
- .lifecycle/features/{name}/spec.md (status: verified)
- .lifecycle/features/{name}/verification.json
- .lifecycle/features/{name}/verification.md
- docs/decisions/*.md (any ADRs created)

# Files to EXCLUDE:
- .env, credentials, secrets
- node_modules/, .venv/, etc.
- .lifecycle/state.json (runtime state, not committed)
- .lifecycle/manifest.json (runtime state, not committed)
```

## State of the Art

| Old Approach | Current Approach | When Changed | Impact |
|--------------|------------------|--------------|--------|
| Simple `outputs.length > 0` gate | Content-level verification check | Phase 7 (now) | COMMIT actually validates TEST results, not just file existence |
| Generic commit messages | Why-centric with spec+ADR refs | Phase 7 (now) | Commit history becomes searchable decision trail |
| No dual gate enforcement | Both structural + behavioral required | Phase 6 (test-stage) | COMMIT inherits and enforces this requirement |

## Adapter Step Design (Recommended)

Based on the established pattern from Stages 1-3, the COMMIT adapter should have these steps:

### Step 0: Stage Initialization
- Read state.json, verify stage = 4 (or transition from 3)
- Check gate condition 3_to_4 from manifest.json
- Read verification.json -- content-level gate check
- Verify all required artifacts exist on disk (spec.md, prototype.html, verification.json, verification.md)
- Update state.json: stage=4, stage_name="COMMIT", status="in_progress"

### Step 1: Verification Gate
- Parse verification.json
- Check specVerification.overall
- Check userVerification.status
- If any FAIL: display gap list, present options (return to DO/TEST, override)
- If all PASS: proceed to Step 2

### Step 2: Commit Preparation
- Collect changed files (from manifest.json 2_do outputs)
- Detect ADR references (from manifest.json 2_do outputs where type="adr")
- Read spec.md for feature summary and step count
- Read deviations count
- Generate why-centric commit message
- Present commit message to user for approval/edit

### Step 3: Git Commit Execution
- Stage files (excluding sensitive files and runtime state)
- Execute git commit with approved message
- Capture commit hash
- Optionally create version tag if user requests

### Step 4: Completion
- Register commit-hash (and optional tag) in manifest.json under `artifacts.4_commit.outputs`
- Set `artifacts.4_commit.completed_at`
- Update state.json: status="completed", resume_hint="COMMIT complete. Ready for DEPLOY stage."
- Announce completion

## SKILL.md and Role-Matrix Updates

### SKILL.md Stage 4 Section (Current)
The existing Stage 4 section in SKILL.md is minimal (7 lines). It needs:
- `Read: $CLAUDE_SKILL_DIR/references/commit-stage.md` directive added
- Pipeline steps listed (matching Stages 1-3 format)
- Primary/Supporting skill roles clarified

### Role-Matrix Stage 4 Row (Current)
Already correct: `Git (direct)` as primary, `ADR (reference in commit msg)` as supporting. No change needed.

### Stage Output Types (Current)
Already correct: `commit-hash, tag (optional)`. No change needed.

## Validation Architecture

### Test Framework
| Property | Value |
|----------|-------|
| Framework | Manual verification (Claude skill, no automated test framework) |
| Config file | none |
| Quick run command | Manual: create a test feature with PLAN->DO->TEST->COMMIT flow |
| Full suite command | Manual: full inner loop integration test |

### Phase Requirements -> Test Map
| Req ID | Behavior | Test Type | Automated Command | File Exists? |
|--------|----------|-----------|-------------------|-------------|
| PIPE-04 | COMMIT stage creates why-centric commit with ADR refs | integration/manual | Run full inner loop on test feature | N/A -- skill file, not code |
| ARCH-04 | Stage gate blocks without required artifacts | integration/manual | Attempt COMMIT with missing/failed verification | N/A -- skill file, not code |

### Sampling Rate
- **Per task commit:** Verify commit-stage.md follows adapter pattern structure
- **Per wave merge:** Verify SKILL.md + role-matrix consistency
- **Phase gate:** Manual inner loop test (PLAN->DO->TEST->COMMIT on a test feature)

### Wave 0 Gaps
None -- this phase produces documentation/skill files, not code requiring automated tests. Validation is structural (does the adapter follow the pattern?) and integration (does the full flow work?).

## Open Questions

1. **Should .lifecycle/ files be committed?**
   - What we know: state.json and manifest.json are runtime state. spec.md and verification reports are artifacts.
   - What's unclear: Which .lifecycle/ files should be in the commit vs excluded
   - Recommendation: Commit feature artifacts (spec.md, prototype.html, verification.*), exclude runtime state (state.json, manifest.json, history/)

2. **Override flow details**
   - What we know: User can override failed verification per CONTEXT.md
   - What's unclear: Should override be recorded as a deviation? How does it affect downstream stages?
   - Recommendation: Record override in commit message with warning prefix. Add `"override": true` flag to manifest output.

3. **Version tag format and trigger**
   - What we know: Role-matrix lists `tag (optional)` as Stage 4 output
   - What's unclear: When to suggest tagging, what format
   - Recommendation: Suggest tag only when user explicitly requests. Format: `v{major}.{minor}.{patch}` per SKILL.md rules.

## Sources

### Primary (HIGH confidence)
- `skill/references/test-stage.md` -- verification.json schema, dual gate pattern, TEST output format
- `skill/references/plan-stage.md` -- adapter structure pattern (Step 0-N + anti-patterns + file locations)
- `skill/references/do-stage.md` -- adapter structure pattern, ADR detection output
- `skill/references/stage-transitions.md` -- gate condition 3_to_4, transition procedure
- `skill/templates/manifest.json` -- gate_rules definition, artifact registry schema
- `skill/SKILL.md` -- Stage 4 section, 9-stage pipeline overview
- `skill/references/role-matrix.md` -- Stage 4 primary/supporting skills, output types

### Secondary (MEDIUM confidence)
- `.planning/phases/07-commit-stage/07-CONTEXT.md` -- user decisions and discretion areas

### Tertiary (LOW confidence)
None -- all findings derived from existing project files

## Metadata

**Confidence breakdown:**
- Standard stack: HIGH -- all tools (Git, verification.json, manifest.json) are already defined in the project
- Architecture: HIGH -- three prior adapter implementations provide exact pattern to follow
- Pitfalls: HIGH -- derived from analysis of existing adapters and their gate logic

**Research date:** 2026-03-22
**Valid until:** 2026-04-22 (stable -- internal project patterns, not external dependencies)
