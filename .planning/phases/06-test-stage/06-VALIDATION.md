---
phase: 6
slug: test-stage
status: draft
nyquist_compliant: false
wave_0_complete: false
created: 2026-03-22
---

# Phase 6 — Validation Strategy

> Per-phase validation contract for feedback sampling during execution.

---

## Test Infrastructure

| Property | Value |
|----------|-------|
| **Framework** | File structure + grep verification (skill reference files) |
| **Config file** | none |
| **Quick run command** | `test -f skill/references/test-stage.md && echo "OK"` |
| **Full suite command** | `grep -c "structural.*verification\|behavioral.*verification\|pass.*fail\|manifest.*comparison" skill/references/test-stage.md` |
| **Estimated runtime** | ~1 second |

---

## Sampling Rate

- **After every task commit:** Quick check reference file exists
- **After every plan wave:** Verify all verification types documented
- **Before `/gsd:verify-work`:** Full suite confirms all requirements met
- **Max feedback latency:** 2 seconds

---

## Per-Task Verification Map

| Task ID | Plan | Wave | Requirement | Test Type | Automated Command | File Exists | Status |
|---------|------|------|-------------|-----------|-------------------|-------------|--------|
| 06-01-01 | 01 | 1 | SPEC-04, PROTO-06, PIPE-03 | file+grep | test-stage.md has all 3 verification types | ❌ W0 | ⬜ pending |

---

## Wave 0 Requirements

- [ ] No test framework needed — deliverables are skill reference files

---

## Manual-Only Verifications

| Behavior | Requirement | Why Manual | Test Instructions |
|----------|-------------|------------|-------------------|
| Behavioral checklist is actionable | PIPE-03 | Requires human judgment | Read generated checklist, verify steps are concrete |
| Dual gate blocks COMMIT without both approvals | PIPE-03 | Requires interactive flow | Run TEST, verify COMMIT blocked until both pass |

---

## Validation Sign-Off

- [ ] All tasks have automated verify or Wave 0 dependencies
- [ ] Sampling continuity
- [ ] Wave 0 covers all MISSING references
- [ ] Feedback latency < 2s
- [ ] `nyquist_compliant: true` set in frontmatter

**Approval:** pending
