# GDT v0.1 — Category Reference

Compact reference, kept separate from the full README so an LLM (or a human) can load just this table into context without the surrounding discussion.

| Category | `sqlglot` mapping | Notes |
|---|---|---|
| `join` | `exp.Join` | Subtype via `.kind`/`.side`: inner/left/right/full/cross |
| `filter` | `exp.Where` | Row-level filtering |
| `set_op` | `exp.Union` / `exp.Except` / `exp.Intersect` | Subtype: union/union_all/except/intersect |
| `rename` | `exp.Alias` wrapping a bare `exp.Column` | Pure passthrough — no transformation |
| `compute` | `exp.Alias` wrapping anything else | `exp.Add`, `exp.Case`, `exp.Func`, etc. — a derived column. Carries `source_columns` (referenced column names) |
| `aggregate` | `exp.Group` + aggregate `exp.Func` subclasses | `Sum`, `Count`, `Avg`, `Min`, `Max`... One entry per aggregate call, not per `GROUP BY` clause — group keys repeat on each call. Carries `source_columns` |
| `window` | `exp.Window` | OVER clause. Carries `source_columns` (the windowed function's own argument, not partition/order columns) |
| `cast` | `exp.Cast` | Type coercion. Carries `source_columns` |
| `conditional` | `exp.Case` | CASE/WHEN logic — dialect equivalents (`IFF`, `IF()`) normalize to the same shape |
| `subquery_cte` | `exp.With` / `exp.CTE` / `exp.Subquery` | Structural complexity signal, not itself a transformation |
| `wildcard_select` | `exp.Star` | `SELECT *` or `t.*`, optionally with dialect `EXCEPT(...)`. `REPLACE(...)` out of scope |
| `json_parse` | `exp.ParseJSON` | String/variant parsed as JSON (`PARSE_JSON`) |
| `json_extract` | `exp.JSONExtract` / `exp.JSONExtractScalar` | Path extraction — `->`/`->>`, `:`, `JSON_VALUE`/`JSON_QUERY` normalize to the same shape |
| `unnest` | `exp.Unnest` | Array/semi-structured unnest. Dialect table functions (`LATERAL FLATTEN`, `LATERAL VIEW EXPLODE`) expected to normalize here |
| `ai_function` | *(name-based, no dedicated node — see `docs/decisions.md` 0003)* | Known AI/ML SQL function calls, matched against a maintained allowlist |
| `udf` | *(name-based, typically `exp.Anonymous` — see `docs/decisions.md` 0003)* | Fallback for unrecognized/non-builtin function calls not matched as `ai_function` |
| `column_hash` | `exp.MD5` / `exp.SHA` / `exp.SHA2`, name-matching fallback for dialect extras | Hashing function calls (`MD5`, `SHA256`, `HASH`, `FARM_FINGERPRINT`, ...) |

GDT version: 0.1.

See `docs/grammar.md` for the full reference (worked examples, edge cases) and `docs/decisions.md` for architectural decisions (dialect normalization, category overlap).
