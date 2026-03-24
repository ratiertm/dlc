# Dev Lifecycle Changelog

## [2.0.0] - 2026-03-25

### Added
- **Configuration system** (Phase 11): lifecycle-config get/set/list with 3-layer resolution (env > config.yaml > defaults)
- **Mini-verify loops** (Phase 12): each spec step verified immediately in DO stage
- **Completeness scoring** (Phase 12): N/10 quality score at every stage completion
- **Safe upgrade** (Phase 13): automatic schema migration with rollback safety
- **config.yaml**: file-based settings with environment variable overrides
- **settings-changelog.md**: automatic change tracking with reasons
- **decisions.md**: lightweight decision log below ADR threshold

### Changed
- state.json version "1.0" -> "2.0" (new `current.completeness` field)
- Mode resolution now uses config layers (env > config.yaml > state.json)
- DO stage pipeline expanded with mini-verify sub-steps (2d.1/2d.2)

## [1.0.0] - 2026-03-22

### Added
- 9-stage pipeline (PLAN through PROMOTE)
- E2E spec format with 5-step chain
- Clickable HTML prototypes
- State management (state.json, manifest.json)
- Living State Document for session restoration
- 4 execution modes (hotfix, feature, release, milestone)
- Stage transition gates with artifact verification
- Memory & Decision Trail (settings-changelog, decisions, WHY+SEE comments)
- State reconcile for recovery from state.json loss
