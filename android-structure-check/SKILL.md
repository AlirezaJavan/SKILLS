---
name: android-structure-check
description: Use when auditing an Android project's structure against the Now in Android (NiA) baseline, or when asked to align/migrate a project to the NiA module/build-logic/version-catalog conventions. Audits the current structure, reports gaps, then emits a step-by-step migration plan where each step is small and explicit enough to be executed by a low-capability local AI (e.g. an Ollama model).
---

# Android Structure Checker

Audit an Android project against the **Now in Android (NiA)** baseline, then produce a migration plan whose steps are atomic enough for a medium-intelligence local AI to execute one at a time.

**The baseline is the single source of truth:** `ANDROID_PROJECT_BASELINE.md`. Read it in full before auditing. Every gap and every plan step must trace back to a rule in that file — do not invent conventions.

The skill runs in three phases: **Audit → Gap Report → Step Plan**. Do not write or modify any project file in this skill — it produces a plan only. Implementation is a separate, later action the user approves.

---

## Phase 1 — Audit

Read the baseline file first, then inspect the target project. Discover, don't assume — for each item, record `present / partial / missing` plus the evidence (file path or its absence).

Walk these checkpoints, in order:

### 1. Build foundation
- `settings.gradle.kts`: `includeBuild("build-logic")` present? `enableFeaturePreview("TYPESAFE_PROJECT_ACCESSORS")`? JDK 17 check?
- Root `build.gradle.kts`: all plugins declared `apply false`?
- `gradle/libs.versions.toml`: exists? Are versions centralized there, or hardcoded in module build files?
- `gradle.properties`: parallel builds, configuration cache, `-Xmx` heap set?

### 2. build-logic
- Does `build-logic/` exist as an included build with its own `settings.gradle.kts` sharing `libs.versions.toml`?
- Which convention plugins exist vs. the baseline set (application, library, library-compose, feature-impl, feature-api, hilt, room, jvm-library)?
- Are helper files present (`KotlinAndroid.kt`, `AndroidCompose.kt`, `ProjectExtensions.kt`)?

### 3. Module topology
- Is logic concentrated in a monolithic `:app`, or split into `:core:*` and `:feature:*`?
- For each core concern (model, common, network, database, data, domain, designsystem, ui, testing) — present as its own module?
- Do features follow the `:feature:X:api` + `:feature:X:impl` split?
- Is `:core:model` / `:core:common` pure JVM (no Android imports)?

### 4. Architecture conventions
- Single `@AndroidEntryPoint` `MainActivity` + `@HiltAndroidApp` Application?
- ViewModels expose `StateFlow` via `stateIn(viewModelScope, WhileSubscribed(5000), …)`?
- Sealed-interface `UiState` per screen? Stateless Screen + `*Route` split?
- Repository interface in `:core:data` with `Default*` impl bound by `@Binds`?
- Dispatchers injected, not hardcoded?

### 5. Code quality markers
- Namespace pattern `com.<co>.<app>.<layer>.<module>`?
- Hardcoded user-facing strings or design tokens?
- `collectAsStateWithLifecycle` vs `collectAsState`? Any `GlobalScope`?

Use the read-only tools (Glob, Grep, Read) for this phase. Prefer reading actual `build.gradle.kts` files over guessing from directory names.

---

## Phase 2 — Gap Report

Output a concise table the user reads before committing to migration. One row per gap, ordered by dependency (foundation gaps first, because later steps depend on them).

```
| # | Area | Baseline expectation | Current state | Severity |
|---|------|----------------------|---------------|----------|
| 1 | build-logic | Included build w/ convention plugins | Absent — config inline per module | Blocker |
| 2 | Version catalog | All versions in libs.versions.toml | Partial — 6 versions hardcoded in :app | High |
| … |
```

Severity scale:
- **Blocker** — nothing else can be aligned until this is fixed (e.g. no version catalog, no build-logic).
- **High** — structural divergence from NiA (monolith, no api/impl split).
- **Medium** — convention/naming divergence inside an otherwise-correct structure.
- **Low** — cosmetic (resource prefixing, idiom polish).

If the project already matches the baseline on an area, omit it from the table — only report gaps. End the report with a one-line summary: how many blockers/high/medium/low.

---

## Phase 3 — Step Plan (for a low-capability local AI)

This is the deliverable. The executing agent is assumed to be a **small local model with limited reasoning** — it cannot infer, cross-reference, or hold the whole project in its head. Every step must therefore be **atomic, ordered, and self-contained.**

Write the plan to a file at the project root: `nia-migration-plan.md`.

### Rules for every step

1. **One action per step.** Create one file, OR edit one file, OR move one directory. Never "create the data layer" — that is dozens of steps.
2. **Absolute, exact paths.** `core/network/build.gradle.kts`, not "the network module's build file".
3. **Literal content, not description.** Paste the full file content or the exact before/after edit block. The executor copies; it does not compose. Pull the content verbatim from `ANDROID_PROJECT_BASELINE.md` and adapt only the package/app name.
4. **Explicit ordering with dependencies.** Number steps. If step 12 needs step 4 done first, say `(requires: step 4)`. Foundation (version catalog → build-logic → settings) always comes before modules; modules before app wiring.
5. **A verification line per step.** A single command or check the executor can run to confirm success, e.g. `Verify: run \`./gradlew :core:model:compileKotlin\` — it must succeed.`
6. **No judgment calls.** If a choice is needed (package name, min SDK), resolve it once at the top of the plan as a "Constants" block and reference it; never leave the executor to decide.

### Plan file template

```markdown
# NiA Migration Plan for <project>

## Constants (use these literal values in every step)
- BASE_PACKAGE: com.acme.pulse
- APP_NAMESPACE: com.acme.pulse
- MIN_SDK: 26
- COMPILE_SDK: 36

## Step 1 — Create the version catalog
File: gradle/libs.versions.toml
Action: Create this file with exactly the following content:
```
<full literal content>
```
Verify: file exists and `./gradlew help` runs without "version catalog" errors.

## Step 2 — …
(requires: step 1)
…
```

### Sizing

Group the steps under phase headers that mirror the dependency order, but keep each numbered step atomic:

1. **Foundation** — version catalog, `gradle.properties`, root build file, settings plugin management.
2. **build-logic** — settings, convention `build.gradle.kts`, then one step per convention plugin / helper file.
3. **Core modules** — one module at a time, each as: create dir → create build file → create source files. Order by dependency (`model` and `common` first, then `network`/`database`, then `data`, then `domain`, `designsystem`, `ui`, `testing`).
4. **Feature modules** — api module then impl module, one step per file.
5. **App wiring** — convert `:app` build file, register modules in settings, add entry providers.
6. **Cleanup** — remove now-dead inline config, fix namespaces, move misplaced files.

A correct plan for a greenfield-to-NiA migration is typically 40–120 steps. That is expected; granularity is the point.

---

## Guardrails

- **Plan only.** This skill never edits project files. It writes exactly one artifact: `nia-migration-plan.md`. If the user wants execution, that is a separate approved action.
- **Trace to baseline.** Every gap and step cites the relevant section of `ANDROID_PROJECT_BASELINE.md`. If something isn't in the baseline, it isn't in the plan.
- **Don't fabricate the current state.** If you can't read a file, say so and mark the checkpoint `unknown` — never assume it passes or fails.
- **Preserve working code.** Migration steps move and rewire existing source; they do not silently discard it. When a file must move, the step is "move X to Y", not "create Y" (which would orphan X).
- **Respect existing names.** If the project already has a base package, use it as a Constant — don't impose `com.myapp`.

## Verification before reporting done

- [ ] Read `ANDROID_PROJECT_BASELINE.md` in full this session.
- [ ] Audit covers all five checkpoint groups; each item marked present/partial/missing with evidence.
- [ ] Gap report is gaps-only, severity-ordered, with a count summary.
- [ ] `nia-migration-plan.md` written with a Constants block and atomic, numbered, dependency-ordered steps.
- [ ] Every step has an exact path, literal content/edit, and a verify line.
- [ ] No project file other than `nia-migration-plan.md` was created or modified.
