---
phase: 12
slug: stage-internal-iteration
status: draft
nyquist_compliant: false
wave_0_complete: false
created: 2026-03-24
---

# Phase 12 — Validation Strategy

> Per-phase validation contract for feedback sampling during execution.

---

## Test Infrastructure

| Property | Value |
|----------|-------|
| **Framework** | Manual verification (file-based skill, no test runner) |
| **Config file** | .lifecycle/config.yaml (verification_strictness) |
| **Quick run command** | `grep "verified" skill/references/do-stage.md` |
| **Full suite command** | `grep -c "mini-verify\|completeness\|retry" skill/references/do-stage.md` |
| **Estimated runtime** | ~1 second |

---

## Sampling Rate

- **After every task commit:** Verify do-stage.md structure intact
- **After every plan wave:** Verify mini-verify + completeness patterns present
- **Before `/gsd:verify-work`:** All 4 requirements pass
- **Max feedback latency:** 2 seconds

---

## Per-Task Verification Map

| Task ID | Plan | Wave | Requirement | Test Type | Automated Command | File Exists | Status |
|---------|------|------|-------------|-----------|-------------------|-------------|--------|
| 12-01-01 | 01 | 1 | ITER-01 | manual | `grep "Step 2d" skill/references/do-stage.md` | ✅ | ⬜ pending |
| 12-01-02 | 01 | 1 | ITER-02 | manual | `grep "retry" skill/references/do-stage.md` | ✅ | ⬜ pending |
| 12-02-01 | 02 | 2 | ITER-03 | manual | `grep "completeness" skill/references/completeness-scoring.md` | ❌ W0 | ⬜ pending |
| 12-02-02 | 02 | 2 | ITER-04 | manual | `grep "Completeness" skill/SKILL.md` | ✅ | ⬜ pending |

*Status: ⬜ pending · ✅ green · ❌ red · ⚠️ flaky*

---

## Wave 0 Requirements

- [ ] `skill/references/completeness-scoring.md` — New reference file for scoring logic

*Existing do-stage.md and SKILL.md cover mini-verify insertion.*

---

## Manual-Only Verifications

| Behavior | Requirement | Why Manual | Test Instructions |
|----------|-------------|------------|-------------------|
| Mini-verify executes after each step | ITER-01 | Claude instruction-following | Implement a feature, verify spec Status updates to verified |
| Retry loop triggers on failure | ITER-02 | Requires actual implementation failure | Intentionally break code, verify 3 retry attempts |
| Completeness score displayed | ITER-03 | Claude judgment, not formula | Complete a stage, verify N/10 score shown |
| Option comparison shown | ITER-04 | Claude decision point display | Trigger a decision, verify Completeness comparison |

---

## Validation Sign-Off

- [ ] All tasks have `<automated>` verify or Wave 0 dependencies
- [ ] Sampling continuity: no 3 consecutive tasks without automated verify
- [ ] Wave 0 covers all MISSING references
- [ ] No watch-mode flags
- [ ] Feedback latency < 2s
- [ ] `nyquist_compliant: true` set in frontmatter

**Approval:** pending
