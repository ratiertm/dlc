---
phase: 3
slug: prototype-format
status: draft
nyquist_compliant: false
wave_0_complete: false
created: 2026-03-22
---

# Phase 3 — Validation Strategy

> Per-phase validation contract for feedback sampling during execution.

---

## Test Infrastructure

| Property | Value |
|----------|-------|
| **Framework** | File/HTML structure verification (no test framework — skill files) |
| **Config file** | none |
| **Quick run command** | `test -f skill/templates/prototype-template.html && echo "OK"` |
| **Full suite command** | `grep -c "data-spec-id\|data-screen\|data-action\|data-field\|prototype-manifest" skill/templates/prototype-template.html` |
| **Estimated runtime** | ~1 second |

---

## Sampling Rate

- **After every task commit:** Quick check template exists
- **After every plan wave:** Verify data-* attributes and manifest present in template
- **Before `/gsd:verify-work`:** Full suite confirms all 4 requirements met
- **Max feedback latency:** 2 seconds

---

## Per-Task Verification Map

| Task ID | Plan | Wave | Requirement | Test Type | Automated Command | File Exists | Status |
|---------|------|------|-------------|-----------|-------------------|-------------|--------|
| 03-01-01 | 01 | 1 | PROTO-01,02,03,04 | file+grep | template + reference + manifest checks | ❌ W0 | ⬜ pending |
| 03-01-02 | 01 | 1 | PROTO-01,02,03 | file+grep | example HTML has hash routing + data-spec-id | ❌ W0 | ⬜ pending |

*Status: ⬜ pending · ✅ green · ❌ red · ⚠️ flaky*

---

## Wave 0 Requirements

- [ ] No test framework needed — deliverables are HTML template + reference doc + example
- [ ] Verification is structural: HTML has required sections, data-* attributes present, manifest parseable

---

## Manual-Only Verifications

| Behavior | Requirement | Why Manual | Test Instructions |
|----------|-------------|------------|-------------------|
| Prototype opens in browser via file:// | PROTO-01 | Requires actual browser | Open prototype-template.html with file:// in Chrome, verify renders |
| Hash routing navigates between screens | PROTO-02 | Requires browser interaction | Click navigation links, verify URL hash changes and screens switch |
| Responsive layout works on mobile viewport | PROTO-01 | Requires browser resize | Resize to 375px width, verify layout adapts |

---

## Validation Sign-Off

- [ ] All tasks have automated verify or Wave 0 dependencies
- [ ] Sampling continuity: no 3 consecutive tasks without automated verify
- [ ] Wave 0 covers all MISSING references
- [ ] No watch-mode flags
- [ ] Feedback latency < 2s
- [ ] `nyquist_compliant: true` set in frontmatter

**Approval:** pending
