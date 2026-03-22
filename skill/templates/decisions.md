# Decision Log

Lightweight one-line decision entries. For decisions that don't meet the ADR threshold (2+ viable approaches AND lasting consequences) but still deserve recording. Direction changes, settings rationale, small tradeoffs.

## Usage

Copy to `.lifecycle/decisions.md` on first lightweight decision.

## Decisions

| Date | Decision | Reason |
|------|----------|--------|
| 2026-01-01 | Use Zustand over Redux for state management | Simpler API, no boilerplate |
| 2026-01-01 | See ADR-001 | Database migration strategy (promoted to ADR) |

## Rules

1. Append only -- never edit or remove existing entries
2. If a decision meets BOTH ADR criteria (2+ viable approaches AND lasting consequences), create an ADR instead and add a cross-reference row here
3. Keep entries to one line. If you need more than 2 lines, it's probably an ADR
