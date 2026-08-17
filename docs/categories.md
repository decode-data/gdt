# GDTO v0.1 — Category Reference

Compact reference, kept separate from the full README so an LLM (or a human) can load just this table into context without the surrounding discussion.

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

GDTO version: 0.1.

See `docs/grammar.md` for the full reference (worked examples, edge cases) and `docs/decisions.md` for architectural decisions (dialect normalization, category overlap).
