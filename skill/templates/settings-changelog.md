# Settings Changelog

Append-only record of all settings and configuration changes. Each entry captures what changed and why, so future sessions never need to re-explain a settings decision.

## Usage

Copy to `.lifecycle/settings-changelog.md` on first settings change detection.

## Changelog

| Date | File | Change | Reason |
|------|------|--------|--------|
| 2026-01-01 12:00 | .env | Added DB_HOST=localhost | Initial database setup |

## Rules

1. Append only -- never edit or remove existing entries
2. One row per change, even if multiple changes happen in one session
3. Settings files include: .env*, *.config.*, package.json (scripts/dependencies), tsconfig.json, *.yaml/*.toml config files, framework config files
