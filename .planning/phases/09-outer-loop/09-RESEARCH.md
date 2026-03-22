# Phase 9: Outer Loop (Stages 5-9) - Research

**Researched:** 2026-03-22
**Domain:** Dev-lifecycle Outer Loop stage adapters (DEPLOY, DEPLOY TEST, DOCUMENT, RETROSPECT, PROMOTE)
**Confidence:** HIGH

## Summary

Phase 9 implements 5 Outer Loop stage adapters following the exact same pattern established by the Inner Loop adapters (plan-stage.md, do-stage.md, test-stage.md, commit-stage.md). Each adapter follows the Step 0-N + Anti-Patterns + File Locations structure. The adapters are simpler than Inner Loop ones because they do not involve the E2E spec/prototype system -- instead they orchestrate external tools and skills.

The key structural insight: Inner Loop adapters average 240-310 lines and follow a rigid pattern. Outer Loop adapters should be shorter (150-250 lines each) since they primarily delegate to external skills (gsd-retrospective, work-log, ADR) or provide project-type-specific checklists rather than implementing complex verification logic. The manifest.json template already has all 9 stage artifact slots defined with gate rules -- no template changes needed.

**Primary recommendation:** Create 5 reference files (deploy-stage.md, deploy-test-stage.md, document-stage.md, retrospect-stage.md, promote-stage.md) using the Inner Loop adapter pattern, then update SKILL.md Stage 5-9 sections with Read directives and role-matrix.md with updated skill mappings.

<user_constraints>
## User Constraints (from CONTEXT.md)

### Locked Decisions
- Inner Loop adapter pattern (Step 0-N + anti-patterns + file locations) applied identically
- 5 Stage adapters as separate reference files (deploy-stage.md, deploy-test-stage.md, document-stage.md, retrospect-stage.md, promote-stage.md)
- All project types supported equally (Phase 1 decision)
- Stage 5 DEPLOY: project-type-specific deploy templates, auto-detect + user confirm
- Stage 6 DEPLOY TEST: smoke tests (server status, core flow, resource check)
- Stage 7 DOCUMENT: architecture/sequence diagrams (Mermaid or canvas-design), CLAUDE.md/README.md update
- Stage 8 RETROSPECT: gsd-retrospective skill invocation, ADR gap check, work-log, Living State update
- Stage 9 PROMOTE: optional demo generation (Maestro + ADB + FFmpeg), skip without penalty

### Claude's Discretion
- Step structure details for each stage adapter
- Deploy template project-type-specific content
- SKILL.md Stage 5-9 section update approach
- role-matrix Stage 5-9 updates

### Deferred Ideas (OUT OF SCOPE)
None -- discussion stayed within phase scope
</user_constraints>

<phase_requirements>
## Phase Requirements

| ID | Description | Research Support |
|----|-------------|-----------------|
| PIPE-05 | Stage 5 DEPLOY -- project-type-specific deploy templates | Inner Loop adapter pattern + project-detection.md provides type detection; deploy-stage.md needs per-type checklists |
| PIPE-06 | Stage 6 DEPLOY TEST -- smoke tests (server, flow, resources) | deploy-test-stage.md with 3-category smoke test framework; gate 5_to_6 already defined in manifest |
| PIPE-07 | Stage 7 DOCUMENT -- architecture/sequence diagrams, CLAUDE.md/README.md | document-stage.md delegates to Mermaid inline; canvas-design optional; CLAUDE.md update pattern from Phase 8 |
| PIPE-08 | Stage 8 RETROSPECT -- retro + ADR gap + work-log + Living State | retrospect-stage.md orchestrates gsd-retrospective, ADR, work-log skills; Living State update per stage-transitions.md |
| PIPE-09 | Stage 9 PROMOTE -- optional demo generation | promote-stage.md with skip-first design; Maestro/ADB/FFmpeg for mobile, screen recording for web |
</phase_requirements>

## Standard Stack

This phase creates Markdown reference files (not code libraries). The "stack" is the existing skill infrastructure.

### Core
| Component | Location | Purpose | Why Standard |
|-----------|----------|---------|--------------|
| Inner Loop adapter pattern | `skill/references/{plan,do,test,commit}-stage.md` | Template structure for new adapters | Proven pattern across 4 stages, consistent Claude behavior |
| project-detection.md | `skill/references/project-detection.md` | Project type detection for DEPLOY | Already defines 9+ types with sub-types |
| manifest.json template | `skill/templates/manifest.json` | Stage 5-9 artifact slots | Already has `5_deploy` through `9_promote` with gate rules |
| stage-transitions.md | `skill/references/stage-transitions.md` | Gate conditions 4_to_5 through 8_to_9 | Already defines all transition gates |

### External Skills (invoked, not modified)
| Skill | Location | Used By | Interface |
|-------|----------|---------|-----------|
| gsd-retrospective | `~/.claude/skills/gsd-retrospective/SKILL.md` | Stage 8 RETROSPECT | Generates retrospective doc to `docs/retrospectives/` |
| work-log | `~/.claude/skills/work-log/SKILL.md` | Stage 8 RETROSPECT | Records to Obsidian vault `Projects/{name}/` |
| ADR | `~/.claude/skills/adr/SKILL.md` | Stage 8 RETROSPECT (gap check) | Manages `docs/decisions/` directory |

## Architecture Patterns

### Adapter File Structure (canonical pattern)

Every stage adapter follows this exact structure, derived from the Inner Loop adapters:

```markdown
# {STAGE_NAME} Stage Adapter

{1-line description}

## When This Runs
- User trigger keywords
- state.json stage number check
- SKILL.md Read directive reference

## Prerequisites
- .lifecycle/ initialized
- state.json and manifest.json exist
- Previous stage gate must pass

## Step 0: Stage Initialization
- Read state.json, verify stage number
- Check gate condition (N-1_to_N)
- Update state.json (stage, stage_name, status, timestamp)

## Step 1-N: {Stage-specific steps}
{Varies per stage}

## Artifact Registration
- Register outputs in manifest.json
- Set completed_at timestamp
- Update state.json status
- Write history entry
- Regenerate LIVING-STATE.md
- Announce completion

## Anti-Patterns
| # | Anti-Pattern | Why It Is Wrong |

## Relationship to Other Skills

## File Locations
| File | Path | Purpose |
```

### Stage 5 DEPLOY: Recommended Step Structure

```
Step 0: Stage Initialization (gate 4_to_5, state update)
Step 1: Project Type Resolution (read state.json detected_type, confirm with user if first deploy)
Step 2: Deploy Checklist Generation (project-type-specific steps)
Step 3: Deploy Execution (user-guided, track each step)
Step 4: Completion (register deploy-log + deploy-artifact in manifest)
```

**Project-type deploy checklists** (embedded in deploy-stage.md as conditional sections):

| Project Type | Key Deploy Steps |
|-------------|-----------------|
| `node-web/next` | `npm run build`, Vercel/Netlify/Docker deploy, env vars check |
| `node-web/react` | `npm run build`, static hosting (S3/Netlify/Vercel), env vars check |
| `node-web/express` | Build, Docker/PM2/systemd, env vars, port config, health endpoint |
| `flutter` | `flutter build apk/ios`, app store / TestFlight / Firebase App Distribution |
| `python/fastapi` | Requirements install, Uvicorn/Gunicorn, Docker, env vars |
| `python/django` | Collectstatic, Gunicorn, nginx, DB migration, env vars |
| `rust` | `cargo build --release`, binary distribution |
| `go` | `go build`, binary distribution, Docker |
| `ios` | Xcode archive, TestFlight / App Store Connect |
| `android-jvm` | Gradle assembleRelease, Play Store / Firebase |
| Generic | Manual steps, user describes their deploy process |

### Stage 6 DEPLOY TEST: Recommended Step Structure

```
Step 0: Stage Initialization (gate 5_to_6, state update)
Step 1: Smoke Test Plan (3 categories: health, core flow, resources)
Step 2: Health Check (endpoint responds, correct status codes)
Step 3: Core Flow Verification (key user journey works end-to-end)
Step 4: Resource Check (memory, disk, connections within bounds)
Step 5: Completion (register smoke-test-report in manifest)
```

### Stage 7 DOCUMENT: Recommended Step Structure

```
Step 0: Stage Initialization (gate 6_to_7, state update)
Step 1: Architecture Diagram (Mermaid flowchart of system components)
Step 2: Sequence Diagrams (Mermaid sequence for key user flows, derived from spec)
Step 3: README/CLAUDE.md Update (add deploy info, ADR list, architecture diagram link)
Step 4: Completion (register architecture-doc + sequence-doc in manifest)
```

### Stage 8 RETROSPECT: Recommended Step Structure

```
Step 0: Stage Initialization (gate 7_to_8, state update)
Step 1: Retrospective Generation (invoke gsd-retrospective skill)
Step 2: ADR Gap Check (scan for unrecorded decisions, suggest ADR creation)
Step 3: Work Log (invoke work-log skill)
Step 4: Living State Update (regenerate LIVING-STATE.md with lessons)
Step 5: Completion (register retrospective + optional ADR in manifest)
```

### Stage 9 PROMOTE: Recommended Step Structure

```
Step 0: Stage Initialization (gate 8_to_9, state update)
Step 1: Skip Check (ask user: "Would you like to create a demo? Skip is fine.")
  - If skip: register skip artifact, set status to "skipped", announce, STOP
Step 2: Demo Type Selection (screen recording, mobile recording, screenshots)
Step 3: Demo Creation Guidance (provide tool-specific instructions)
Step 4: Completion (register demo/announcement in manifest)
```

### SKILL.md Update Pattern

Current SKILL.md Stage 5-9 sections are brief placeholders (3-5 lines each). Update to match Inner Loop pattern:

```markdown
### Stage 5: DEPLOY
Purpose: Ship to target environment using project-type-specific template.
Primary: dev-lifecycle (deploy orchestration)
Supporting: Project-specific tools
Outputs: Deploy log, deploy artifact

Read: `$CLAUDE_SKILL_DIR/references/deploy-stage.md`

Pipeline:
1. Stage init (gate check, project type resolution)
2. Deploy checklist (type-specific steps)
3. Deploy execution (user-guided)
4. Completion (artifact registration, state update)
```

### role-matrix.md Update Pattern

Current role-matrix Stage 5-9 entries need refinement:

| Stage | Current Primary | Updated Primary | Change Needed |
|-------|----------------|----------------|---------------|
| 5 DEPLOY | "Project-specific template" | dev-lifecycle (deploy orchestration) | Primary is dev-lifecycle, it provides type-specific checklist |
| 6 DEPLOY TEST | "Project-specific template" | dev-lifecycle (smoke test orchestration) | Primary is dev-lifecycle, it defines 3-category smoke framework |
| 7 DOCUMENT | "canvas-design (optional)" | dev-lifecycle (documentation orchestration) | Primary is dev-lifecycle, Mermaid inline; canvas-design remains optional/supporting |
| 8 RETROSPECT | "gsd-retrospective" | dev-lifecycle (retrospective orchestration) | dev-lifecycle is primary orchestrator; gsd-retrospective is the delegated skill |
| 9 PROMOTE | "Manual / optional" | dev-lifecycle (optional promotion) | dev-lifecycle provides the skip-or-proceed flow |

**Rationale:** Inner Loop established that dev-lifecycle is always primary for orchestration. External skills are supporting/delegated, not primary. This is consistent with the adapter pattern where dev-lifecycle "owns" the stage workflow.

## Don't Hand-Roll

| Problem | Don't Build | Use Instead | Why |
|---------|-------------|-------------|-----|
| Retrospective generation | Custom retro format | gsd-retrospective skill | Already has structured format, git analysis, phase detection |
| Work logging | Custom work log | work-log skill | Already integrates with Obsidian vault, handles git diff collection |
| ADR creation | Custom ADR format | ADR skill | Already has numbered format, decision extraction, `docs/decisions/` management |
| Project type detection | Re-implement detection | Existing project-detection.md | Already defines 9+ types with sub-types, detection flow, multi-type handling |
| Gate conditions | New gate logic | Existing manifest.json gate_rules | Gates 4_to_5 through 8_to_9 already defined |
| Mermaid diagrams | Custom diagram format | Mermaid syntax inline in .md files | Standard, renderable in GitHub/GitLab, no external tools needed |

**Key insight:** The Outer Loop stages primarily orchestrate existing tools and skills. The adapter's value is in the structured flow (Step 0-N), not in reimplementing what skills already do.

## Common Pitfalls

### Pitfall 1: Making DEPLOY stage execute deployment commands directly
**What goes wrong:** Claude tries to run `npm run build && vercel deploy` automatically
**Why it happens:** Inner Loop adapters (DO, TEST) perform direct actions. DEPLOY involves production systems.
**How to avoid:** DEPLOY stage provides checklists and guides the user. User executes deploy commands. Claude tracks completion.
**Warning signs:** Adapter contains `bash` commands for production deployment

### Pitfall 2: Blocking on Stage 9 PROMOTE
**What goes wrong:** Pipeline gets stuck because user doesn't want to create a demo
**Why it happens:** Treating PROMOTE as required when it's explicitly optional
**How to avoid:** Step 1 of promote-stage.md must offer skip as the default path. "Skip" registers a skip artifact so the pipeline cleanly completes.
**Warning signs:** No skip path in the adapter flow

### Pitfall 3: Forgetting Living State update in RETROSPECT
**What goes wrong:** Lessons captured in retrospective but not reflected in LIVING-STATE.md
**Why it happens:** RETROSPECT focuses on generating the retro document and forgets the Living State sync
**How to avoid:** Explicit step in retrospect-stage.md to regenerate LIVING-STATE.md with lessons from the retro
**Warning signs:** LIVING-STATE.md does not mention recent retrospective findings

### Pitfall 4: DOCUMENT stage trying to generate perfect diagrams
**What goes wrong:** Spending excessive time on pixel-perfect architecture diagrams
**Why it happens:** Treating DOCUMENT as a design phase rather than a recording phase
**How to avoid:** DOCUMENT generates "good enough" Mermaid diagrams that capture structure, not aesthetics. Wireframe-level quality is fine.
**Warning signs:** Multiple iterations on diagram styling

### Pitfall 5: Inconsistent adapter structure across 5 new files
**What goes wrong:** Each adapter has slightly different section ordering, naming, or completion flow
**Why it happens:** Writing 5 files without a strict template
**How to avoid:** Write one adapter (e.g., deploy-stage.md) first, validate against Inner Loop pattern, then use it as template for remaining 4
**Warning signs:** "When This Runs" section missing from some adapters, inconsistent Anti-Patterns table format

### Pitfall 6: RETROSPECT not delegating to gsd-retrospective
**What goes wrong:** retrospect-stage.md reimplements retrospective generation instead of invoking the existing skill
**Why it happens:** Desire for "integrated" experience leads to duplicating skill logic
**How to avoid:** ARCH-02 requires adapter pattern -- define invocation interface, don't reimplement. The adapter tells Claude "invoke gsd-retrospective", not "generate a retrospective by scanning git log..."
**Warning signs:** retrospect-stage.md contains git log commands or retro template structure

## Code Examples

### Adapter Header Pattern (from commit-stage.md)

```markdown
# {STAGE} Stage Adapter

This reference defines the complete {STAGE} Stage workflow -- Stage {N} of the dev-lifecycle pipeline. When Claude reads this file, it follows the instructions below to {purpose}.

## When This Runs

- User says "{keywords}" or similar
- `state.json` `current.stage = {N}` (or transitioning from {N-1} to {N})
- SKILL.md Stage {N} section directs here via `Read: $CLAUDE_SKILL_DIR/references/{stage}-stage.md`

## Prerequisites

- `.lifecycle/` directory initialized (from PLAN stage)
- `state.json` and `manifest.json` exist
- {Previous stage} must have outputs -- gate {N-1}_to_{N} must pass (`artifacts.{N-1}_{prev}.outputs.length > 0`)
```

### Step 0 Pattern (from commit-stage.md)

```markdown
## Step 0: Stage Initialization

**Actions:**

1. Read `.lifecycle/state.json` and verify:
   - `current.feature` has a value
   - `current.stage` = `{N}` (or transition from {N-1} to {N})

2. Check gate condition ({N-1}_to_{N}):
   - Read `.lifecycle/manifest.json`
   - Verify: `artifacts.{N-1}_{prev}.outputs.length > 0`
   - If gate fails: set `current.status = "blocked"`, display missing artifacts, STOP

3. Update `state.json`:
   - `current.stage` = `{N}`
   - `current.stage_name` = `"{STAGE}"`
   - `current.status` = `"in_progress"`
   - `progress.current_stage_started_at` = current ISO 8601 timestamp
```

### Completion Pattern (from commit-stage.md)

```markdown
## Step {last}: Completion

1. Register outputs in `manifest.json` under `artifacts.{N}_{stage}.outputs`:
   ```json
   [{ "type": "{type}", "path": "{path}", "created_at": "{ISO-8601}", "status": "complete" }]
   ```

2. Set `artifacts.{N}_{stage}.completed_at` = current ISO 8601 timestamp

3. Update `state.json`:
   - `current.status` = `"completed"`
   - `session.last_active` = current ISO 8601 timestamp
   - `session.resume_hint` = `"{STAGE} complete. Ready for {NEXT} stage."`

4. Write history entry to `.lifecycle/history/`

5. Regenerate `.lifecycle/LIVING-STATE.md`

6. Announce completion
```

### SKILL.md Stage Section Pattern (from Stage 4)

```markdown
### Stage {N}: {NAME}

Purpose: {one-line purpose}
Primary: dev-lifecycle ({role description})
Supporting: {skills list}
Outputs: {artifact types}

Read: `$CLAUDE_SKILL_DIR/references/{stage}-stage.md`

Pipeline:
1. {step description}
2. {step description}
...
```

### Deploy Checklist Conditional Pattern (for deploy-stage.md)

```markdown
## Step 2: Deploy Checklist

Based on `state.json` `project.detected_type`, provide the appropriate checklist:

### If `node-web/next`
- [ ] Run `npm run build` -- verify no build errors
- [ ] Check environment variables are set in target platform
- [ ] Deploy via platform CLI (vercel/netlify) or Docker
- [ ] Verify build output URL accessible

### If `flutter`
- [ ] Run `flutter build apk` and/or `flutter build ios`
- [ ] Sign the release build
- [ ] Upload to distribution platform (Play Store/TestFlight/Firebase)
- [ ] Note distribution URL/version

### If `python/fastapi`
...

### If no type detected or `generic`
Ask user: "Describe your deploy process. I'll create a checklist from your steps."
```

### Smoke Test 3-Category Pattern (for deploy-test-stage.md)

```markdown
## Step 2: Health Check

Verify the deployed service is responding:

- [ ] Main endpoint returns 2xx status
- [ ] Response time < 5 seconds
- [ ] No error messages in response body

## Step 3: Core Flow Verification

Test the primary user journey end-to-end:

- [ ] Can reach the entry point (home page / API root)
- [ ] Authentication works (if applicable)
- [ ] Primary feature works (based on spec feature name)
- [ ] Data persists correctly (if applicable)

## Step 4: Resource Check

Verify resources are within acceptable bounds:

- [ ] No OOM errors in logs
- [ ] Disk usage not at capacity
- [ ] Database connections available (if applicable)
- [ ] No rate limiting being triggered
```

## State of the Art

| Old Approach | Current Approach | When Changed | Impact |
|--------------|------------------|--------------|--------|
| Stage 5-9 placeholder text in SKILL.md | Full adapter references with Read directives | This phase | Stages 5-9 become as structured as 1-4 |
| Role-matrix "Project-specific template" | dev-lifecycle as primary orchestrator for all 9 stages | This phase | Consistent orchestration model |
| No skip mechanism for optional stages | Explicit skip artifact registration | This phase | Clean pipeline completion without mandatory promotion |

## Open Questions

1. **DEPLOY stage: how detailed should project-type checklists be?**
   - What we know: project-detection.md defines 9+ types. Each needs deploy steps.
   - What's unclear: Should each type have 5-10 detailed steps or 3-5 high-level ones?
   - Recommendation: Start with 3-5 high-level steps per type. Users will provide specifics. The checklist is a guide, not a script.

2. **DOCUMENT stage: canvas-design skill availability**
   - What we know: role-matrix lists canvas-design as optional for Stage 7
   - What's unclear: Whether canvas-design skill is installed or available
   - Recommendation: Make Mermaid the default (always available), canvas-design purely optional enhancement. Document stage should work perfectly without canvas-design.

3. **Mode-based skipping for Outer Loop**
   - What we know: stage-transitions.md defines skip rules per mode (feature mode skips 5, 6, 9)
   - What's unclear: How skip interacts with adapter Step 0 initialization
   - Recommendation: Step 0 should check `current.mode` and if current stage is in the skippable list, offer skip as first option. Skipping registers a `{ "type": "skipped", ... }` artifact to satisfy gate conditions.

## Validation Architecture

### Test Framework
| Property | Value |
|----------|-------|
| Framework | Manual verification (skill files are Markdown, not executable code) |
| Config file | None |
| Quick run command | Read each adapter file and verify structure against pattern |
| Full suite command | Verify all 5 adapters + SKILL.md + role-matrix consistency |

### Phase Requirements to Test Map
| Req ID | Behavior | Test Type | Automated Command | File Exists? |
|--------|----------|-----------|-------------------|-------------|
| PIPE-05 | deploy-stage.md follows adapter pattern | manual | Verify sections: When This Runs, Prerequisites, Step 0-N, Anti-Patterns, File Locations | Wave 0 |
| PIPE-06 | deploy-test-stage.md has 3-category smoke tests | manual | Verify health/flow/resource sections exist | Wave 0 |
| PIPE-07 | document-stage.md generates Mermaid diagrams | manual | Verify Mermaid syntax examples in adapter | Wave 0 |
| PIPE-08 | retrospect-stage.md delegates to gsd-retrospective | manual | Verify skill invocation instructions, no reimplementation | Wave 0 |
| PIPE-09 | promote-stage.md has skip-first design | manual | Verify Step 1 skip check exists | Wave 0 |

### Sampling Rate
- **Per task:** Verify new adapter against Inner Loop adapter structure
- **Per wave:** Cross-check all adapters for consistency
- **Phase gate:** All 5 adapters exist, SKILL.md updated, role-matrix updated

### Wave 0 Gaps
None -- this phase creates Markdown reference files, not executable code. Validation is structural consistency checking against Inner Loop adapters.

## Sources

### Primary (HIGH confidence)
- `skill/references/plan-stage.md` (311 lines) -- canonical adapter pattern
- `skill/references/commit-stage.md` (243 lines) -- completion/anti-patterns/file-locations pattern
- `skill/references/do-stage.md` -- Step 0 gate check pattern
- `skill/references/test-stage.md` -- Step 0 prerequisite pattern
- `skill/references/stage-transitions.md` -- gate conditions, status values, mode-based skipping
- `skill/references/role-matrix.md` -- current Stage 5-9 skill mappings
- `skill/references/project-detection.md` -- 9+ project types with sub-type detection
- `skill/templates/manifest.json` -- Stage 5-9 artifact slots already defined

### Secondary (MEDIUM confidence)
- `~/.claude/skills/gsd-retrospective/SKILL.md` -- retro generation interface (3-step: detect, gather, generate)
- `~/.claude/skills/work-log/SKILL.md` -- work log recording interface (Obsidian vault)
- `~/.claude/skills/adr/SKILL.md` -- ADR creation interface (docs/decisions/)

### Tertiary (LOW confidence)
- canvas-design skill availability -- referenced in role-matrix but not verified as installed

## Metadata

**Confidence breakdown:**
- Standard stack: HIGH -- all components are existing project files, fully read and verified
- Architecture: HIGH -- adapter pattern proven across 4 Inner Loop stages, pattern is mechanical to replicate
- Pitfalls: HIGH -- derived from analyzing 4 existing adapters and noting common structural requirements

**Research date:** 2026-03-22
**Valid until:** 2026-04-22 (stable -- Markdown reference files, no external dependency versions)
