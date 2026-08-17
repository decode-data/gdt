# Contributing to GDTO

All changes land via pull request against `main` — direct pushes are disabled by branch protection.

## Spec changes specifically

GDTO is a versioned spec other things depend on (`madflow-sqlops` pins a version; more implementations may exist later). Because of that:

- **State the version impact in the PR description**: does this change require a MINOR bump (new category, new optional field) or a MAJOR bump (changes the shape of an existing category)? See `README.md` → Versioning.
- **Propose new categories via an issue first**, before a PR. A new GDTO category affects every downstream implementation — worth discussing the `sqlglot` mapping and edge cases before it's code-reviewed as a fait accompli.
- Changes to `schema/gdto-v0.1.schema.json` must keep the schema and `docs/categories.md` in sync — a PR touching one without the other will be asked to fix that before merge.

## Review

Sole maintainer currently — required approving reviews is set to 0 so merges aren't blocked, but every change still goes through a PR for the sake of the diff/history, not just for review gating. This will change (required reviews raised) once there are other regular contributors.
