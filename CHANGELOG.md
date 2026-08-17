# Changelog

Format follows [Keep a Changelog](https://keepachangelog.com/en/1.1.0/). Versioning follows the scheme in `README.md` → Versioning: MINOR bumps add a category or a field, MAJOR bumps change the shape of an existing category. Each entry here corresponds to a git tag (`vMAJOR.MINOR.0`) and a GitHub Release — see `CONTRIBUTING.md` → Releases for the process.

## [Unreleased]

### Changed

- Restyled "GDT" to lowercase "gdt" in prose throughout docs and the schema's `title`/`description` fields — a stylistic change only. Identifiers (`gdt_version`, `schema/gdt-v0.1.schema.json`, the `decode-data/gdt` repo name/`$id`) were already lowercase. Historical CHANGELOG entries below are left as originally published.

## [0.2.0] - 2026-08-17

### Added

- `wildcard_select` category for `SELECT *` / `t.*`, optionally with a dialect `EXCEPT(...)` clause — previously invisible to GDT entirely. See issue #3.
- `source_columns` field on `compute`, `aggregate`, `cast`, and `window` entries — the column names referenced in that entry's expression, as a lightweight structural lineage signal (not a lineage graph; see `docs/grammar.md` → Compute for scope).
- `json_parse`, `json_extract`, and `unnest` categories for JSON/semi-structured operations (`PARSE_JSON`, path extraction via `->`/`->>`/`:`/`JSON_VALUE`/`JSON_QUERY`, and array unnesting via `UNNEST`/`LATERAL FLATTEN`/`LATERAL VIEW EXPLODE`). See issue #6.
- `ai_function`, `udf`, and `column_hash` categories for AI/ML function calls, user-defined function calls, and hashing function calls. `ai_function`/`udf` introduce a new, function-name-based detection basis (no dedicated `sqlglot` AST node exists for either) — see `docs/decisions.md` (0003) for the reasoning and consequences. See issue #8.

### Changed

- Renamed the spec from GDTO to GDT throughout — docs, the schema itself (output field `gdto_version` → `gdt_version`), and the schema filename (`gdto-v0.1.schema.json` → `gdt-v0.1.schema.json`). The GitHub repo was also renamed `decode-data/gdto` → `decode-data/gdt`.
- Dropped "Operations" from the README title (`Grammar of Data Transformation Operations` → `Grammar of Data Transformation`).

## [0.1.0] - 2026-08-17

Initial finalized release of the v0.1 spec.

### Added

- Full category table: `join`, `filter`, `set_op`, `rename`, `compute`, `aggregate`, `window`, `cast`, `conditional`, `subquery_cte`.
- Finalized JSON Schema (`schema/gdt-v0.1.schema.json`) — `aggregate`, `window`, `cast`, `conditional`, and `subquery_cte` now have real shapes (previously unshaped placeholders).
- `docs/grammar.md` — full grammar reference with worked examples and edge cases for every category.
- `docs/decisions.md` — architectural decisions: (0001) GDT normalizes across dialect/engine surface syntax at the schema level, not per-dialect; (0002) `compute` and the structural-signal categories (`conditional`/`cast`/`aggregate`/`window`) are not mutually exclusive.
