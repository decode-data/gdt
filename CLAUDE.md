# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this repo is

gdt (Grammar of Data Transformation) is a **spec-only repository** — no code, no build, no tests, no dependencies to install. It defines a versioned taxonomy of SQL transformation operations plus a JSON Schema for the structured output a "gdt tagger" emits when it classifies a SQL query against that taxonomy. There is nothing to run or compile; the work here is writing/editing Markdown and JSON Schema, and keeping them consistent with each other.

Formerly named GDTO ("Grammar of Data Transformation Operations") — renamed to GDT, then restyled lowercase to gdt (prose only — identifiers like `gdt_version`, the `decode-data/gdt` repo name, and the schema `$id` were already lowercase). If you see "GDTO" or "GDT" anywhere (old links, cached docs, memory from a prior session), it refers to this same repo under an old name/styling.

Status: v0.2, released (tag `v0.2.0`). Check `CHANGELOG.md` → `[Unreleased]` before assuming `v0.2.0` is what `main` currently describes.

## Repo structure

- `README.md` — full spec discussion: category table, output shape example, versioning policy, relationship to the reference implementation, LLM-friendliness rationale.
- `docs/categories.md` — the same category table as the README, stripped of surrounding discussion, kept as a standalone LLM-citation-friendly reference.
- `docs/grammar.md` — the full grammar reference: one section per category with meaning, `sqlglot` mapping, worked SQL example(s) with expected gdt JSON output, and known edge cases/gaps. This is where field-level detail and dialect-variant examples (Snowflake/BigQuery/DuckDB) live — `docs/categories.md` deliberately doesn't carry this.
- `docs/decisions.md` — ADR-style architectural decisions (append-only, numbered). Currently: (0001) gdt normalizes across dialect/engine surface syntax at the schema level rather than preserving per-dialect detail; (0002) `compute` and the structural-signal categories (`conditional`/`cast`/`aggregate`/`window`) are not mutually exclusive — they can and do overlap on the same node; (0003) `ai_function`/`udf` are detected by function-name matching against a maintained allowlist, not a dedicated `sqlglot` AST node — a real departure from how every other category is grounded, and the one place `ai_function`/`udf` are mutually exclusive with each other (allowlist checked first) rather than overlapping. Check here before re-litigating any of these.
- `schema/gdt-v0.1.schema.json` — the authoritative JSON Schema (2020-12) for a gdt tagged summary. This is the priming artifact for LLMs reasoning about gdt output — keep it complete and well-commented rather than pointing elsewhere.
- `CHANGELOG.md` — Keep a Changelog format; each entry corresponds to a git tag (`vMAJOR.MINOR.0`) and GitHub Release.
- `CONTRIBUTING.md` — process rules for spec changes, including the release process (git tags + GitHub Releases) — see below.
- `.github/pull_request_template.md` — PR template enforcing the version-impact and sync checklist from CONTRIBUTING.md.
- `.github/workflows/validate.yml` — CI on every PR: validates the schema is syntactically valid JSON Schema (2020-12, via `jsonschema`'s `Draft202012Validator`), and fails if `schema/gdt-v0.1.schema.json` changed without `docs/categories.md` changing (or vice versa) — this makes the sync rule below a real gate, not just an honor-system checklist item. If you touch the schema, expect this to fail unless `docs/categories.md` also changes in the same PR — that's it working correctly, not a bug (it caught exactly this while `source_columns` was being added).
- `CODEOWNERS` — currently a placeholder (`@REPLACE_ME`), not yet filled in.

## The core invariant: three files must stay in sync

`README.md`'s category table, `docs/categories.md`, and `schema/gdt-v0.1.schema.json` describe the same categories (currently seventeen: `join`, `filter`, `set_op`, `rename`, `compute`, `aggregate`, `window`, `cast`, `conditional`, `subquery_cte`, `wildcard_select`, `json_parse`, `json_extract`, `unnest`, `ai_function`, `udf`, `column_hash`) at different levels of detail. **Any change to the schema must be checked against `docs/categories.md`** — CI enforces this (see `.github/workflows/validate.yml` above), and CONTRIBUTING.md/the PR template both call it out too. `docs/grammar.md` should also be kept current but isn't CI-enforced — update it by hand alongside the schema.

## Design grounding

The category table is grounded in `sqlglot`'s `expressions` module (`core.py` for base nodes, `query.py` for `Select`/`Union`/joins, `functions.py` for `Func` and aggregates) — but the spec is explicitly **not** `sqlglot`-specific by design. It's meant to be implementable against other SQL parsers, and per `docs/decisions.md` (0001), the normalization principle is framed generally enough to extend past SQL entirely (DataFrame APIs like `pandas`/`polars`/`spark` map onto the same categories) — see that decision before assuming anything here is SQL-only forever. Don't add categories or fields that only make sense in `sqlglot` terms without noting the general concept they represent.

The `rename`/`compute` split is the highest-leverage distinction in the spec: whether the immediate child of an `Alias` is a bare `Column` (rename, pure passthrough) or anything else (compute, a derived column). This single test backs downstream rules like "certain transformations may only rename, never compute" — keep it precise when editing. Note that `compute` and the structural-signal categories aren't exclusive (`docs/decisions.md` 0002) — a `CASE` inside a computed column tags as both `compute` and `conditional`, don't "fix" that as if it were a bug.

Most categories with an expression-summary-style field also carry an optional `source_columns` field — the column names referenced in that entry's expression (`compute`, `aggregate`, `cast`, `window`, `json_parse`, `json_extract`, `unnest`, `ai_function`, `udf`, `column_hash`). It's a deliberately lightweight structural-lineage signal (not a lineage graph, no cross-model/multi-hop resolution) — full column lineage stays out of scope per README → "What this is not." Don't expand `source_columns` into graph-building; that's a different tool's job.

`ai_function` and `udf` are the one place two categories are mutually exclusive by convention rather than overlapping — see `docs/decisions.md` (0003) before assuming the "categories overlap" rule from 0002 applies universally.

## Versioning policy (semantic versioning, independent of any implementation)

- MINOR bump: adds a category or a field.
- MAJOR bump: changes the shape of an existing category.
- Every PR description must state which kind of version impact it has (or none, for typo/doc-only fixes) — this is enforced by the PR template, not by CI.
- New categories must be proposed via an issue before a PR, since a new category affects every downstream implementation (`wildcard_select`/#3, `json_parse`+`json_extract`+`unnest`/#6, `ai_function`+`udf`+`column_hash`/#8 are the precedents to follow — issue first, then a PR referencing it).
- **Releases**: after a version-bumping PR merges to `main`, add/confirm a `CHANGELOG.md` entry, tag `vMAJOR.MINOR.0` (patch always `0`, matching the schema's `gdt_version` const granularity), and cut a GitHub Release from that tag with notes from the changelog entry. See CONTRIBUTING.md → Releases for the full process. This is manual, not automated — `validate.yml` only checks schema validity/sync, it doesn't cut releases. Note: `gdt_version` bumping is easy to forget when several category-adding PRs land before a release is actually cut (happened once already — v0.2.0 batched three MINOR PRs' worth of additions before the const got bumped) — check the const against `CHANGELOG.md`'s latest entry before assuming they're in sync.

## Relationship to other repos (do not conflate)

- `madflow-sqlops` (separate repo, MIT license) — the reference implementation. A `sqlglot`-based Python package that walks a query's AST and emits gdt-conforming output. This repo has **no dependency** on that one; that one depends on this one's schema version. Don't add `sqlglot`-implementation code here.
- gdt is deliberately independent of any particular consuming application. A consumer's own rules layer (PII/style/strategy concerns, etc.) is out of scope for gdt. If a gdt-native ruleset + verifier is ever built, CONTRIBUTING/README indicate it would be a separate, minimal, gdt-only schema + CLI living in this repo — not folded into `madflow-sqlops` (which stays rule-free by design) and not merged into any consumer's own rules config.

## Licensing

Apache 2.0 for this spec repo (deliberate — the patent grant matters for a spec meant to be reimplemented against parsers other than `sqlglot`). The reference implementation `madflow-sqlops` is MIT, matching `sqlglot`. Don't treat these as needing to match; the split is intentional (see `LICENSE_NOTE.md`).

## Review process

Sole maintainer currently; required PR approvals are set to 0 so merges aren't blocked, but every change still goes through a PR (direct pushes to `main` are disabled by branch protection). This will change once there are regular contributors — don't assume 0-approval merging is a permanent policy when reasoning about process changes.
