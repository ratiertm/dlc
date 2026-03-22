---
phase: 8
slug: memory-decision-trail
status: draft
nyquist_compliant: true
wave_0_complete: true
created: 2026-03-22
---

# Phase 8 — Validation Strategy

## Test Infrastructure

| Property | Value |
|----------|-------|
| **Framework** | File structure + grep verification |
| **Quick run command** | `test -f skill/templates/living-state.md && echo "OK"` |
| **Full suite command** | `grep -c "Settings Changelog" skill/templates/settings-changelog.md && grep -c "Decision Log" skill/templates/decisions.md && grep -c "^## " skill/templates/living-state.md && grep "LIVING-STATE" skill/references/stage-transitions.md && grep "WHY+SEE" skill/SKILL.md` |
| **Estimated runtime** | ~1 second |

## Per-Task Verification Map

| Task ID | Plan | Wave | Requirement | Test Type | Status |
|---------|------|------|-------------|-----------|--------|
| 08-01-01 | 01 | 1 | MEMO-01, MEMO-02 | file+grep | pending |
| 08-01-02 | 01 | 1 | MEMO-03 | file+grep | pending |
| 08-02-01 | 02 | 2 | MEMO-01, MEMO-02, MEMO-03 | grep (stage adapters) | pending |
| 08-02-02 | 02 | 2 | MEMO-03, MEMO-04 | grep (SKILL.md + do-stage.md) | pending |

## Artifact Verification

| Artifact | Created By | Verify Command |
|----------|-----------|----------------|
| `skill/templates/settings-changelog.md` | 08-01 Task 1 | `grep "Settings Changelog" skill/templates/settings-changelog.md` |
| `skill/templates/decisions.md` | 08-01 Task 1 | `grep "Decision Log" skill/templates/decisions.md` |
| `skill/templates/living-state.md` | 08-01 Task 2 | `grep -c "^## " skill/templates/living-state.md` (expect 6+) |
| `skill/references/stage-transitions.md` (updated) | 08-02 Task 1 | `grep "Update Living State" skill/references/stage-transitions.md` |
| `skill/references/do-stage.md` (updated) | 08-02 Task 1 | `grep "memory files" skill/references/do-stage.md` |
| `skill/references/commit-stage.md` (updated) | 08-02 Task 1 | `grep "memory files" skill/references/commit-stage.md` |
| `skill/SKILL.md` (updated) | 08-02 Task 2 | `grep "Memory.*Decision Trail" skill/SKILL.md && grep "WHY+SEE" skill/SKILL.md` |

## Wave 0 Requirements

- [x] No test framework needed -- file existence and grep verification only

## Manual-Only Verifications

| Behavior | Requirement | Why Manual | Test Instructions |
|----------|-------------|------------|-------------------|
| Living State restores context | MEMO-03 | Requires new session test | Start fresh session, read Living State, verify context |

## Validation Sign-Off

- [x] All tasks have automated verify
- [x] Feedback latency < 2s
- [x] `nyquist_compliant: true` set in frontmatter

**Approval:** pending
