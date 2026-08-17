# Contributing to GDT

All changes land via pull request against `main` — direct pushes are disabled by branch protection.

## Spec changes specifically

GDT is a versioned spec other things depend on (`madflow-sqlops` pins a version; more implementations may exist later). Because of that:

- **State the version impact in the PR description**: does this change require a MINOR bump (new category, new optional field) or a MAJOR bump (changes the shape of an existing category)? See `README.md` → Versioning.
- **Propose new categories via an issue first**, before a PR. A new GDT category affects every downstream implementation — worth discussing the `sqlglot` mapping and edge cases before it's code-reviewed as a fait accompli.
- Changes to `schema/gdt-v0.1.schema.json` must keep the schema and `docs/categories.md` in sync — a PR touching one without the other will be asked to fix that before merge.

## Releases

GDT is versioned with git tags and GitHub Releases so downstream implementations (`madflow-sqlops` and any future one) can pin a specific version instead of floating against `main`.

- **Tag format:** `vMAJOR.MINOR.0` — full semver, but the patch slot is always `0` since the spec's own versioning granularity (see `README.md` → Versioning) is MAJOR.MINOR only; it matches the `gdt_version` const in the schema (e.g. schema `"0.1"` → tag `v0.1.0`). A doc/typo fix with no version impact doesn't get a new tag at all — it just lands on `main` between releases.
- **When to cut one:** after a PR that bumps `gdt_version` (or otherwise changes MINOR/MAJOR per the PR template's version-impact checklist) merges to `main`.
- **Process:** on `main`, after merge — add a dated entry to `CHANGELOG.md` (if not already added in the merged PR), tag (`git tag vX.Y.0 && git push origin vX.Y.0`), then create a GitHub Release from that tag with notes copied from the `CHANGELOG.md` entry.
- **Consumers pin by tag, not `main`:** e.g. fetch or vendor `schema/gdt-v0.1.schema.json` from `https://raw.githubusercontent.com/decode-data/gdt/vX.Y.0/schema/gdt-v0.1.schema.json`, or reference the tag/release directly. `madflow-sqlops`'s own README already states this intent ("Pin the GDT schema version explicitly... don't fetch it dynamically at runtime").
- A `validate` GitHub Actions workflow checks that the schema is valid JSON Schema (2020-12) on every PR — it doesn't cut releases automatically; tagging and publishing the Release is still a manual step while there's a single maintainer.

## Review

Sole maintainer currently — required approving reviews is set to 0 so merges aren't blocked, but every change still goes through a PR for the sake of the diff/history, not just for review gating. This will change (required reviews raised) once there are other regular contributors.
