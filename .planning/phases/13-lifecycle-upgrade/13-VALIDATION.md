---
phase: 13
slug: lifecycle-upgrade
status: draft
nyquist_compliant: false
wave_0_complete: false
created: 2026-03-25
---

# Phase 13 — Validation Strategy

> Per-phase validation contract for feedback sampling during execution.

---

## Test Infrastructure

| Property | Value |
|----------|-------|
| **Framework** | Manual verification (file-based skill) |
| **Config file** | skill/VERSION |
| **Quick run command** | `cat skill/VERSION` |
| **Full suite command** | `ls skill/VERSION skill/CHANGELOG.md skill/references/lifecycle-upgrade.md` |
| **Estimated runtime** | ~1 second |

---

## Sampling Rate

- **After every task commit:** Verify new files exist
- **After every plan wave:** Verify upgrade reference is complete
- **Before `/gsd:verify-work`:** All 4 requirements pass
- **Max feedback latency:** 2 seconds

---

## Per-Task Verification Map

| Task ID | Plan | Wave | Requirement | Test Type | Automated Command | File Exists | Status |
|---------|------|------|-------------|-----------|-------------------|-------------|--------|
| 13-01-01 | 01 | 1 | UPGR-01 | manual | `grep "migration" skill/references/lifecycle-upgrade.md` | ❌ W0 | ⬜ pending |
| 13-01-02 | 01 | 1 | UPGR-02 | manual | `grep "marker" skill/references/lifecycle-upgrade.md` | ❌ W0 | ⬜ pending |
| 13-01-03 | 01 | 1 | UPGR-03 | manual | `grep "rollback\|backup" skill/references/lifecycle-upgrade.md` | ❌ W0 | ⬜ pending |
| 13-02-01 | 02 | 2 | UPGR-04 | manual | `grep "changelog\|CHANGELOG" skill/SKILL.md` | ✅ | ⬜ pending |

---

## Wave 0 Requirements

- [ ] `skill/VERSION` — Version file
- [ ] `skill/CHANGELOG.md` — Changelog
- [ ] `skill/references/lifecycle-upgrade.md` — Upgrade procedure reference

---

## Manual-Only Verifications

| Behavior | Requirement | Why Manual | Test Instructions |
|----------|-------------|------------|-------------------|
| Schema migration runs on git pull | UPGR-01 | Requires actual version mismatch | Simulate v1.0 .lifecycle/, pull v2.0, verify migration |
| Migration marker prevents re-run | UPGR-02 | Requires two upgrade runs | Run upgrade twice, verify second skips |
| Rollback restores state | UPGR-03 | Requires intentional failure | Break migration mid-way, verify backup restored |
| Changelog shown after upgrade | UPGR-04 | Claude instruction-following | Upgrade, verify changelog summary displayed |

---

## Validation Sign-Off

- [ ] All tasks have verify or Wave 0 dependencies
- [ ] Wave 0 covers all MISSING references
- [ ] Feedback latency < 2s
- [ ] `nyquist_compliant: true` set in frontmatter

**Approval:** pending
