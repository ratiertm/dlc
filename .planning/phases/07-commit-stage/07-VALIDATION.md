---
phase: 7
slug: commit-stage
status: draft
nyquist_compliant: false
wave_0_complete: false
created: 2026-03-22
---

# Phase 7 — Validation Strategy

> Per-phase validation contract for feedback sampling during execution.

---

## Test Infrastructure

| Property | Value |
|----------|-------|
| **Framework** | File structure + grep verification |
| **Config file** | none |
| **Quick run command** | `test -f skill/references/commit-stage.md && echo "OK"` |
| **Full suite command** | `grep -c "verification.*gate\|why.*commit\|artifact.*check\|override" skill/references/commit-stage.md` |
| **Estimated runtime** | ~1 second |

---

## Sampling Rate

- **After every task commit:** Quick check reference exists
- **After every plan wave:** Verify key sections present
- **Max feedback latency:** 2 seconds

---

## Per-Task Verification Map

| Task ID | Plan | Wave | Requirement | Test Type | Automated Command | File Exists | Status |
|---------|------|------|-------------|-----------|-------------------|-------------|--------|
| 07-01-01 | 01 | 1 | PIPE-04, ARCH-04 | file+grep | commit-stage.md has gate + commit + artifact sections | ❌ W0 | ⬜ pending |

---

## Wave 0 Requirements

- [ ] No test framework needed — skill reference files only

---

## Manual-Only Verifications

| Behavior | Requirement | Why Manual | Test Instructions |
|----------|-------------|------------|-------------------|
| COMMIT blocked on FAIL status | ARCH-04 | Requires interactive flow | Trigger TEST with FAIL, verify COMMIT refuses |

---

## Validation Sign-Off

- [ ] All tasks have automated verify
- [ ] Feedback latency < 2s
- [ ] `nyquist_compliant: true` set in frontmatter

**Approval:** pending
