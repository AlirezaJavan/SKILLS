# SKILLS

This repository contains Claude Code skill modules for Android development.

## Available skills

- `android-dev/`
  - Expert Android development guidance for Maintainable, Asynchronous, and Dependable (MAD) Android apps.
  - Includes rules for reading architecture docs, planning non-trivial changes, implementing idiomatic Kotlin/Compose/Hilt code, and verifying diffs against a clean code checklist.
  - See `android-dev/README.md` for details.

- `android-structure-check/`
  - Audit Android project structure against the Now in Android (NiA) baseline.
  - Produces a gap report and an atomic `nia-migration-plan.md` for low-capability local AI execution.
  - Uses `ANDROID_PROJECT_BASELINE.md` as the single source of truth.
  - See `android-structure-check/README.md` for details.

## Repository layout

- `android-dev/` — Android development skill guidance and conventions.
- `android-structure-check/` — Android project auditing and migration planning skill.
