# gdt — Grammar of Data Transformation

**Status:** v0.2 — expect revision as real-world SQL patterns surface from downstream implementations.
**License:** Apache 2.0 — a spec meant to be reimplemented against parsers other than `sqlglot` someday; the patent grant matters more here than for a single library.

## What this is

A versioned taxonomy of SQL transformation operations, plus a JSON Schema for the structured summary a parser emits when it tags a query against this grammar. It is **not** a parser, and it is **not** `sqlglot`-specific by design — it's grounded in `sqlglot`'s expression classes because that's the first (and for now, only) implementation, but the vocabulary is meant to outlive that choice. Anyone building a SQL-comprehension tool should be able to implement gdt tagging without adopting `sqlglot`.

## What this is not

- Not a linter, not a rules engine, not a lineage tool. Those are consumers of gdt output, not part of the spec.
- Not tied to dbt, or to any particular transformation tool or data stack — consumers may target dbt, plain SQL pipelines, a pandas/Spark job, or something else entirely.
- Not a formatter/generator spec — style/formatting rules are a separate concern owned by whatever consumes gdt output.

## v0.2 categories

Grounded in `sqlglot`'s `expressions` module (`core.py` for base nodes, `query.py` for `Select`/`Union`/joins, `functions.py` for the `Func` base and aggregates):

| Category | `sqlglot` mapping | Notes |
|---|---|---|
| `join` | `exp.Join` | Subtype via `.kind`/`.side`: inner/left/right/full/cross |
| `filter` | `exp.Where` | Row-level filtering |
| `set_op` | `exp.Union` / `exp.Except` / `exp.Intersect` | Subtype: union/union_all/except/intersect |
| `rename` | `exp.Alias` wrapping a bare `exp.Column` | Pure passthrough — no transformation |
| `compute` | `exp.Alias` wrapping anything else | `exp.Add`, `exp.Case`, `exp.Func`, etc. — a derived column |
| `aggregate` | `exp.Group` + aggregate `exp.Func` subclasses | `Sum`, `Count`, `Avg`, `Min`, `Max`... One entry per aggregate call, not per `GROUP BY` clause — group keys repeat on each call |
| `window` | `exp.Window` | OVER clause |
| `cast` | `exp.Cast` | Type coercion |
| `conditional` | `exp.Case` | CASE/WHEN logic — dialect equivalents (`IFF`, `IF()`) normalize to the same shape |
| `subquery_cte` | `exp.With` / `exp.CTE` / `exp.Subquery` | Structural complexity signal, not itself a transformation |
| `wildcard_select` | `exp.Star` | `SELECT *` or `t.*`, optionally with dialect `EXCEPT(...)`. `REPLACE(...)` out of scope |
| `json_parse` | `exp.ParseJSON` | String/variant parsed as JSON (`PARSE_JSON`) |
| `json_extract` | `exp.JSONExtract` / `exp.JSONExtractScalar` | Path extraction — `->`/`->>`, `:`, `JSON_VALUE`/`JSON_QUERY` normalize to the same shape |
| `unnest` | `exp.Unnest` | Array/semi-structured unnest. Dialect table functions (`LATERAL FLATTEN`, `LATERAL VIEW EXPLODE`) expected to normalize here |
| `ai_function` | *(name-based, no dedicated node — see `docs/decisions.md` 0003)* | Known AI/ML SQL function calls, matched against a maintained allowlist |
| `udf` | *(name-based, typically `exp.Anonymous` — see `docs/decisions.md` 0003)* | Fallback for unrecognized/non-builtin function calls not matched as `ai_function` |
| `column_hash` | `exp.MD5` / `exp.SHA` / `exp.SHA2`, name-matching fallback for dialect extras | Hashing function calls (`MD5`, `SHA256`, `HASH`, `FARM_FINGERPRINT`, ...) |

See `docs/categories.md` for a compact, LLM-citation-friendly version of this table on its own.

The `rename`/`compute` split matters most in practice: "is the immediate child of this `Alias` a bare `Column`" is the whole test, and it's what backs a rule like "certain transformations may only rename, never compute."

## Output shape

See `schema/gdt-v0.1.schema.json` for the authoritative JSON Schema. Example:

```json
{
  "gdt_version": "0.2",
  "operations": {
    "join": [{ "kind": "inner", "tables": ["orders", "customers"], "keys": ["customer_id"] }],
    "rename": [{ "source": "cust_id", "output": "customer_id" }],
    "compute": [{ "output": "total_with_tax", "expression_summary": "amount * (1 + tax_rate)" }],
    "filter": [{ "summary": "status = 'completed'" }]
  }
}
```

## Versioning

Semantic versioning, independent of any implementation's release cadence. A MINOR bump adds a category or a field; a MAJOR bump changes the shape of an existing category. Implementations declare which gdt version they target.

## Relationship to madflow-sqlops

`madflow-sqlops` (separate repo, MIT license) is the reference implementation: a `sqlglot`-based Python package that walks a query's AST and emits output conforming to this spec. This repo has no dependency on that one. That one depends on this one's schema version.

## LLM-friendliness

gdt isn't invokable, so it doesn't need a Skill — but it should be citation-friendly for any LLM reasoning about gdt output. The JSON Schema is the priming artifact, not this README — keep it as a single, complete, well-commented file an LLM can load whole into context. `docs/categories.md` is the compact reference for quick priming; this README carries the fuller discussion; `docs/grammar.md` carries the full worked-example/edge-case reference; `docs/decisions.md` records the architectural decisions (dialect/engine normalization, category overlap) that affect implementations.

## Possible future addition: a gdt-native ruleset + verifier

Not built now. "Which gdt operations are allowed on which matched models" is a gdt-level concept, distinct from a consuming application's own rules layer (which may also cover PII/style/strategy — app-specific concerns that don't belong here). If this ever gets built, it's a minimal, standalone, gdt-only ruleset schema + CLI verifier living in this repo — deliberately not in `madflow-sqlops`, which stays rule-free by design.

## Status / open tasks

- [x] v0.1 category table drafted
- [x] Formal JSON Schema finalized (`schema/`)
- [x] Grammar doc with edge cases (`docs/grammar.md`) — the `Case`-in-`compute` question is resolved in `docs/decisions.md` (0002): both, since `compute` and the structural-signal categories are independent
- [x] CONTRIBUTING.md
- [x] Dialect/engine normalization decision (`docs/decisions.md`, 0001)
- [x] First tagged release (`v0.1.0`)
- [x] Second tagged release (`v0.2.0`) — `wildcard_select`, `source_columns`, `json_parse`/`json_extract`/`unnest`, `ai_function`/`udf`/`column_hash`, `docs/decisions.md` (0003), rename from GDTO to GDT — see `CHANGELOG.md` and `CONTRIBUTING.md` → Releases
- No code in this repo — spec-only until `madflow-sqlops` needs to reference a tagged version.
