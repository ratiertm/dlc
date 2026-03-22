---
phase: 4
slug: plan-stage
status: draft
nyquist_compliant: false
wave_0_complete: false
created: 2026-03-22
---

# Phase 4 — Validation Strategy

> Per-phase validation contract for feedback sampling during execution.

---

## Test Infrastructure

| Property | Value |
|----------|-------|
| **Framework** | File structure + grep verification (skill files, not application code) |
| **Config file** | none |
| **Quick run command** | `test -f skill/references/plan-stage.md && echo "OK"` |
| **Full suite command** | `grep -c "preflight\|spec.*generat\|prototype.*generat\|agreement.*gate" skill/references/plan-stage.md` |
| **Estimated runtime** | ~1 second |

---

## Sampling Rate

- **After every task commit:** Quick check reference file exists
- **After every plan wave:** Verify all 4 pipeline steps documented
- **Before `/gsd:verify-work`:** Full suite confirms all requirements met
- **Max feedback latency:** 2 seconds

---

## Per-Task Verification Map

| Task ID | Plan | Wave | Requirement | Test Type | Automated Command | File Exists | Status |
|---------|------|------|-------------|-----------|-------------------|-------------|--------|
| 04-01-01 | 01 | 1 | PIPE-01, SPEC-02, PROTO-05 | file+grep | plan-stage.md has all 4 steps | ❌ W0 | ⬜ pending |

*Status: ⬜ pending · ✅ green · ❌ red · ⚠️ flaky*

---

## Wave 0 Requirements

- [ ] No test framework needed — deliverables are skill reference files
- [ ] Verification is structural: reference file has all pipeline steps documented

---

## Manual-Only Verifications

| Behavior | Requirement | Why Manual | Test Instructions |
|----------|-------------|------------|-------------------|
| Agreement gate blocks without approval | PROTO-05 | Requires user interaction flow | Invoke PLAN stage, verify it waits for explicit approval |
| Preflight surfaces retrospective lessons | PIPE-01 | Requires retrospective data | Create mock retrospective, invoke PLAN, verify lessons shown |

---

## Validation Sign-Off

- [ ] All tasks have automated verify or Wave 0 dependencies
- [ ] Sampling continuity: no 3 consecutive tasks without automated verify
- [ ] Wave 0 covers all MISSING references
- [ ] No watch-mode flags
- [ ] Feedback latency < 2s
- [ ] `nyquist_compliant: true` set in frontmatter

**Approval:** pending
