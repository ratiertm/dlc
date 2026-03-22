---
phase: 8
slug: memory-decision-trail
status: draft
nyquist_compliant: false
wave_0_complete: false
created: 2026-03-22
---

# Phase 8 — Validation Strategy

## Test Infrastructure

| Property | Value |
|----------|-------|
| **Framework** | File structure + grep verification |
| **Quick run command** | `test -f skill/references/memory-trail.md && echo "OK"` |
| **Full suite command** | `grep -c "settings.*changelog\|decision.*log\|living.*state\|WHY.*SEE" skill/references/memory-trail.md` |
| **Estimated runtime** | ~1 second |

## Per-Task Verification Map

| Task ID | Plan | Wave | Requirement | Test Type | Status |
|---------|------|------|-------------|-----------|--------|
| 08-01-01 | 01 | 1 | MEMO-01,02,03,04 | file+grep | ⬜ pending |

## Wave 0 Requirements

- [ ] No test framework needed — skill reference files

## Manual-Only Verifications

| Behavior | Requirement | Why Manual | Test Instructions |
|----------|-------------|------------|-------------------|
| Living State restores context | MEMO-03 | Requires new session test | Start fresh session, read Living State, verify context |

## Validation Sign-Off

- [ ] All tasks have automated verify
- [ ] Feedback latency < 2s
- [ ] `nyquist_compliant: true` set in frontmatter

**Approval:** pending
