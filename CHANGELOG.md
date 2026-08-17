# Changelog

Format follows [Keep a Changelog](https://keepachangelog.com/en/1.1.0/). Versioning follows the scheme in `README.md` → Versioning: MINOR bumps add a category or a field, MAJOR bumps change the shape of an existing category. Each entry here corresponds to a git tag (`vMAJOR.MINOR.0`) and a GitHub Release — see `CONTRIBUTING.md` → Releases for the process.

## [Unreleased]

### Added

- `wildcard_select` category for `SELECT *` / `t.*`, optionally with a dialect `EXCEPT(...)` clause — previously invisible to GDTO entirely. See issue #3.
- `source_columns` field on `compute`, `aggregate`, `cast`, and `window` entries — the column names referenced in that entry's expression, as a lightweight structural lineage signal (not a lineage graph; see `docs/grammar.md` → Compute for scope).
- `ai_function`, `udf`, and `column_hash` categories for AI/ML function calls, user-defined function calls, and hashing function calls. `ai_function`/`udf` introduce a new, function-name-based detection basis (no dedicated `sqlglot` AST node exists for either) — see `docs/decisions.md` (0003) for the reasoning and consequences. See issue #8.

## [0.1.0] - 2026-08-17

Initial finalized release of the v0.1 spec.

### Added

- Full category table: `join`, `filter`, `set_op`, `rename`, `compute`, `aggregate`, `window`, `cast`, `conditional`, `subquery_cte`.
- Finalized JSON Schema (`schema/gdto-v0.1.schema.json`) — `aggregate`, `window`, `cast`, `conditional`, and `subquery_cte` now have real shapes (previously unshaped placeholders).
- `docs/grammar.md` — full grammar reference with worked examples and edge cases for every category.
- `docs/decisions.md` — architectural decisions: (0001) GDTO normalizes across dialect/engine surface syntax at the schema level, not per-dialect; (0002) `compute` and the structural-signal categories (`conditional`/`cast`/`aggregate`/`window`) are not mutually exclusive.
