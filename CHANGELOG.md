# Changelog

Format follows [Keep a Changelog](https://keepachangelog.com/en/1.1.0/). Versioning follows the scheme in `README.md` → Versioning: MINOR bumps add a category or a field, MAJOR bumps change the shape of an existing category. Each entry here corresponds to a git tag (`vMAJOR.MINOR.0`) and a GitHub Release — see `CONTRIBUTING.md` → Releases for the process.

## [Unreleased]

### Added

- `source_columns` field on `compute`, `aggregate`, `cast`, and `window` entries — the column names referenced in that entry's expression, as a lightweight structural lineage signal (not a lineage graph; see `docs/grammar.md` → Compute for scope).

## [0.1.0] - 2026-08-17

Initial finalized release of the v0.1 spec.

### Added

- Full category table: `join`, `filter`, `set_op`, `rename`, `compute`, `aggregate`, `window`, `cast`, `conditional`, `subquery_cte`.
- Finalized JSON Schema (`schema/gdto-v0.1.schema.json`) — `aggregate`, `window`, `cast`, `conditional`, and `subquery_cte` now have real shapes (previously unshaped placeholders).
- `docs/grammar.md` — full grammar reference with worked examples and edge cases for every category.
- `docs/decisions.md` — architectural decisions: (0001) GDTO normalizes across dialect/engine surface syntax at the schema level, not per-dialect; (0002) `compute` and the structural-signal categories (`conditional`/`cast`/`aggregate`/`window`) are not mutually exclusive.
