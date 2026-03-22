---
phase: 9
slug: outer-loop
status: draft
nyquist_compliant: true
wave_0_complete: true
created: 2026-03-22
---

# Phase 9 — Validation Strategy

## Test Infrastructure

| Property | Value |
|----------|-------|
| **Framework** | File structure + grep verification |
| **Quick run command** | `ls skill/references/deploy-stage.md skill/references/retrospect-stage.md 2>/dev/null && echo "OK"` |
| **Full suite command** | `for f in deploy deploy-test document retrospect promote; do test -f skill/references/${f}-stage.md && echo "${f}: OK" || echo "${f}: MISSING"; done` |
| **Estimated runtime** | ~1 second |

## Per-Task Verification Map

| Task ID | Plan | Wave | Requirement | Test Type | Status |
|---------|------|------|-------------|-----------|--------|
| 09-01-01 | 01 | 1 | PIPE-05,06 | file+grep | pending |
| 09-01-02 | 01 | 1 | PIPE-07,08,09 | file+grep | pending |
| 09-02-01 | 02 | 2 | PIPE-05~09 | grep (SKILL.md + role-matrix) | pending |

## Wave 0 Requirements

- [x] No test framework needed — skill reference files only

## Manual-Only Verifications

| Behavior | Requirement | Why Manual | Test Instructions |
|----------|-------------|------------|-------------------|
| Deploy checklist is project-type-appropriate | PIPE-05 | Requires project context | Run DEPLOY on a web project, verify web-specific checklist |

## Validation Sign-Off

- [x] All tasks have automated verify
- [x] Feedback latency < 2s
- [x] `nyquist_compliant: true` set in frontmatter

**Approval:** pending
