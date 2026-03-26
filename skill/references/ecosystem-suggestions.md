# Ecosystem Skill Suggestions

This reference defines how dev-lifecycle suggests relevant companion skills at stage transitions. Suggestions are proactive (shown automatically) but user-controllable via the `proactive` config setting.

## Suggestion Mapping

The mapping table defines which skills to suggest when entering each stage. Each entry includes the trigger condition, suggested skill/command, and why it is relevant.

| Entering Stage | Suggested Skill | Command | Why |
|----------------|----------------|---------|-----|
| 1 PLAN | ADR | `/adr` | Surface past decisions relevant to new feature planning |
| 2 DO | investigate | `/investigate` | Systematic debugging if implementation hits unexpected issues |
| 3 TEST | qa | `/qa` | Comprehensive QA testing beyond spec verification |
| 3 TEST | PDCA analyze | `/pdca analyze` | Gap analysis between plan and implementation |
| 4 COMMIT | review | `/review` | Pre-landing code review before commit |
| 5 DEPLOY | setup-deploy | `/setup-deploy` | Configure deployment if not already set up |
| 6 DEPLOY TEST | canary | `/canary` | Post-deploy canary monitoring |
| 7 DOCUMENT | document-release | `/document-release` | Sync documentation with shipped changes |
| 8 RETROSPECT | retro | `/retro` | Engineering retrospective with metrics |
| 8 RETROSPECT | work-log | `/work-log` | Record work session to Obsidian vault |

## Suggestion Logic

When a stage transition succeeds (gate passes, entering Stage N+1):

1. **Check proactive setting:** Resolve `proactive` using config layers (env > config.yaml > default=true)
   - If `proactive: false`: skip suggestions entirely, continue transition
   - If `proactive: true`: proceed to step 2

2. **Look up Stage N+1** in the Suggestion Mapping table above

3. **Filter already-active skills:** If the user has already invoked a suggested skill in this session, do not re-suggest it

4. **Present suggestions** (non-blocking, informational only):
   ```
   Companion skill suggestions for Stage {N+1} {NAME}:
   - {command} -- {why}
   - {command} -- {why}

   (Disable with: lifecycle-config set proactive false)
   ```

5. **Continue transition:** Suggestions never block or delay the stage transition. The user can act on them or ignore them.

## Proactive Control

The `proactive` setting in lifecycle-config controls whether suggestions appear:

- **Default:** `true` (suggestions enabled)
- **Disable:** `lifecycle-config set proactive false`
- **Re-enable:** `lifecycle-config set proactive true`
- **Per-session override:** `LIFECYCLE_PROACTIVE=false` environment variable

When disabled, no suggestions are shown at any stage transition. All other transition behavior (gates, history, analytics, Living State) remains unchanged.

## Extensibility

To add a new skill to the suggestion mapping:

1. Add a row to the Suggestion Mapping table in this file
2. Specify: entering stage, skill name, command, and why it is relevant
3. No changes needed to SKILL.md or stage-transitions.md -- they read this file dynamically

**Guidelines for adding skills:**
- Only suggest skills that are directly relevant to the entering stage's purpose
- Prefer skills that complement (not duplicate) dev-lifecycle's primary work at that stage
- Each stage should have 1-3 suggestions maximum (avoid overwhelming the user)
- The command must be a real, invocable skill command (not hypothetical)

## Anti-Patterns

- **Blocking transitions on suggestions:** Suggestions are informational. Never wait for user response before completing the transition.
- **Suggesting dev-lifecycle's own stages:** Do not suggest "run PLAN" when entering PLAN. The stage adapter already handles this.
- **Over-suggesting:** More than 3 suggestions per stage creates noise. Quality over quantity.
- **Suggesting during auto-skip:** If a stage is auto-skipped, do not show suggestions for it. Only suggest for stages the user will actually enter.
