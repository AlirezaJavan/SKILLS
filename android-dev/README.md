# Android Dev Skill

This module describes the expectations for the Android Dev skill used by the assistant.

## Core mission

Be an expert Senior Android Engineer and Software Architect who delivers modern Android applications that are scalable, maintainable, performant, and secure.

## Workflow

1. **Read** relevant architecture and style docs before changing code.
2. **Plan** non-trivial changes with a goal, current state, steps, tests, and risks.
3. **Implement** using existing patterns, idiomatic Kotlin, and constructor injection.
4. **Verify** with a per-file review and whole-diff checklist.

## Comment policy

- Default to zero comments.
- Only keep comments for non-trivial algorithm rationale or regulatory/business rationale.
- Do not use comments that restate code, method signatures, or add section banners.

## Key expectations

- Follow Clean Architecture and separate UI, domain, and data layers.
- Use MVVM/MVI and unidirectional data flow.
- Prefer Jetpack Compose, Coroutines, `Flow`, Hilt, Room, Retrofit/Ktor, and Kotlinx Serialization.
- Inject dispatchers and avoid `GlobalScope`.
- Use sealed UI state models for loading, success, and error.
- Prefer descriptive names, guard clauses, and small single-purpose functions.
- Keep code simple, avoid over-engineering, and do not fix unrelated issues.

## Verification

Before reporting done, verify the diff against the skill checklist:

- Clean code and readability
- Function and file design
- SOLID principles
- DRY and KISS
- Architecture and loose coupling
- Maintainability and testability
- Security and privacy
- Concurrency and lifecycle handling
- Scope and boy scout rule

This README is derived from `SKILL.md` and summarizes the skill guidance for contributors and reviewers.