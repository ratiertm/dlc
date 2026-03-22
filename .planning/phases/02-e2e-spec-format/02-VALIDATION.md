---
phase: 2
slug: e2e-spec-format
status: draft
nyquist_compliant: false
wave_0_complete: false
created: 2026-03-22
---

# Phase 2 — Validation Strategy

> Per-phase validation contract for feedback sampling during execution.

---

## Test Infrastructure

| Property | Value |
|----------|-------|
| **Framework** | Manual file/markdown verification (skill files, not application code) |
| **Config file** | none |
| **Quick run command** | `test -f ~/.claude/skills/dev-lifecycle/templates/spec-template.md && echo "OK"` |
| **Full suite command** | `grep -c "## Screen\|## Connection\|## Processing\|## Response\|## Error" ~/.claude/skills/dev-lifecycle/templates/spec-template.md` |
| **Estimated runtime** | ~1 second |

---

## Sampling Rate

- **After every task commit:** Quick check template exists
- **After every plan wave:** Verify all 5 chain sections present
- **Before `/gsd:verify-work`:** Full suite must confirm template + example + reference
- **Max feedback latency:** 2 seconds

---

## Per-Task Verification Map

| Task ID | Plan | Wave | Requirement | Test Type | Automated Command | File Exists | Status |
|---------|------|------|-------------|-----------|-------------------|-------------|--------|
| 02-01-01 | 01 | 1 | SPEC-01 | file+grep | `grep -c "e2e-" ~/.claude/skills/dev-lifecycle/templates/spec-template.md` | ❌ W0 | ⬜ pending |

*Status: ⬜ pending · ✅ green · ❌ red · ⚠️ flaky*

---

## Wave 0 Requirements

- [ ] No test framework needed — deliverables are markdown template + reference doc
- [ ] Verification is structural: template has all 5 sections, example has all fields filled

*Existing infrastructure covers all phase requirements via grep checks.*

---

## Manual-Only Verifications

| Behavior | Requirement | Why Manual | Test Instructions |
|----------|-------------|------------|-------------------|
| Spec format is understandable | SPEC-01 | Requires human reading | Read spec-template.md, verify a developer could fill it in |
| Example spec covers realistic feature | SPEC-01 | Requires domain judgment | Read example spec, verify it captures a full round-trip flow |

---

## Validation Sign-Off

- [ ] All tasks have automated verify or Wave 0 dependencies
- [ ] Sampling continuity: no 3 consecutive tasks without automated verify
- [ ] Wave 0 covers all MISSING references
- [ ] No watch-mode flags
- [ ] Feedback latency < 2s
- [ ] `nyquist_compliant: true` set in frontmatter

**Approval:** pending
