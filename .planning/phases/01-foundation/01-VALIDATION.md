---
phase: 1
slug: foundation
status: draft
nyquist_compliant: false
wave_0_complete: false
created: 2026-03-22
---

# Phase 1 — Validation Strategy

> Per-phase validation contract for feedback sampling during execution.

---

## Test Infrastructure

| Property | Value |
|----------|-------|
| **Framework** | Manual file verification (no test framework — this phase produces markdown/JSON files, not code) |
| **Config file** | none |
| **Quick run command** | `ls .lifecycle/state.json .lifecycle/manifest.json 2>/dev/null && echo "OK"` |
| **Full suite command** | `cat ~/.claude/skills/dev-lifecycle/SKILL.md | wc -l && ls ~/.claude/skills/dev-lifecycle/references/ 2>/dev/null` |
| **Estimated runtime** | ~1 second |

---

## Sampling Rate

- **After every task commit:** Run quick command to verify files exist
- **After every plan wave:** Run full suite to verify SKILL.md line count and references/ structure
- **Before `/gsd:verify-work`:** Full suite must confirm all artifacts present
- **Max feedback latency:** 2 seconds

---

## Per-Task Verification Map

| Task ID | Plan | Wave | Requirement | Test Type | Automated Command | File Exists | Status |
|---------|------|------|-------------|-----------|-------------------|-------------|--------|
| 01-01-01 | 01 | 1 | FOUND-01 | file check | `grep -L "muse\|opc@\|flutter build\|PIN" ~/.claude/skills/dev-lifecycle/SKILL.md` | ❌ W0 | ⬜ pending |
| 01-01-02 | 01 | 1 | FOUND-05 | file check | `test -f ~/.claude/skills/dev-lifecycle/references/role-matrix.md` | ❌ W0 | ⬜ pending |
| 01-02-01 | 02 | 1 | FOUND-02 | json check | `python3 -c "import json; json.load(open('.lifecycle/state.json'))"` | ❌ W0 | ⬜ pending |
| 01-02-02 | 02 | 1 | FOUND-03 | json check | `python3 -c "import json; json.load(open('.lifecycle/manifest.json'))"` | ❌ W0 | ⬜ pending |

*Status: ⬜ pending · ✅ green · ❌ red · ⚠️ flaky*

---

## Wave 0 Requirements

- [ ] No test framework needed — this phase produces skill files (markdown) and config (JSON), not application code
- [ ] Verification is structural: files exist, JSON is valid, SKILL.md has no muse references

*Existing infrastructure covers all phase requirements via file/JSON checks.*

---

## Manual-Only Verifications

| Behavior | Requirement | Why Manual | Test Instructions |
|----------|-------------|------------|-------------------|
| Skill loads in any project | FOUND-01 | Requires invoking Claude in different project directories | Open 2 different projects, invoke dev-lifecycle, confirm no errors |
| Role matrix is complete | FOUND-05 | Requires human review of 9x6 matrix accuracy | Read role-matrix.md, verify each cell is correct |

---

## Validation Sign-Off

- [ ] All tasks have automated verify or Wave 0 dependencies
- [ ] Sampling continuity: no 3 consecutive tasks without automated verify
- [ ] Wave 0 covers all MISSING references
- [ ] No watch-mode flags
- [ ] Feedback latency < 2s
- [ ] `nyquist_compliant: true` set in frontmatter

**Approval:** pending
