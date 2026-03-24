# Living State

**Updated:** {timestamp}
**Feature:** {current feature name}
**Stage:** {N} {NAME} -- {status}
**Mode:** {execution mode}

## Usage

Copy to `.lifecycle/LIVING-STATE.md` at project initialization. Regenerate (overwrite) at every stage transition and session end.

## Current State

- Inner loop progress: {stages completed summary}
- Completeness: {N}/10 ({brief justification})
- Deviations: {count and brief summary}
- Key blockers: {active blockers or "none"}

## Active Settings

| Setting | Value | Changed | Reason |
|---------|-------|---------|--------|
| {file} {key} | {value} | {date} | {reason} |

> Populated from last 20 entries of `.lifecycle/settings-changelog.md`

## Recent Decisions

| Date | Decision | Reason |
|------|----------|--------|
| {date} | {decision} | {reason} |

> Populated from last 10 entries of `.lifecycle/decisions.md`

## Event Timeline

| Date | Event |
|------|-------|
| {date} | {event description} |

> Populated from `.lifecycle/history/` JSON entries and `manifest.json` stage completions

## Key ADRs

- {ADR-NNN}: {title} [{status}]

> Active ADRs from `docs/decisions/` with one-line summaries

## Resume Hint

{what to do next}

> Copied from `state.json` `session.resume_hint`

<!-- GENERATION INSTRUCTIONS (for Claude, not for display)
Update triggers:
1. Every stage transition (read stage-transitions.md)
2. Every session end
Sources:
- state.json -> Current State, header fields, Resume Hint
- state.json current.completeness -> Completeness score
- settings-changelog.md -> Active Settings (last 20)
- decisions.md -> Recent Decisions (last 10)
- .lifecycle/history/ -> Event Timeline
- docs/decisions/ -> Key ADRs
- manifest.json -> stage completions for Event Timeline
Target size: ~100 lines max. Truncate older entries if needed.
-->
