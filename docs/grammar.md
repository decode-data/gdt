# GDT v0.2 — Grammar

The full grammar reference: what each category means, its `sqlglot` mapping, at least one worked example with its expected GDT output, and known edge cases. `docs/categories.md` is the compact table version of this for quick LLM priming; this doc carries the discussion. The authoritative shape for every field below is `schema/gdt-v0.1.schema.json` — if this doc and the schema ever disagree, the schema wins and this doc has a bug.

Examples use generic ANSI SQL as the default. `join`, `filter`, `set_op`, `rename`, `compute`, `aggregate`, `subquery_cte` don't vary across dialects in ways that affect GDT output, so one example suffices for each. `conditional`, `cast`, and `wildcard_select` do have real dialect-specific spellings (Snowflake `IFF`, BigQuery/DuckDB `IF()`, `TRY_CAST`/`SAFE_CAST`, BigQuery/Snowflake/DuckDB `SELECT * EXCEPT(...)`) that are exactly what the normalization decision in `docs/decisions.md` (0001) is about — those sections show the dialect variants explicitly, to make the "same GDT output regardless of spelling" claim concrete rather than asserted.

Every category is independent — see `docs/decisions.md` (0002) for why `compute` and the structural-signal categories (`conditional`, `cast`, `aggregate`, `window`) can and do overlap on the same query.

---

## `join`

**Meaning:** a row-level join between two relations.

**`sqlglot` mapping:** `exp.Join`. Subtype comes from `.kind` (inner/cross) and `.side` (left/right/full).

**Example:**

```sql
SELECT o.id, c.name
FROM orders o
INNER JOIN customers c ON o.customer_id = c.id
```

```json
{
  "join": [
    { "kind": "inner", "tables": ["orders", "customers"], "keys": ["customer_id"] }
  ]
}
```

**Edge cases:**

- A `CROSS JOIN` has no `ON` clause — `keys` is empty or omitted, `kind` is `"cross"`.
- A multi-condition join (`ON a.x = b.x AND a.y = b.y`) lists every equality key in `keys`; a non-equality join condition (`ON a.x > b.x`) isn't representable as a key — falls back to being visible only via the query's overall structure, not encoded in `keys`. (Not solved in v0.1: no `condition_summary` fallback field exists yet for non-equi-joins. Candidate for a future MINOR bump if this turns out to matter in practice.)
- A self-join (`orders o1 JOIN orders o2 ON ...`) lists `"orders"` twice in `tables` — the two occurrences are only distinguished by their aliases in the source SQL, which GDT doesn't currently carry. Consumers needing to distinguish them must re-parse.

---

## `filter`

**Meaning:** row-level filtering — a `WHERE` clause.

**`sqlglot` mapping:** `exp.Where`.

**Example:**

```sql
SELECT * FROM orders WHERE status = 'completed'
```

```json
{
  "filter": [
    { "summary": "status = 'completed'" }
  ]
}
```

**Edge cases:**

- `HAVING` (`exp.Having`) is not `filter` — it post-filters aggregated rows, not source rows. v0.1 has no dedicated category for it; a `HAVING` clause is currently untagged. Worth a MINOR bump (new category `having_filter`, or extending `filter` with a `stage: "having"` field) once a real query surfaces the gap.
- A `WHERE` with multiple `AND`-ed conditions is one `filter` entry with the whole conjunction in `summary`, not split into per-condition entries — GDT tags "there is row-level filtering here and this is what it says," not a full boolean-expression decomposition.

---

## `set_op`

**Meaning:** a set operation combining two query results.

**`sqlglot` mapping:** `exp.Union` / `exp.Except` / `exp.Intersect`. Subtype: union/union_all/except/intersect.

**Example:**

```sql
SELECT id FROM active_customers
UNION ALL
SELECT id FROM archived_customers
```

```json
{
  "set_op": [
    { "kind": "union_all" }
  ]
}
```

**Edge cases:**

- A chain of three or more set ops (`A UNION B UNION ALL C`) produces one `set_op` entry per pairwise combination, in AST order — two entries for a three-way chain, not one.

---

## `rename`

**Meaning:** a pure passthrough of a source column under a new name — no transformation.

**`sqlglot` mapping:** `exp.Alias` wrapping a bare `exp.Column`.

**Example:**

```sql
SELECT cust_id AS customer_id FROM raw_customers
```

```json
{
  "rename": [
    { "source": "cust_id", "output": "customer_id" }
  ]
}
```

**Edge cases:**

- `SELECT t.cust_id AS customer_id` (table-qualified column) — `source` is the column name only (`"cust_id"`), not the qualified form (`"t.cust_id"`). The table qualifier is dropped; if that distinction matters to a consumer, it isn't currently preserved.
- `SELECT cust_id FROM ...` with no `AS` at all isn't `rename` — there's no `Alias` node, so it isn't tagged as anything. Only explicit renames are tagged; passthrough-with-original-name is invisible to GDT by design (nothing changed, nothing to tag).

---

## `compute`

**Meaning:** a derived output column — the immediate child of the `Alias` is anything other than a bare `Column`.

**`sqlglot` mapping:** `exp.Alias` wrapping anything else (`exp.Add`, `exp.Case`, `exp.Func`, etc.).

**Example:**

```sql
SELECT amount * (1 + tax_rate) AS total_with_tax FROM orders
```

```json
{
  "compute": [
    { "output": "total_with_tax", "expression_summary": "amount * (1 + tax_rate)", "source_columns": ["amount", "tax_rate"] }
  ]
}
```

**Edge case — resolved: a `CASE` (or cast/aggregate/window) inside a `compute` expression.** See `docs/decisions.md` (0002) for the full reasoning; short version here:

```sql
SELECT CASE WHEN status = 'active' THEN 1 ELSE 0 END AS is_active FROM users
```

produces **both**:

```json
{
  "compute": [
    { "output": "is_active", "expression_summary": "CASE WHEN status = 'active' THEN 1 ELSE 0 END", "source_columns": ["status"] }
  ],
  "conditional": [
    {
      "output": "is_active",
      "branches": [
        { "condition": "status = 'active'", "result": "1" }
      ],
      "default": "0"
    }
  ]
}
```

`compute` is decided purely by the top-level `Alias` test (lineage: "is this column derived at all?"); `conditional`/`cast`/`aggregate`/`window` are populated by a full-tree walk regardless of what wraps them (structure: "does branching/casting/aggregating/windowing happen anywhere?"). The two are independent, so both fire here. This also applies when the nesting is deeper — `CASE WHEN x THEN 1 ELSE 0 END * price AS weighted_flag` still emits a `conditional` entry for the nested `CASE` even though the `compute` entry's `expression_summary` is the whole multiplication, not just the `CASE`; that `compute` entry's `source_columns` would be `["x", "price"]` — every column referenced anywhere in the full expression, not just the ones in the nested `CASE`.

`compute`, `aggregate`, `cast`, and `window` all carry a `source_columns` field (every `exp.Column` found in the relevant expression's subtree) — a lightweight, structural lineage signal. It's deliberately not a lineage graph: no cross-model resolution, no multi-hop tracing through CTEs/joins, just "which columns does this specific expression reference." Full column lineage stays out of scope (see README → "What this is not").

---

## `aggregate`

**Meaning:** an aggregate function call, optionally grouped.

**`sqlglot` mapping:** aggregate `exp.Func` subclasses (`exp.Sum`, `exp.Count`, `exp.Avg`, `exp.Min`, `exp.Max`, ...) for the per-call fields; `exp.Group` for `group_by_keys`.

**Example:**

```sql
SELECT customer_id, SUM(amount) AS total_spent, COUNT(DISTINCT order_id) AS order_count
FROM orders
GROUP BY customer_id
```

```json
{
  "aggregate": [
    {
      "function": "sum",
      "argument_summary": "amount",
      "output": "total_spent",
      "distinct": false,
      "group_by_keys": ["customer_id"],
      "source_columns": ["amount"]
    },
    {
      "function": "count",
      "argument_summary": "order_id",
      "output": "order_count",
      "distinct": true,
      "group_by_keys": ["customer_id"],
      "source_columns": ["order_id"]
    }
  ]
}
```

**Edge cases:**

- One entry per aggregate function call, **not** one entry per `GROUP BY` clause — `group_by_keys` is repeated verbatim on every aggregate belonging to the same query. A query with three aggregate calls and one `GROUP BY customer_id` produces three `aggregate` entries, each carrying `group_by_keys: ["customer_id"]`. This is a deliberate flat/denormalized choice matching the rest of the spec's per-occurrence style, not a bug.
- A bare scalar aggregate with no `GROUP BY` (`SELECT SUM(amount) FROM orders`) has an empty or absent `group_by_keys` and no `output` (nothing aliases it).
- `COUNT(*)` has no column argument — `source_columns` is empty or absent, `argument_summary` is `"*"`.
- `SELECT SUM(amount) AS total_spent ...` is aliased — this is simultaneously a `compute` occurrence (`Alias` wrapping a non-`Column`) by the same reasoning as the `CASE` edge case above. v0.1 tags it in **both** `aggregate` and `compute`, consistent with the "not mutually exclusive" decision in `docs/decisions.md` (0002).

---

## `window`

**Meaning:** a windowed (`OVER (...)`) function call.

**`sqlglot` mapping:** `exp.Window`. Frame details from `exp.WindowSpec`.

**Example:**

```sql
SELECT
  customer_id,
  order_date,
  SUM(amount) OVER (
    PARTITION BY customer_id
    ORDER BY order_date
    ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
  ) AS running_total
FROM orders
```

```json
{
  "window": [
    {
      "function": "sum",
      "output": "running_total",
      "partition_by": ["customer_id"],
      "order_by": ["order_date"],
      "frame": { "kind": "rows", "start": "UNBOUNDED PRECEDING", "end": "CURRENT ROW" },
      "source_columns": ["amount"]
    }
  ]
}
```

**Edge cases:**

- No explicit frame clause (`OVER (PARTITION BY x ORDER BY y)` with no `ROWS`/`RANGE`) — `frame` is omitted entirely, not defaulted to the SQL-standard implicit frame. GDT reports what's textually present, not what the engine would infer.
- `ROW_NUMBER() OVER (PARTITION BY x)` with no `ORDER BY` — `order_by` is empty or absent; this is valid SQL for some window functions and not an error.
- A window function used without an alias — `output` is omitted, same as unaliased `aggregate`.

---

## `cast`

**Meaning:** an explicit type coercion.

**`sqlglot` mapping:** `exp.Cast`. Scope is deliberately narrow — type coercion only, not date/number formatting functions like `TO_CHAR`, which have no GDT category in v0.1.

**Generic ANSI example:**

```sql
SELECT CAST(order_id AS VARCHAR) AS order_id_str FROM orders
```

```json
{
  "cast": [
    { "source_summary": "order_id", "target_type": "varchar", "output": "order_id_str", "safe": false, "source_columns": ["order_id"] }
  ]
}
```

**Dialect variants — same GDT output, different spelling (see `docs/decisions.md` 0001):**

Snowflake / BigQuery safe-cast:

```sql
-- Snowflake
SELECT TRY_CAST(raw_value AS NUMBER) AS parsed_value FROM staging
```

```sql
-- BigQuery
SELECT SAFE_CAST(raw_value AS NUMERIC) AS parsed_value FROM staging
```

Both normalize to:

```json
{
  "cast": [
    { "source_summary": "raw_value", "target_type": "numeric", "output": "parsed_value", "safe": true, "source_columns": ["raw_value"] }
  ]
}
```

`target_type` is normalized to the GDT tagger's own lowercase type vocabulary (`numeric`), not preserved as the dialect's literal spelling (`NUMBER` in Snowflake vs. `NUMERIC` in BigQuery) — this is the same schema-level-normalization principle applied to type names, not just to function shape.

**Edge cases:**

- An *implicit* cast (no `CAST`/`TRY_CAST` in the source SQL, but the engine would coerce types anyway) is not tagged — GDT only sees what `sqlglot` parses as an explicit `exp.Cast` node. Implicit coercion is invisible to a syntax-level tagger by construction.
- `CAST(x AS VARCHAR(50))` — parameterized types: `target_type` is `"varchar"`; the length parameter is currently dropped. If precision/scale/length ever matters to a consumer, that's a candidate future field, not present in v0.1.

---

## `conditional`

**Meaning:** branching logic — `CASE`/`WHEN` and its dialect-specific single-branch equivalents.

**`sqlglot` mapping:** `exp.Case`. `exp.Case.this` for the simple-CASE operand (absent in searched CASE), `exp.Case.ifs` for the WHEN/THEN pairs, `exp.Case.default` for the ELSE.

**Generic ANSI example (searched CASE):**

```sql
SELECT CASE WHEN status = 'active' THEN 1 ELSE 0 END AS is_active FROM users
```

```json
{
  "conditional": [
    {
      "output": "is_active",
      "branches": [
        { "condition": "status = 'active'", "result": "1" }
      ],
      "default": "0"
    }
  ]
}
```

**Simple-CASE variant** (operand compared against each branch, `operand_summary` present):

```sql
SELECT CASE region WHEN 'US' THEN 'domestic' WHEN 'CA' THEN 'domestic' ELSE 'international' END AS shipping_class
FROM orders
```

```json
{
  "conditional": [
    {
      "output": "shipping_class",
      "operand_summary": "region",
      "branches": [
        { "condition": "'US'", "result": "'domestic'" },
        { "condition": "'CA'", "result": "'domestic'" }
      ],
      "default": "'international'"
    }
  ]
}
```

**Dialect variants — same GDT output, different spelling (this is the concrete case for the decision in `docs/decisions.md` 0001):**

Snowflake `IFF`:

```sql
SELECT IFF(status = 'active', 1, 0) AS is_active FROM users
```

BigQuery / DuckDB `IF()`:

```sql
SELECT IF(status = 'active', 1, 0) AS is_active FROM users
```

Both normalize to the **same** shape as the searched-`CASE` example above:

```json
{
  "conditional": [
    {
      "output": "is_active",
      "branches": [
        { "condition": "status = 'active'", "result": "1" }
      ],
      "default": "0"
    }
  ]
}
```

A single-branch `IFF`/`IF()` maps onto `conditional`'s `branches` array as a one-element array (rather than getting its own category or a `branches`-of-length-1-only special case) — a two-branch `IFF` and an equivalent two-`WHEN` `CASE` are indistinguishable in GDT output, which is the point.

**Edge cases:**

- `NULLIF(a, b)` and `COALESCE(a, b, c)` are *not* tagged `conditional` in v0.1, even though they're conditional in effect (`sqlglot` parses them as `exp.Nullif`/`exp.Coalesce`, not `exp.Case`, and they don't have a WHEN/THEN branch structure to map onto `branches`). This is a known gap, not an oversight — flagged here rather than silently mapped onto a shape that doesn't fit.
- See `compute`'s section above and `docs/decisions.md` (0002) for the resolved question of whether a `CASE` nested inside a larger computed expression also gets its own top-level `conditional` entry (yes).

---

## `subquery_cte`

**Meaning:** structural complexity — a CTE or an inline derived table. Not itself a transformation; a signal about query shape, orthogonal to every other category.

**`sqlglot` mapping:** `exp.With` (the `WITH` clause container) / `exp.CTE` (one entry) / `exp.Subquery` (an inline derived table).

**Example:**

```sql
WITH recent_orders AS (
  SELECT * FROM orders WHERE order_date > '2026-01-01'
)
SELECT customer_id, COUNT(*) AS order_count
FROM recent_orders
GROUP BY customer_id
```

```json
{
  "subquery_cte": [
    { "kind": "cte", "alias": "recent_orders", "location": "with" }
  ]
}
```

**Inline derived table:**

```sql
SELECT customer_id, order_count
FROM (SELECT customer_id, COUNT(*) AS order_count FROM orders GROUP BY customer_id) t
```

```json
{
  "subquery_cte": [
    { "kind": "subquery", "alias": "t", "location": "from" }
  ]
}
```

**Edge cases:**

- A `WITH` clause with multiple CTEs produces one `subquery_cte` entry per CTE, in declaration order — not one entry for the whole `WITH` block.
- A correlated subquery in a `WHERE` clause (`WHERE EXISTS (SELECT 1 FROM ... WHERE inner.x = outer.x)`) is `kind: "subquery"`, `location: "where"` — correlation itself (the reference to the outer query) isn't separately flagged; only presence and location are.
- A recursive CTE (`WITH RECURSIVE ...`) is tagged the same as a non-recursive one (`kind: "cte"`) — v0.1 has no `recursive` boolean field. Candidate for a future MINOR bump.

---

## `wildcard_select`

**Meaning:** a `SELECT *` (or `t.*`) wildcard — selecting all columns rather than explicitly enumerating them. Exists so a consumer can build a rule like "forbid `SELECT *`" or "only allow `SELECT * EXCEPT(...)` for known-safe exclusions"; without it, `SELECT *` was invisible to GDT (it's an `exp.Star` node, not an `exp.Alias`, so it doesn't match `rename` or `compute`).

**`sqlglot` mapping:** `exp.Star` — appears bare (`SELECT *`) or as the `this` of a qualified `exp.Column` (`t.*`). Dialect-specific `EXCEPT(...)` support is exposed as extra args on the same node in dialects that have it (BigQuery, Snowflake, DuckDB).

**Example — bare star:**

```sql
SELECT * FROM orders
```

```json
{
  "wildcard_select": [
    { "kind": "star" }
  ]
}
```

**Example — qualified star:**

```sql
SELECT o.*, c.name FROM orders o JOIN customers c ON o.customer_id = c.id
```

```json
{
  "wildcard_select": [
    { "kind": "qualified_star", "qualifier": "o" }
  ]
}
```

**Example — dialect `EXCEPT(...)`, same normalized shape across dialects (per `docs/decisions.md`, 0001):**

```sql
-- BigQuery / Snowflake / DuckDB
SELECT * EXCEPT (internal_id, updated_at) FROM orders
```

```json
{
  "wildcard_select": [
    { "kind": "star", "except_columns": ["internal_id", "updated_at"] }
  ]
}
```

**Edge cases:**

- `REPLACE(...)` (BigQuery-style `SELECT * REPLACE (upper(name) AS name)`) is explicitly out of scope for v1 — no field captures it. Known gap, not an oversight; a plain `wildcard_select` entry is still emitted for the `*` itself, just without the replacement detail. Candidate for a future MINOR bump (a `replace_columns` field) if this turns out to matter in practice.
- `COUNT(*)` does **not** produce a `wildcard_select` entry — that `*` is a function-call argument (`exp.Star` inside `exp.Count`), not a `SELECT *` in the projection list. Only `exp.Star` nodes appearing directly in the SELECT list are tagged here.
- A query with both a wildcard and explicit columns (`SELECT *, computed_col AS x FROM t`) produces a `wildcard_select` entry for the `*` and, independently, whatever category the other selected expressions warrant (`compute` for `computed_col AS x` above) — the categories aren't exclusive of each other, same as everywhere else in GDT.

---

## `json_parse`

**Meaning:** parsing a string or variant value into a JSON-typed value.

**`sqlglot` mapping:** `exp.ParseJSON`.

**Example:**

```sql
-- Snowflake / BigQuery
SELECT PARSE_JSON(raw_payload) AS payload FROM events
```

```json
{
  "json_parse": [
    { "source_summary": "raw_payload", "output": "payload", "safe": false, "source_columns": ["raw_payload"] }
  ]
}
```

**Safe variant:**

```sql
SELECT TRY_PARSE_JSON(raw_payload) AS payload FROM events
```

```json
{
  "json_parse": [
    { "source_summary": "raw_payload", "output": "payload", "safe": true, "source_columns": ["raw_payload"] }
  ]
}
```

**Edge cases:**

- `json_parse` is distinct from `cast` even though both are type-coercion-flavored — `exp.ParseJSON` is its own node, not `exp.Cast`, and there's no requirement that a consumer treat them the same way (a `CAST(x AS VARIANT)` and a `PARSE_JSON(x)` aren't always equivalent at the engine level, e.g. `CAST` on an already-valid-JSON string vs. `PARSE_JSON` doing actual parsing/validation).
- A `PARSE_JSON(...)` call that isn't aliased has no `output`, same convention as unaliased `aggregate`/`cast`/`window` entries.

---

## `json_extract`

**Meaning:** extracting a value from a JSON/semi-structured object by path.

**`sqlglot` mapping:** `exp.JSONExtract` (returns a nested JSON value) / `exp.JSONExtractScalar` (returns a scalar). Dialect-specific spellings normalize onto this same shape (see `docs/decisions.md`, 0001) — this is the concrete case for that decision, alongside `conditional`/`cast`.

**Generic example (scalar extraction):**

```sql
SELECT payload ->> 'email' AS user_email FROM events
```

```json
{
  "json_extract": [
    { "source_summary": "payload", "path": "$.email", "scalar": true, "output": "user_email", "source_columns": ["payload"] }
  ]
}
```

**Dialect variants — same normalized shape:**

```sql
-- BigQuery
SELECT JSON_VALUE(payload, '$.email') AS user_email FROM events
```

```sql
-- Snowflake (colon path operator)
SELECT payload:email::string AS user_email FROM events
```

Both normalize to the same `json_extract` shape as the Postgres example above (the Snowflake variant also produces a `cast` entry for the trailing `::string`, since that's an independent `exp.Cast` node — see `docs/decisions.md` 0002 on categories not being mutually exclusive).

**Non-scalar extraction** (`scalar: false`):

```sql
SELECT payload -> 'address' AS address_json FROM events
```

```json
{
  "json_extract": [
    { "source_summary": "payload", "path": "$.address", "scalar": false, "output": "address_json", "source_columns": ["payload"] }
  ]
}
```

**Edge cases:**

- Chained path access (`payload -> 'user' -> 'address' ->> 'city'`) is represented as **one** `json_extract` entry with the full merged path (`path: "$.user.address.city"`, `scalar: true`), not one entry per `->`/`->>` hop — `path` is the fully resolved path to the final extraction, matching how a single `JSON_VALUE(payload, '$.user.address.city')` call would tag.
- `source_summary` is always the innermost source expression (`payload` above), even for chained access — not the intermediate sub-extraction.

---

## `unnest`

**Meaning:** unnesting an array or semi-structured value into rows.

**`sqlglot` mapping:** `exp.Unnest`. **Confidence note:** this category is grounded with confidence in `exp.Unnest` itself (BigQuery/DuckDB/Postgres `UNNEST(...)`). Dialect-specific table functions that do the same logical thing — Snowflake `LATERAL FLATTEN(input => ...)`, Spark/Hive `LATERAL VIEW EXPLODE(...)` — are *expected* to normalize onto this same shape per the dialect-normalization decision (`docs/decisions.md`, 0001), but the exact `sqlglot` AST node those parse to (and whether `sqlglot` already unifies them with `exp.Unnest` or leaves them as separate function-call nodes) needs confirming against real parses when this category is implemented. Flagged honestly rather than asserted — don't treat the Snowflake/Spark examples below as verified `sqlglot` behavior the way the rest of this document's examples are.

**Example:**

```sql
-- BigQuery / DuckDB / Postgres
SELECT o.order_id, tag
FROM orders o, UNNEST(o.tags) AS tag
```

```json
{
  "unnest": [
    { "source_summary": "o.tags", "alias": "tag", "ordinality": false, "location": "from", "source_columns": ["tags"] }
  ]
}
```

**With ordinality:**

```sql
SELECT o.order_id, tag, idx
FROM orders o, UNNEST(o.tags) WITH ORDINALITY AS t(tag, idx)
```

```json
{
  "unnest": [
    { "source_summary": "o.tags", "alias": "tag", "ordinality": true, "location": "from", "source_columns": ["tags"] }
  ]
}
```

**Expected (unverified) dialect equivalents:**

```sql
-- Snowflake
SELECT o.order_id, f.value AS tag
FROM orders o, LATERAL FLATTEN(input => o.tags) f
```

```sql
-- Spark / Hive
SELECT o.order_id, tag
FROM orders o LATERAL VIEW EXPLODE(o.tags) t AS tag
```

Both are expected to normalize to the same `unnest` shape as the BigQuery example — pending the confidence-note verification above.

**Edge cases:**

- `UNNEST` used directly in the `SELECT` list rather than a table reference (some dialects allow this) isn't distinguished from the `FROM`/`JOIN` form by a different `kind` — v0.1 only tracks `location` (`from`/`join`), not this finer distinction. Candidate gap if it turns out to matter.
- Unnesting a literal array (`UNNEST([1, 2, 3])`) has an empty or absent `source_columns`, same convention as `COUNT(*)` and literal-only `compute` expressions.

---

## `ai_function`

**Meaning:** a call to a recognized AI/ML SQL function.

**Grounding — read this before the example.** Every category above this one is grounded in a specific `sqlglot` AST node class. `ai_function` isn't — `sqlglot` has no dedicated node for "this is an AI vendor function," so it parses as an ordinary function call (a recognized dialect `Func` subclass, or `exp.Anonymous` if unrecognized). Detection here is **function-name matching against a maintained allowlist**, not AST-node matching. See `docs/decisions.md` (0003) for the full reasoning and the consequences (an implementation must own and maintain that allowlist; expect false negatives for functions not yet in it).

**Example:**

```sql
-- Snowflake Cortex
SELECT ticket_id, AI_COMPLETE('llama3-8b', prompt_text) AS response
FROM support_tickets
```

```json
{
  "ai_function": [
    { "function": "ai_complete", "output": "response", "argument_summary": "'llama3-8b', prompt_text", "source_columns": ["prompt_text"] }
  ]
}
```

**Dialect examples (illustrative — allowlist entries an implementation would need, not verified `sqlglot` parse output):**

```sql
-- BigQuery
SELECT ML.GENERATE_TEXT(MODEL my_model, prompt_text) AS response FROM support_tickets
```

```sql
-- Databricks
SELECT ai_query('my-endpoint', prompt_text) AS response FROM support_tickets
```

**Edge cases:**

- An unrecognized AI-shaped function call (not yet in the tagger's allowlist) is not tagged `ai_function` — it silently falls through to `udf` instead (see below), since the tagger has no way to distinguish "unknown AI function" from "unknown UDF" by name alone. This is the expected failure mode of name-based detection, not a bug to work around per-query.
- GDT does not enumerate specific AI/ML function names anywhere in the schema or this doc, deliberately — see `docs/decisions.md` (0003) for why.

---

## `udf`

**Meaning:** a call to a user-defined (non-builtin) function.

**Grounding:** same basis as `ai_function` — no dedicated `sqlglot` node. In practice this is usually `exp.Anonymous` (`sqlglot`'s fallback for a function name it doesn't recognize as a builtin for the query's dialect). A tagger should check the `ai_function` allowlist first; only tag `udf` if that doesn't match. See `docs/decisions.md` (0003).

**Example:**

```sql
SELECT customer_id, my_schema.normalize_phone(phone_number) AS phone
FROM customers
```

```json
{
  "udf": [
    { "function": "my_schema.normalize_phone", "output": "phone", "argument_summary": "phone_number", "source_columns": ["phone_number"] }
  ]
}
```

**Edge cases:**

- `ai_function` and `udf` are **mutually exclusive** by the precedence rule above — this is a deliberate exception to the "categories overlap" pattern from `docs/decisions.md` (0002), because here the two categories are competing classifications of the same node, not orthogonal signals about different aspects of it.
- A genuine dialect builtin that `sqlglot` doesn't yet recognize for that dialect (parses as `exp.Anonymous` even though it's not actually user-defined) is indistinguishable from a real UDF at this layer — a known false-positive source, not solvable without a per-dialect builtin registry GDT doesn't maintain.

---

## `column_hash`

**Meaning:** a call to a hashing function.

**Grounding:** mostly a normal, AST-node-grounded category — `exp.MD5`, `exp.SHA`, `exp.SHA2` are real dedicated `sqlglot` nodes. Dialect-specific hash functions `sqlglot` has no dedicated class for (Snowflake's generic `HASH()`, BigQuery's `FARM_FINGERPRINT`) fall back to function-name matching, the same mechanism `ai_function`/`udf` use as their *primary* basis — here it's only a fallback for the minority of cases. Don't read `column_hash`'s overall confidence as equivalent to `ai_function`/`udf`'s; it's much closer to `conditional`/`cast`'s dialect-normalization story.

**Example:**

```sql
SELECT customer_id, SHA2(email, 256) AS email_hash FROM customers
```

```json
{
  "column_hash": [
    { "function": "sha2", "argument_summary": "email", "output": "email_hash", "algorithm_bits": 256, "source_columns": ["email"] }
  ]
}
```

**Dialect fallback example:**

```sql
-- Snowflake
SELECT customer_id, HASH(email) AS email_hash FROM customers
```

```json
{
  "column_hash": [
    { "function": "hash", "argument_summary": "email", "output": "email_hash", "source_columns": ["email"] }
  ]
}
```

**Edge cases:**

- Hashing multiple columns together (`SHA2(CONCAT(first_name, last_name), 256)`) still produces one `column_hash` entry — `argument_summary` captures the whole argument expression as a string, and `source_columns` lists every column referenced within it (`["first_name", "last_name"]`), same convention as `compute`.
- `column_hash` doesn't imply anything about *why* a column is hashed (PII masking vs. a dedup surrogate key vs. a partition key) — that judgment is a consumer's rule to build on top of this tag, not something GDT infers.
