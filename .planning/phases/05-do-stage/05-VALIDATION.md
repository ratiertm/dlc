---
phase: 5
slug: do-stage
status: draft
nyquist_compliant: false
wave_0_complete: false
created: 2026-03-22
---

# Phase 5 — Validation Strategy

> Per-phase validation contract for feedback sampling during execution.

---

## Test Infrastructure

| Property | Value |
|----------|-------|
| **Framework** | File structure + grep verification (skill reference files) |
| **Config file** | none |
| **Quick run command** | `test -f skill/references/do-stage.md && echo "OK"` |
| **Full suite command** | `grep -c "checklist\|deviation\|ADR\|gate" skill/references/do-stage.md` |
| **Estimated runtime** | ~1 second |

---

## Sampling Rate

- **After every task commit:** Quick check reference file exists
- **After every plan wave:** Verify all key sections present
- **Before `/gsd:verify-work`:** Full suite confirms all requirements met
- **Max feedback latency:** 2 seconds

---

## Per-Task Verification Map

| Task ID | Plan | Wave | Requirement | Test Type | Automated Command | File Exists | Status |
|---------|------|------|-------------|-----------|-------------------|-------------|--------|
| 05-01-01 | 01 | 1 | SPEC-03, SPEC-05, PIPE-02 | file+grep | do-stage.md has checklist + deviation + ADR sections | ❌ W0 | ⬜ pending |

*Status: ⬜ pending · ✅ green · ❌ red · ⚠️ flaky*

---

## Wave 0 Requirements

- [ ] No test framework needed — deliverables are skill reference files
- [ ] Verification is structural: reference file has all required pipeline steps

---

## Manual-Only Verifications

| Behavior | Requirement | Why Manual | Test Instructions |
|----------|-------------|------------|-------------------|
| Deviation gate pauses for user confirmation | SPEC-05 | Requires interactive session | Trigger deviation during DO, verify Claude pauses and asks |
| ADR detection triggers on tradeoff | PIPE-02 | Requires conversational context | Make tradeoff decision during DO, verify ADR suggestion |

---

## Validation Sign-Off

- [ ] All tasks have automated verify or Wave 0 dependencies
- [ ] Sampling continuity: no 3 consecutive tasks without automated verify
- [ ] Wave 0 covers all MISSING references
- [ ] No watch-mode flags
- [ ] Feedback latency < 2s
- [ ] `nyquist_compliant: true` set in frontmatter

**Approval:** pending
