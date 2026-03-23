---
phase: 11
slug: configuration
status: draft
nyquist_compliant: false
wave_0_complete: false
created: 2026-03-24
---

# Phase 11 — Validation Strategy

> Per-phase validation contract for feedback sampling during execution.

---

## Test Infrastructure

| Property | Value |
|----------|-------|
| **Framework** | Manual verification (file-based skill, no test runner) |
| **Config file** | .lifecycle/config.yaml |
| **Quick run command** | `cat .lifecycle/config.yaml` |
| **Full suite command** | `grep -c "." .lifecycle/settings-changelog.md` |
| **Estimated runtime** | ~1 second |

---

## Sampling Rate

- **After every task commit:** Verify config.yaml structure
- **After every plan wave:** Verify layered resolution works
- **Before `/gsd:verify-work`:** All 3 requirements pass
- **Max feedback latency:** 2 seconds

---

## Per-Task Verification Map

| Task ID | Plan | Wave | Requirement | Test Type | Automated Command | File Exists | Status |
|---------|------|------|-------------|-----------|-------------------|-------------|--------|
| 11-01-01 | 01 | 1 | CONF-01 | manual | `grep "mode:" .lifecycle/config.yaml` | ❌ W0 | ⬜ pending |
| 11-01-02 | 01 | 1 | CONF-02 | manual | `LIFECYCLE_MODE=hotfix cat .lifecycle/config.yaml` | ❌ W0 | ⬜ pending |
| 11-02-01 | 02 | 1 | CONF-03 | manual | `grep "config" .lifecycle/settings-changelog.md` | ✅ | ⬜ pending |

*Status: ⬜ pending · ✅ green · ❌ red · ⚠️ flaky*

---

## Wave 0 Requirements

- [ ] `skill/templates/config.yaml` — default config template
- [ ] `skill/references/lifecycle-config.md` — config resolution reference

*Existing settings-changelog.md template covers CONF-03.*

---

## Manual-Only Verifications

| Behavior | Requirement | Why Manual | Test Instructions |
|----------|-------------|------------|-------------------|
| Config get/set via Claude | CONF-01 | Claude instruction-following, not CLI | Ask Claude to set mode, verify config.yaml updated |
| Env var override | CONF-02 | Requires env var context | Set LIFECYCLE_MODE=hotfix, verify Claude uses it |
| Changelog entry | CONF-03 | Requires Claude to append | Change setting, verify settings-changelog.md has entry |

---

## Validation Sign-Off

- [ ] All tasks have `<automated>` verify or Wave 0 dependencies
- [ ] Sampling continuity: no 3 consecutive tasks without automated verify
- [ ] Wave 0 covers all MISSING references
- [ ] No watch-mode flags
- [ ] Feedback latency < 2s
- [ ] `nyquist_compliant: true` set in frontmatter

**Approval:** pending
