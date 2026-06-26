# Android Structure Checker

This skill audits an Android project against the **Now in Android (NiA)** baseline and generates a migration plan for moving toward the NiA module, build-logic, and version-catalog conventions.

It is driven by the baseline document at `ANDROID_PROJECT_BASELINE.md`, which defines the expected project structure, convention plugin layout, version catalog usage, and module topology.

## What this skill does

- Reads `ANDROID_PROJECT_BASELINE.md` first and treats it as the single source of truth.
- Inspects the target project for foundation, build-logic, module topology, architecture, and code quality markers.
- Produces a gap report that highlights missing or partial compliance.
- Generates an atomic, dependency-ordered migration plan in `nia-migration-plan.md`.

## Key behavior

- The skill does not modify project files directly.
- It produces only a plan; implementation is a separate later action.
- Every gap and plan step must trace back to a rule in `ANDROID_PROJECT_BASELINE.md`.
- Steps are written for a low-capability local AI and use exact paths and literal content.

## Usage

Use this skill when you want to:

- audit an Android repository against NiA conventions,
- identify structural and convention gaps,
- create a precise migration plan for a local executor.

## Files

- `SKILL.md` — skill metadata and execution rules.
- `ANDROID_PROJECT_BASELINE.md` — baseline conventions and architecture expectations.
- `README.md` — this explanation of the skill and how it works.
