# GDTO Decisions

Architectural decisions that affect how GDTO is implemented, not just what it says. ADR-style: one entry per decision, append-only — superseding a decision gets a new entry that references the old one, not an edit to it.

---

## 0001 — Normalization happens at the schema/category level, not per surface syntax

**Status:** Decided (v0.1).

**Question:** `sqlglot`'s AST includes dialect-specific node subclasses in places (e.g. Snowflake's `IFF(cond, a, b)` parses differently from a generic `CASE WHEN cond THEN a ELSE b END`, even though they're semantically identical). Does a GDTO tagger normalize these onto the same category/shape, or does GDTO preserve the dialect-specific detail and push normalization onto whatever consumes the tagged output?

This was raised as an explicit open question in `madflow-sqlops` (the reference implementation), which is blocked on it: its README lists `[ ] JSON Schema pinned from gdto (blocked on gdto finalizing v0.1 schema)` and states the call has to be made here, in GDTO, first — not decided unilaterally in the implementation.

**Decision:** GDTO normalizes at the schema/category level. A Snowflake `IFF(cond, a, b)` and an ANSI `CASE WHEN cond THEN a ELSE b END` both tag as `conditional`, with the same `branches` shape (see `docs/grammar.md` → Conditional). The schema has no `dialect` field and no per-dialect variant shapes anywhere. Mapping a dialect-specific AST node onto the right generic category is an implementation responsibility — `madflow-sqlops`'s tagging step, not something GDTO encodes.

**Reasoning:**

- The category vocabulary (`join`, `filter`, `rename`, `compute`, `aggregate`, `window`, `cast`, `conditional`, `subquery_cte`, `set_op`) is already semantic, not syntactic — it describes *what operation happened*, not *which keyword spelled it*. Preserving dialect detail in the schema would mean encoding syntax GDTO deliberately doesn't care about.
- The README already commits to this direction: GDTO is grounded in `sqlglot`'s expression classes for now but is explicitly "not `sqlglot`-specific by design," with a vocabulary "meant to outlive that choice." A schema that varies its shape by dialect (or by parser) would contradict that from day one.
- It's the cheaper failure mode. If an implementation gets a normalization wrong (tags a construct into the wrong category), that's a bug in one tagger, fixable in isolation. If the schema itself branched by dialect, every consumer (rules engines, lineage tools) would need dialect-aware logic to ask "does this query use a conditional?" — the exact complexity GDTO exists to hide.
- It matches what the reference implementation already expects of itself: `madflow-sqlops`'s own testing plan states "GDTO categories should be dialect-agnostic even though the SQL text isn't," with cross-dialect fixtures planned for exactly this (Snowflake vs. BigQuery vs. DuckDB variants of the same logical operation expected to produce identical tagged output).

**Consequence for `madflow-sqlops`:** unblocked — the schema is finalized (see `schema/gdto-v0.1.schema.json`), and the tagging step should map every dialect-specific single-branch conditional function (`IFF`, BigQuery/DuckDB `IF()`, etc.) onto `conditional`'s `branches` shape, every dialect-specific safe-cast variant (`TRY_CAST`, `SAFE_CAST`) onto `cast`'s `safe: true`, and so on — never introduce a dialect-keyed field to route around a category that doesn't fit.

**A future-proofing note, not a v0.1 commitment:** the same reasoning generalizes past SQL dialects to surface syntax/engine more broadly. The category vocabulary is relational-algebra-shaped, not SQL-shaped — it already maps cleanly onto DataFrame APIs (`pandas`/`polars`/`spark`): `.merge()` → `join`, `.query()`/boolean indexing → `filter`, `.assign()` → `compute`, `.groupby().agg()` → `aggregate`, `.rolling()`/window functions → `window`, `.astype()` → `cast`, `np.where()`/`case_when()` → `conditional`. Framing this decision as "normalize across surface syntax/engine" rather than narrowly "normalize across SQL dialects" means a future non-SQL tagger doesn't need a second normalization ADR — it inherits this one, and only needs a new mapping row per category in that tagger's own docs.

One category doesn't generalize cleanly: `subquery_cte` is SQL-native (`WITH` / derived tables). DataFrame APIs have no CTE equivalent — method chaining is structurally similar but isn't the same concept, and forcing a mapping would be strained. If a non-SQL tagger is ever built, `subquery_cte` is expected to stay empty or be explicitly out of scope for it, not stretched to fit. This is a documentation note only — v0.1 stays `sqlglot`/SQL-grounded, no DataFrame-specific fields or categories are added now.

---

## 0002 — `compute` and the structural-signal categories are not mutually exclusive

**Status:** Decided (v0.1).

**Question:** When a `CASE` expression (or a nested aggregate, cast, or window function) appears inside the expression an `Alias` wraps, is the occurrence tagged `compute`, `conditional` (or `aggregate`/`cast`/`window`), or both? E.g. `SELECT CASE WHEN status = 'active' THEN 1 ELSE 0 END AS is_active FROM users`.

**Decision:** Both. `compute` and the four structural-signal categories (`conditional`, `cast`, `aggregate`, `window`) answer different questions and are populated independently:

- `compute` is a column-lineage signal: is this output column a pure passthrough of a source column, or is it derived? It's decided purely by the existing top-level `Alias` test (bare `Column` child → `rename`; anything else → `compute`) — nothing about *what kind* of derivation it is.
- `conditional` / `cast` / `aggregate` / `window` are structural-complexity signals: does this query use branching logic / type coercion / aggregation / windowing *anywhere*? They're populated by walking the full AST and emitting one entry per matching node, regardless of nesting depth or whether that node also sits inside a column already tagged `compute`.

So the example above produces **both** a `compute` entry (`output: "is_active"`, `expression_summary: "CASE WHEN status = 'active' THEN 1 ELSE 0 END"`) **and** a `conditional` entry for the same `CASE`. See `docs/grammar.md` for the full worked example and more edge cases (a `CASE` nested inside arithmetic, a `CAST` inside a `CASE` branch, etc.).

**Reasoning:** this mirrors how the spec already treats `subquery_cte` — the category table describes it as "a structural complexity signal, not itself a transformation," explicitly orthogonal to whatever else is happening in the query. Extending that same orthogonality to `conditional`/`cast`/`aggregate`/`window` is consistent, not a new pattern. It's also the more useful answer for actual consumers: lineage tooling cares whether `is_active` is derived (that's `compute`); a rule like "flag any `CASE` used in staging models" needs to see the `CASE` even when it's three levels deep inside an arithmetic expression that `compute` only ever describes as an opaque `expression_summary` string. Collapsing the two into a single tag would silently lose whichever signal the consumer needed.
