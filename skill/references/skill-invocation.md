# Skill Invocation Guide

This document details how dev-lifecycle invokes each coordinated skill: what commands are available, when dev-lifecycle triggers them, and what output to expect.

## GSD (Get Shit Done)

**Available Commands:**
- `/gsd:discuss-phase` -- Gather context for a phase through discussion
- `/gsd:plan-phase` -- Create execution plans for a phase
- `/gsd:execute-phase` -- Execute plans with task-level commits
- `/gsd:verify-work` -- Verify completed work against plan criteria
- `/gsd:complete-milestone` -- Complete a milestone with verification

**When dev-lifecycle invokes:**
- **Stage 1 PLAN:** dev-lifecycle generates spec + prototype independently (Read: $CLAUDE_SKILL_DIR/references/plan-stage.md). GSD available for broader project planning if user invokes directly.
- **Stage 3 TEST:** `/gsd:verify-work` to verify implementation against plan criteria

**Expected output:**
- Stage 1: .lifecycle/features/{name}/spec.md + prototype.html (via plan-stage.md adapter)
- Stage 3: Verification report with pass/fail per criterion

---

## PDCA (Plan-Do-Check-Act)

**Available Commands:**
- `/pdca plan` -- Define objectives, success criteria, action items
- `/pdca do` -- Track implementation progress against plan
- `/pdca check` -- Compare results against objectives
- `/pdca analyze` -- Identify gaps and improvement actions

**When dev-lifecycle invokes:**
- **Stage 1 PLAN (supporting):** `/pdca plan` to supplement GSD planning with cycle tracking
- **Stage 2 DO (primary):** `/pdca do` to track implementation progress
- **Stage 3 TEST (supporting):** `/pdca analyze` to identify gaps between plan and result

**Expected output:**
- Stage 1: PDCA plan entry in `.pdca-status.json`
- Stage 2: Progress updates, completion tracking
- Stage 3: Gap analysis with actionable items

---

## ADR (Architecture Decision Records)

**Available Commands:**
- `/adr` -- Create or update an architecture decision record

**When dev-lifecycle invokes:**
- **Stage 1 PLAN:** When trade-offs are detected during planning discussions
- **Stage 2 DO:** Auto-detect decisions during implementation (e.g., "chose A over B")
- **Stage 4 COMMIT:** Reference relevant ADRs in commit messages
- **Stage 8 RETROSPECT:** Gap check -- identify decisions made but not recorded as ADRs

**Expected output:**
- ADR document in `docs/decisions/NNN-slug.md`
- WHY+SEE code comments inserted at decision points

---

## gsd-retrospective

**Available Commands:**
- `/gsd-retrospective` -- Generate a structured retrospective document

**When dev-lifecycle invokes:**
- **Stage 8 RETROSPECT:** After deployment is verified, trigger retrospective generation

**Expected output:**
- Retrospective document with sections:
  - What went well
  - What went wrong
  - What was surprising
  - Key decisions (linked to ADRs)
  - Technical debt identified
  - Lessons for next phase
  - Metrics (commits, files changed, bugs found)
- CLAUDE.md updated with retrospective link and key lessons

---

## work-log

**Available Commands:**
- `/work-log` -- Log work session to Obsidian vault

**When dev-lifecycle invokes:**
- **Stage 8 RETROSPECT:** After retrospective is generated, log the work session

**Expected output:**
- Work log entry in Obsidian vault with session details, accomplishments, and links to artifacts

---

## canvas-design

**Available Commands:**
- `/canvas-design` -- Generate architecture diagrams (PNG art or HTML interactive)

**When dev-lifecycle invokes:**
- **Stage 7 DOCUMENT (optional):** When architecture documentation is needed

**Expected output:**
- Architecture diagram (HTML interactive or PNG)
- Can include: system overview, component relationships, data flow

---

## Invocation Pattern

dev-lifecycle suggests the skill invocation command to the user. It does NOT auto-execute skills. The user triggers the skill, and dev-lifecycle tracks the output artifact.

**Flow:**
1. dev-lifecycle detects current stage from state.json
2. dev-lifecycle identifies the primary skill for that stage (from role-matrix.md)
3. dev-lifecycle presents the suggested command:
   ```
   Stage 2 DO -- Primary skill: PDCA
   Suggested: /pdca do
   ```
4. User invokes the skill (or chooses a different approach)
5. dev-lifecycle observes the output artifact
6. dev-lifecycle registers the artifact in manifest.json
7. When gate conditions are met, dev-lifecycle suggests transitioning to the next stage

**Why suggest instead of auto-execute:**
- Users may have context that changes which skill is appropriate
- Skills may need user input during execution
- Direct invocation preserves user agency
- Skills work independently -- they do not need dev-lifecycle to function
