# Completeness Scoring Reference

Cross-stage quality scoring system. Claude assesses Completeness as a contextual N/10 judgment at every stage completion and at decision points.

## When This Runs

- At the completion of each stage (after all stage-specific work, before the completion announcement)
- At any decision point where multiple options are presented to the user

## Stage Completion Score

At the completion of each stage, before the completion announcement, assess Completeness:

**Dimensions (contextual weighting, not formula):**

| Dimension | What to Consider | Applies To |
|-----------|-----------------|------------|
| Artifact completeness | Are all expected outputs present and well-formed? | All stages |
| Spec step coverage | What percentage of steps are verified vs implemented vs failed? | DO, TEST |
| Deviation count | How many deviations from original spec? | DO, TEST |
| Verification pass rate | Mini-verify successes vs failures, retry counts | DO |
| Edge case coverage | Are error conditions handled? Validation complete? | TEST |
| Traceability | SPEC comments present? ADRs linked? | COMMIT |
| Documentation coverage | Architecture, sequence diagrams, README current? | DOCUMENT |

**Score:** N/10 -- Claude's contextual judgment considering relevant dimensions for the current stage. This is NOT a formula. Do not create weighted averages or point systems.

**Display at stage completion (add to each stage's completion announcement):**

```
Stage {N} {NAME} -- COMPLETE
Completeness: {N}/10 ({brief justification referencing key dimensions})
```

**Examples:**
- DO: `Completeness: 8/10 (all 5 steps verified, 1 deviation, 2 retries needed)`
- TEST: `Completeness: 9/10 (all steps pass, prototype match 100%, 1 traceability warning)`
- COMMIT: `Completeness: 7/10 (all files staged, 2/5 SPEC comments missing)`

## Decision Point Comparison

When presenting options at any decision point during any stage, include a Completeness projection for each option:

**Format:**
```
Option A: {N}/10 ({brief quality assessment})
Option B: {N}/10 ({brief quality assessment})
Option C: {N}/10 ({brief quality assessment})
```

**Examples:**
```
Option A: 6/10 (happy path only, missing error handling for 3 conditions)
Option B: 9/10 (full coverage including edge cases, adds ~30min)
Option C: 7/10 (covers errors but skips loading states)
```

**When to show comparison:**
- When user must choose between implementation approaches
- When deviation handling presents alternatives
- When TEST stage failure offers fix vs accept vs revise options
- NOT for trivial choices (variable naming, formatting)

## State Persistence

Store Completeness in `state.json` after each stage completion:

```json
{
  "current": {
    "completeness": {
      "score": 8,
      "reason": "all steps verified, 1 deviation, 2 retries needed",
      "stage": "DO",
      "assessed_at": "{ISO-8601}"
    }
  }
}
```

LIVING-STATE.md reads `current.completeness` during regeneration to display the score.

## Anti-Patterns

| # | Anti-Pattern | Why It Is Wrong |
|---|-------------|-----------------|
| 1 | **Formula-based scoring** | User explicitly decided against formulas. Claude judges holistically. Do not create weighted averages or point systems. |
| 2 | **Completeness as a gate** | Completeness is informational. Existing gate conditions (manifest.json artifact checks) are the ONLY blockers. A low score does not prevent stage transition. |
| 3 | **Inflated scores** | Be honest. 10/10 means genuinely complete with no gaps. Most real implementations score 7-9/10. |
| 4 | **Scores without justification** | Always include a brief parenthetical explaining the key factors. A bare number is not useful. |
