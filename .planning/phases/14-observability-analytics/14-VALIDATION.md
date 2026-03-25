---
phase: 14
slug: observability-analytics
status: draft
nyquist_compliant: false
wave_0_complete: false
created: 2026-03-25
---

# Phase 14 — Validation Strategy

> Per-phase validation contract for feedback sampling during execution.

---

## Test Infrastructure

| Property | Value |
|----------|-------|
| **Framework** | Manual verification (skill reference docs, not executable code) |
| **Config file** | none |
| **Quick run command** | `grep -c "OBSV-0" skill/references/observability.md` |
| **Full suite command** | `ls .planning/phases/14-observability-analytics/*-SUMMARY.md` |
| **Estimated runtime** | ~2 seconds |

---

## Sampling Rate

- **After every task commit:** Verify file exists and has required sections
- **After every plan wave:** Run full content grep for all OBSV requirements
- **Before `/gsd:verify-work`:** All 5 success criteria must be provable
- **Max feedback latency:** 5 seconds

---

## Per-Task Verification Map

| Task ID | Plan | Wave | Requirement | Test Type | Automated Command | File Exists | Status |
|---------|------|------|-------------|-----------|-------------------|-------------|--------|
| 14-01-01 | 01 | 1 | OBSV-01 | content | `grep "sessions/" skill/references/observability.md` | ❌ W0 | ⬜ pending |
| 14-01-02 | 01 | 1 | OBSV-02 | content | `grep "stage-transitions.jsonl" skill/references/observability.md` | ❌ W0 | ⬜ pending |
| 14-01-03 | 01 | 1 | OBSV-03 | content | `grep "rework-events.jsonl" skill/references/observability.md` | ❌ W0 | ⬜ pending |
| 14-01-04 | 01 | 1 | OBSV-04 | content | `grep "snapshot" skill/references/observability.md` | ❌ W0 | ⬜ pending |
| 14-01-05 | 01 | 1 | OBSV-05 | content | `grep "time-per-stage" skill/references/observability.md` | ❌ W0 | ⬜ pending |
| 14-02-01 | 02 | 2 | OBSV-01 | content | `grep "session" skill/SKILL.md` | ✅ | ⬜ pending |

*Status: ⬜ pending · ✅ green · ❌ red · ⚠️ flaky*

---

## Wave 0 Requirements

- [ ] `skill/references/observability.md` — reference file covering all OBSV requirements
- [ ] Session context, analytics JSONL, rework tracking, snapshot diff, time metrics formats

*Existing infrastructure (SKILL.md, state.json, manifest.json) provides foundation.*

---

## Manual-Only Verifications

| Behavior | Requirement | Why Manual | Test Instructions |
|----------|-------------|------------|-------------------|
| Session file created on session start | OBSV-01 | Requires actual Claude session | Start new session, check .lifecycle/sessions/ |
| JSONL append on transition | OBSV-02 | Requires actual stage transition | Advance a stage, check analytics/ |

---

## Validation Sign-Off

- [ ] All tasks have `<automated>` verify or Wave 0 dependencies
- [ ] Sampling continuity: no 3 consecutive tasks without automated verify
- [ ] Wave 0 covers all MISSING references
- [ ] No watch-mode flags
- [ ] Feedback latency < 5s
- [ ] `nyquist_compliant: true` set in frontmatter

**Approval:** pending
