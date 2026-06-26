# Android Project Baseline Guide

This document defines the baseline architecture, module structure, build system, and code conventions for all new Android projects. It is modeled after the **Now in Android (NiA)** open-source project and must be followed precisely by any AI agent creating a new project or adding to an existing one.

---

## Table of Contents

1. [Project Philosophy](#1-project-philosophy)
2. [Directory Structure Overview](#2-directory-structure-overview)
3. [Root Files Setup](#3-root-files-setup)
4. [Version Catalog (`libs.versions.toml`)](#4-version-catalog)
5. [build-logic: Convention Plugins](#5-build-logic-convention-plugins)
6. [Core Modules](#6-core-modules)
7. [Feature Modules (api + impl split)](#7-feature-modules)
8. [App Module](#8-app-module)
9. [Navigation Architecture](#9-navigation-architecture)
10. [Compose UI Patterns](#10-compose-ui-patterns)
11. [Adding a New Feature](#11-adding-a-new-feature)
12. [Adding a New Core Module](#12-adding-a-new-core-module)
13. [Testing Patterns](#13-testing-patterns)
14. [Code Style & Naming Rules](#14-code-style--naming-rules)
15. [Checklist for New Projects](#15-checklist-for-new-projects)

---

## 1. Project Philosophy

- **Single-activity** Compose app. No fragments. One `MainActivity`.
- **Unidirectional Data Flow (UDF)**: UI emits events → ViewModel processes → emits immutable `UiState` → UI renders.
- **Clean Architecture layers**: `UI → ViewModel → UseCase → Repository → DataSource`.
- **Hilt** for dependency injection everywhere. Never instantiate dependencies inside business logic.
- **Modularization-first**: every feature is two modules — `:feature:X:api` (public contract) and `:feature:X:impl` (implementation). Every shared concern is a `:core:X` module.
- **Convention plugins** in `build-logic/` replace per-module boilerplate. Every `build.gradle.kts` in a feature or core module is ≤ 30 lines.
- **Version catalog** (`gradle/libs.versions.toml`) is the single source of truth for all library versions.
- **No mocking** in tests. Use lightweight `Fake*` implementations of repository/datasource interfaces.
- **Kotlin only**. No Java for new files. Idiomatic Kotlin: data classes, sealed classes, extension functions, scope functions.

---

## 2. Directory Structure Overview

```
root/
├── app/                          # Application module (single entry point)
├── build-logic/                  # Convention plugins (shared build logic)
│   ├── convention/
│   │   ├── build.gradle.kts
│   │   └── src/main/kotlin/
│   │       ├── AndroidApplicationConventionPlugin.kt
│   │       ├── AndroidLibraryConventionPlugin.kt
│   │       ├── AndroidFeatureImplConventionPlugin.kt
│   │       ├── AndroidFeatureApiConventionPlugin.kt
│   │       ├── AndroidLibraryComposeConventionPlugin.kt
│   │       ├── AndroidRoomConventionPlugin.kt
│   │       ├── HiltConventionPlugin.kt
│   │       ├── JvmLibraryConventionPlugin.kt
│   │       └── ... (helpers: KotlinAndroid.kt, AndroidCompose.kt, etc.)
│   └── settings.gradle.kts
├── core/
│   ├── analytics/
│   ├── common/                   # Pure JVM utilities (no Android framework)
│   ├── data/                     # Repository implementations
│   ├── data-test/                # Fake repositories for tests
│   ├── database/                 # Room database, DAOs
│   ├── datastore/                # DataStore preferences
│   ├── datastore-proto/          # Protocol Buffers for DataStore
│   ├── datastore-test/
│   ├── designsystem/             # Theme, colors, typography, icons
│   ├── domain/                   # Use cases (pure Kotlin)
│   ├── model/                    # Data models (pure JVM, no Android)
│   ├── navigation/               # Navigation keys, destinations
│   ├── network/                  # Retrofit clients, DTOs
│   ├── notifications/
│   ├── screenshot-testing/
│   ├── testing/                  # Shared test utilities, Fake implementations
│   └── ui/                       # Reusable Compose components
├── feature/
│   ├── home/
│   │   ├── api/                  # Public: NavKey, navigation contract
│   │   └── impl/                 # Private: Screen, ViewModel, navigation
│   └── settings/
│       └── impl/                 # Settings has no api (no deep link into it)
├── sync/
│   ├── work/                     # WorkManager sync tasks
│   └── sync-test/
├── lint/                         # Custom lint checks
├── benchmarks/                   # Macrobenchmark, baseline profile generation
├── gradle/
│   ├── libs.versions.toml        # All library/plugin versions
│   └── wrapper/
│       └── gradle-wrapper.properties
├── build.gradle.kts              # Root: declares all plugins (apply false)
├── settings.gradle.kts           # Module includes, plugin management
├── gradle.properties             # JVM args, build flags
└── local.defaults.properties     # Default local config (backend URL, etc.)
```

---

## 3. Root Files Setup

### `settings.gradle.kts`

```kotlin
pluginManagement {
    // build-logic is an included build — its plugins are available to all modules
    includeBuild("build-logic")
    repositories {
        google {
            content {
                includeGroupByRegex("com\\.android.*")
                includeGroupByRegex("com\\.google.*")
                includeGroupByRegex("androidx.*")
            }
        }
        mavenCentral()
        gradlePluginPortal()
    }
}

dependencyResolutionManagement {
    repositoriesMode = RepositoriesMode.FAIL_ON_PROJECT_REPOS
    repositories {
        google {
            content {
                includeGroupByRegex("com\\.android.*")
                includeGroupByRegex("com\\.google.*")
                includeGroupByRegex("androidx.*")
            }
        }
        mavenCentral()
    }
}

rootProject.name = "myapp"

// Enables projects.core.data, projects.feature.home.api syntax
enableFeaturePreview("TYPESAFE_PROJECT_ACCESSORS")

// Enforce JDK 17+
check(JavaVersion.current().isCompatibleWith(JavaVersion.VERSION_17)) {
    "This project requires JDK 17+. Current: ${JavaVersion.current()}"
}

// --- Modules ---
include(":app")

include(":core:analytics")
include(":core:common")
include(":core:data")
include(":core:data-test")
include(":core:database")
include(":core:datastore")
include(":core:datastore-proto")
include(":core:datastore-test")
include(":core:designsystem")
include(":core:domain")
include(":core:model")
include(":core:navigation")
include(":core:network")
include(":core:notifications")
include(":core:screenshot-testing")
include(":core:testing")
include(":core:ui")

include(":feature:home:api")
include(":feature:home:impl")
include(":feature:settings:impl")

include(":lint")
include(":sync:work")
include(":sync:sync-test")
```

### `build.gradle.kts` (root)

The root build file only declares plugins used across the project with `apply false`. This keeps the classpath consistent.

```kotlin
plugins {
    alias(libs.plugins.android.application) apply false
    alias(libs.plugins.android.library) apply false
    alias(libs.plugins.android.test) apply false
    alias(libs.plugins.compose) apply false
    alias(libs.plugins.kotlin.jvm) apply false
    alias(libs.plugins.kotlin.serialization) apply false
    alias(libs.plugins.hilt) apply false
    alias(libs.plugins.ksp) apply false
    alias(libs.plugins.room) apply false
    alias(libs.plugins.spotless) apply false

    // Custom root plugin (dependency graph, etc.)
    alias(libs.plugins.myapp.root)
}
```

### `gradle.properties`

```properties
# Build performance
org.gradle.jvmargs=-Xmx4g -XX:+UseG1GC -XX:MaxMetaspaceSize=512m -XX:+HeapDumpOnOutOfMemoryError -Dfile.encoding=UTF-8
org.gradle.parallel=true
org.gradle.caching=true
org.gradle.configuration-cache=true
kotlin.daemon.jvm.options=-Xmx2g

# Android
android.useAndroidX=true
android.nonTransitiveRClass=true
android.enableJetifier=false
android.defaults.buildfeatures.resvalues=false
android.defaults.buildfeatures.shaders=false

# Kotlin
kotlin.code.style=official
```

### `local.defaults.properties`

```properties
# Checked in — safe defaults only. Developers override in local.properties (git-ignored).
BACKEND_URL=https://api.example.com/
```

---

## 4. Version Catalog

**Path:** `gradle/libs.versions.toml`

This is the single source of truth for ALL versions. Never hardcode a version inside a `build.gradle.kts`.

```toml
[versions]
# Build tools
androidGradlePlugin = "9.0.0"
kotlin = "2.3.0"
ksp = "2.3.0-1.0.29"

# AndroidX
androidxActivity = "1.9.3"
androidxComposeBom = "2025.09.01"
androidxCoreSplashscreen = "1.0.1"
androidxDataStore = "1.2.0"
androidxLifecycle = "2.10.0"
androidxNavigation = "2.8.5"
androidxNavigation3 = "1.0.0"
androidxRoom = "2.8.3"
androidxTestCore = "1.7.0-rc01"
androidxTestEspresso = "3.6.1"
androidxTestRunner = "1.6.2"
androidxTracing = "1.3.0"
androidxWindowManager = "1.4.0"
androidxWork = "2.10.0"

# DI
hilt = "2.59"
hiltExt = "1.2.0"

# Networking
okhttp = "4.12.0"
retrofit = "2.11.0"

# Storage
protobuf = "4.29.2"
protobufPlugin = "0.9.6"

# Image
coil = "2.7.0"

# Async
kotlinxCoroutines = "1.10.1"
kotlinxDatetime = "0.6.1"
kotlinxSerialization = "1.8.1"

# Testing
junit = "4.13.2"
junit5 = "5.11.4"
robolectric = "4.16"
roborazzi = "1.56.0"
truth = "1.4.4"
turbine = "1.2.0"
mockk = "1.13.16"

# Quality
spotless = "8.3.0"
detekt = "1.23.8"

# Firebase (optional)
firebaseBom = "33.7.0"
firebaseCrashlyticsPlugin = "3.0.6"
gmsPlugin = "4.4.4"

# Desugar
androidDesugarJdkLibs = "2.1.5"

[libraries]
# --- AndroidX ---
androidx-activity-compose = { group = "androidx.activity", name = "activity-compose", version.ref = "androidxActivity" }
androidx-compose-bom = { group = "androidx.compose", name = "compose-bom", version.ref = "androidxComposeBom" }
androidx-compose-foundation = { group = "androidx.compose.foundation", name = "foundation" }
androidx-compose-foundation-layout = { group = "androidx.compose.foundation", name = "foundation-layout" }
androidx-compose-material-iconsExtended = { group = "androidx.compose.material", name = "material-icons-extended" }
androidx-compose-material3 = { group = "androidx.compose.material3", name = "material3" }
androidx-compose-material3-adaptive = { group = "androidx.compose.material3.adaptive", name = "adaptive" }
androidx-compose-material3-navigationSuite = { group = "androidx.compose.material3", name = "material3-adaptive-navigation-suite" }
androidx-compose-runtime = { group = "androidx.compose.runtime", name = "runtime" }
androidx-compose-runtime-tracing = { group = "androidx.compose.runtime", name = "runtime-tracing" }
androidx-compose-ui = { group = "androidx.compose.ui", name = "ui" }
androidx-compose-ui-test = { group = "androidx.compose.ui", name = "ui-test-junit4" }
androidx-compose-ui-testManifest = { group = "androidx.compose.ui", name = "ui-test-manifest" }
androidx-compose-ui-tooling = { group = "androidx.compose.ui", name = "ui-tooling" }
androidx-compose-ui-tooling-preview = { group = "androidx.compose.ui", name = "ui-tooling-preview" }
androidx-compose-ui-util = { group = "androidx.compose.ui", name = "ui-util" }
androidx-core-ktx = { group = "androidx.core", name = "core-ktx", version = "1.16.0" }
androidx-core-splashscreen = { group = "androidx.core", name = "core-splashscreen", version.ref = "androidxCoreSplashscreen" }
androidx-datastore-core = { group = "androidx.datastore", name = "datastore", version.ref = "androidxDataStore" }
androidx-datastore-preferences = { group = "androidx.datastore", name = "datastore-preferences", version.ref = "androidxDataStore" }
androidx-hilt-navigation-compose = { group = "androidx.hilt", name = "hilt-navigation-compose", version.ref = "hiltExt" }
androidx-lifecycle-runtimeCompose = { group = "androidx.lifecycle", name = "lifecycle-runtime-compose", version.ref = "androidxLifecycle" }
androidx-lifecycle-viewModelCompose = { group = "androidx.lifecycle", name = "lifecycle-viewmodel-compose", version.ref = "androidxLifecycle" }
androidx-navigation-compose = { group = "androidx.navigation", name = "navigation-compose", version.ref = "androidxNavigation" }
androidx-navigation3-runtime = { group = "androidx.navigation3", name = "navigation3-runtime", version.ref = "androidxNavigation3" }
androidx-navigation3-ui = { group = "androidx.navigation3", name = "navigation3-ui", version.ref = "androidxNavigation3" }
androidx-room-compiler = { group = "androidx.room", name = "room-compiler", version.ref = "androidxRoom" }
androidx-room-ktx = { group = "androidx.room", name = "room-ktx", version.ref = "androidxRoom" }
androidx-room-runtime = { group = "androidx.room", name = "room-runtime", version.ref = "androidxRoom" }
androidx-test-core = { group = "androidx.test", name = "core", version.ref = "androidxTestCore" }
androidx-test-espresso-core = { group = "androidx.test.espresso", name = "espresso-core", version.ref = "androidxTestEspresso" }
androidx-test-runner = { group = "androidx.test", name = "runner", version.ref = "androidxTestRunner" }
androidx-tracing-ktx = { group = "androidx.tracing", name = "tracing-ktx", version.ref = "androidxTracing" }
androidx-window-core = { group = "androidx.window", name = "window-core", version.ref = "androidxWindowManager" }
androidx-work-ktx = { group = "androidx.work", name = "work-runtime-ktx", version.ref = "androidxWork" }
androidx-work-testing = { group = "androidx.work", name = "work-testing", version.ref = "androidxWork" }

# --- Coil ---
coil-kt = { group = "io.coil-kt", name = "coil", version.ref = "coil" }
coil-kt-compose = { group = "io.coil-kt", name = "coil-compose", version.ref = "coil" }
coil-kt-svg = { group = "io.coil-kt", name = "coil-svg", version.ref = "coil" }

# --- DI ---
hilt-android = { group = "com.google.dagger", name = "hilt-android", version.ref = "hilt" }
hilt-android-testing = { group = "com.google.dagger", name = "hilt-android-testing", version.ref = "hilt" }
hilt-compiler = { group = "com.google.dagger", name = "hilt-android-compiler", version.ref = "hilt" }
javax-inject = { group = "javax.inject", name = "javax.inject", version = "1" }

# --- Kotlin ---
kotlin-stdlib = { group = "org.jetbrains.kotlin", name = "kotlin-stdlib", version.ref = "kotlin" }
kotlin-test = { group = "org.jetbrains.kotlin", name = "kotlin-test", version.ref = "kotlin" }
kotlinx-coroutines-android = { group = "org.jetbrains.kotlinx", name = "kotlinx-coroutines-android", version.ref = "kotlinxCoroutines" }
kotlinx-coroutines-core = { group = "org.jetbrains.kotlinx", name = "kotlinx-coroutines-core", version.ref = "kotlinxCoroutines" }
kotlinx-coroutines-test = { group = "org.jetbrains.kotlinx", name = "kotlinx-coroutines-test", version.ref = "kotlinxCoroutines" }
kotlinx-datetime = { group = "org.jetbrains.kotlinx", name = "kotlinx-datetime", version.ref = "kotlinxDatetime" }
kotlinx-serialization-json = { group = "org.jetbrains.kotlinx", name = "kotlinx-serialization-json", version.ref = "kotlinxSerialization" }

# --- Network ---
okhttp-logging = { group = "com.squareup.okhttp3", name = "logging-interceptor", version.ref = "okhttp" }
retrofit-core = { group = "com.squareup.retrofit2", name = "retrofit", version.ref = "retrofit" }
retrofit-kotlin-serialization = { group = "com.squareup.retrofit2", name = "converter-kotlinx-serialization", version.ref = "retrofit" }

# --- Protobuf ---
protobuf-kotlin-lite = { group = "com.google.protobuf", name = "protobuf-kotlin-lite", version.ref = "protobuf" }
protobuf-protoc = { group = "com.google.protobuf", name = "protoc", version.ref = "protobuf" }

# --- Testing ---
junit = { group = "junit", name = "junit", version.ref = "junit" }
mockk = { group = "io.mockk", name = "mockk", version.ref = "mockk" }
robolectric = { group = "org.robolectric", name = "robolectric", version.ref = "robolectric" }
roborazzi = { group = "io.github.takahirom.roborazzi", name = "roborazzi", version.ref = "roborazzi" }
roborazzi-compose = { group = "io.github.takahirom.roborazzi", name = "roborazzi-compose", version.ref = "roborazzi" }
truth = { group = "com.google.truth", name = "truth", version.ref = "truth" }
turbine = { group = "app.cash.turbine", name = "turbine", version.ref = "turbine" }

# --- Desugar ---
android-desugarJdkLibs = { group = "com.android.tools", name = "desugar_jdk_libs", version.ref = "androidDesugarJdkLibs" }

[bundles]
androidx-compose-ui-test = ["androidx-compose-ui-test", "androidx-compose-ui-testManifest"]

[plugins]
# AGP
android-application = { id = "com.android.application", version.ref = "androidGradlePlugin" }
android-library = { id = "com.android.library", version.ref = "androidGradlePlugin" }
android-test = { id = "com.android.test", version.ref = "androidGradlePlugin" }

# Kotlin
kotlin-jvm = { id = "org.jetbrains.kotlin.jvm", version.ref = "kotlin" }
kotlin-serialization = { id = "org.jetbrains.kotlin.plugin.serialization", version.ref = "kotlin" }
compose = { id = "org.jetbrains.kotlin.plugin.compose", version.ref = "kotlin" }

# KSP / Room / Hilt
ksp = { id = "com.google.devtools.ksp", version.ref = "ksp" }
room = { id = "androidx.room", version.ref = "androidxRoom" }
hilt = { id = "com.google.dagger.hilt.android", version.ref = "hilt" }

# Protobuf
protobuf = { id = "com.google.protobuf", version.ref = "protobufPlugin" }

# Quality
spotless = { id = "com.diffplug.spotless", version.ref = "spotless" }
roborazzi = { id = "io.github.takahirom.roborazzi", version.ref = "roborazzi" }

# Firebase (optional)
firebase-crashlytics = { id = "com.google.firebase.crashlytics", version.ref = "firebaseCrashlyticsPlugin" }
gms = { id = "com.google.gms.google-services", version.ref = "gmsPlugin" }

# Custom convention plugins — defined in build-logic/convention/
myapp-android-application = { id = "myapp.android.application" }
myapp-android-application-compose = { id = "myapp.android.application.compose" }
myapp-android-application-flavors = { id = "myapp.android.application.flavors" }
myapp-android-library = { id = "myapp.android.library" }
myapp-android-library-compose = { id = "myapp.android.library.compose" }
myapp-android-feature-impl = { id = "myapp.android.feature.impl" }
myapp-android-feature-api = { id = "myapp.android.feature.api" }
myapp-android-room = { id = "myapp.android.room" }
myapp-hilt = { id = "myapp.hilt" }
myapp-jvm-library = { id = "myapp.jvm.library" }
myapp-root = { id = "myapp.root" }
```

---

## 5. build-logic: Convention Plugins

The `build-logic/` directory is an **included build** that provides shared Gradle convention plugins. This keeps module-level `build.gradle.kts` files minimal (5–30 lines) by centralizing boilerplate.

### `build-logic/settings.gradle.kts`

```kotlin
dependencyResolutionManagement {
    repositories {
        google()
        mavenCentral()
    }
    versionCatalogs {
        create("libs") {
            from(files("../gradle/libs.versions.toml"))
        }
    }
}

rootProject.name = "build-logic"
include(":convention")
```

### `build-logic/convention/build.gradle.kts`

```kotlin
import org.jetbrains.kotlin.gradle.dsl.JvmTarget

plugins {
    `kotlin-dsl`
}

group = "com.myapp.buildlogic"

java {
    sourceCompatibility = JavaVersion.VERSION_17
    targetCompatibility = JavaVersion.VERSION_17
}

kotlin {
    compilerOptions {
        jvmTarget = JvmTarget.JVM_17
    }
}

dependencies {
    compileOnly(libs.android.gradlePlugin)
    compileOnly(libs.android.tools.common)
    compileOnly(libs.compose.gradlePlugin)
    compileOnly(libs.kotlin.gradlePlugin)
    compileOnly(libs.ksp.gradlePlugin)
    compileOnly(libs.room.gradlePlugin)
}

// Register each convention plugin
gradlePlugin {
    plugins {
        register("androidApplication") {
            id = "myapp.android.application"
            implementationClass = "AndroidApplicationConventionPlugin"
        }
        register("androidApplicationCompose") {
            id = "myapp.android.application.compose"
            implementationClass = "AndroidApplicationComposeConventionPlugin"
        }
        register("androidApplicationFlavors") {
            id = "myapp.android.application.flavors"
            implementationClass = "AndroidApplicationFlavorsConventionPlugin"
        }
        register("androidLibrary") {
            id = "myapp.android.library"
            implementationClass = "AndroidLibraryConventionPlugin"
        }
        register("androidLibraryCompose") {
            id = "myapp.android.library.compose"
            implementationClass = "AndroidLibraryComposeConventionPlugin"
        }
        register("androidFeatureImpl") {
            id = "myapp.android.feature.impl"
            implementationClass = "AndroidFeatureImplConventionPlugin"
        }
        register("androidFeatureApi") {
            id = "myapp.android.feature.api"
            implementationClass = "AndroidFeatureApiConventionPlugin"
        }
        register("androidRoom") {
            id = "myapp.android.room"
            implementationClass = "AndroidRoomConventionPlugin"
        }
        register("hilt") {
            id = "myapp.hilt"
            implementationClass = "HiltConventionPlugin"
        }
        register("jvmLibrary") {
            id = "myapp.jvm.library"
            implementationClass = "JvmLibraryConventionPlugin"
        }
        register("root") {
            id = "myapp.root"
            implementationClass = "RootConventionPlugin"
        }
    }
}
```

### Helper: `KotlinAndroid.kt`

```kotlin
// build-logic/convention/src/main/kotlin/com/myapp/buildlogic/KotlinAndroid.kt
package com.myapp.buildlogic

import com.android.build.api.dsl.CommonExtension
import org.gradle.api.Project
import org.gradle.api.plugins.JavaPluginExtension
import org.gradle.kotlin.dsl.configure
import org.gradle.kotlin.dsl.dependencies
import org.gradle.kotlin.dsl.withType
import org.jetbrains.kotlin.gradle.dsl.JvmTarget
import org.jetbrains.kotlin.gradle.tasks.KotlinCompile

internal fun Project.configureKotlinAndroid(
    commonExtension: CommonExtension<*, *, *, *, *, *>,
) {
    commonExtension.apply {
        compileSdk = 36

        defaultConfig {
            minSdk = 26
        }

        compileOptions {
            sourceCompatibility = JavaVersion.VERSION_11
            targetCompatibility = JavaVersion.VERSION_11
            isCoreLibraryDesugaringEnabled = true
        }
    }

    configureKotlin()

    dependencies {
        "coreLibraryDesugaring"(libs.findLibrary("android.desugarJdkLibs").get())
    }
}

internal fun Project.configureKotlinJvm() {
    extensions.configure<JavaPluginExtension> {
        sourceCompatibility = JavaVersion.VERSION_11
        targetCompatibility = JavaVersion.VERSION_11
    }
    configureKotlin()
}

private fun Project.configureKotlin() {
    tasks.withType<KotlinCompile>().configureEach {
        compilerOptions {
            jvmTarget.set(JvmTarget.JVM_11)
            freeCompilerArgs.addAll(
                listOf(
                    "-opt-in=kotlinx.coroutines.ExperimentalCoroutinesApi",
                )
            )
        }
    }
}
```

### Helper: `AndroidCompose.kt`

```kotlin
// build-logic/convention/src/main/kotlin/com/myapp/buildlogic/AndroidCompose.kt
package com.myapp.buildlogic

import com.android.build.api.dsl.CommonExtension
import org.gradle.api.Project
import org.gradle.kotlin.dsl.dependencies

internal fun Project.configureAndroidCompose(
    commonExtension: CommonExtension<*, *, *, *, *, *>,
) {
    commonExtension.apply {
        buildFeatures {
            compose = true
        }
    }

    dependencies {
        val bom = libs.findLibrary("androidx-compose-bom").get()
        "implementation"(platform(bom))
        "androidTestImplementation"(platform(bom))
        "implementation"(libs.findLibrary("androidx-compose-ui-tooling-preview").get())
        "debugImplementation"(libs.findLibrary("androidx-compose-ui-tooling").get())
    }
}
```

### Helper: `AppFlavors.kt`

```kotlin
// build-logic/convention/src/main/kotlin/com/myapp/buildlogic/AppFlavors.kt
package com.myapp.buildlogic

import com.android.build.api.dsl.ApplicationExtension
import com.android.build.api.dsl.ApplicationProductFlavor
import org.gradle.api.Project

enum class AppFlavor(val dimension: String, val applicationIdSuffix: String?) {
    Demo(dimension = "contentType", applicationIdSuffix = ".demo"),
    Prod(dimension = "contentType", applicationIdSuffix = null),
}

enum class AppBuildType(val applicationIdSuffix: String?) {
    Debug(applicationIdSuffix = ".debug"),
    Release(applicationIdSuffix = null),
}

internal fun Project.configureFlavors(
    applicationExtension: ApplicationExtension,
) {
    applicationExtension.apply {
        flavorDimensions += AppFlavor.Demo.dimension
        productFlavors {
            AppFlavor.values().forEach { flavor ->
                create(flavor.name.lowercase()) {
                    dimension = flavor.dimension
                    if (flavor.applicationIdSuffix != null) {
                        applicationIdSuffix = flavor.applicationIdSuffix
                    }
                }
            }
        }
    }
}
```

### Convention Plugin: `AndroidApplicationConventionPlugin.kt`

```kotlin
// build-logic/convention/src/main/kotlin/AndroidApplicationConventionPlugin.kt
import com.android.build.api.dsl.ApplicationExtension
import com.myapp.buildlogic.AppBuildType
import com.myapp.buildlogic.configureKotlinAndroid
import org.gradle.api.Plugin
import org.gradle.api.Project
import org.gradle.kotlin.dsl.configure

class AndroidApplicationConventionPlugin : Plugin<Project> {
    override fun apply(target: Project) {
        with(target) {
            with(pluginManager) {
                apply("com.android.application")
                apply("org.jetbrains.kotlin.android")
            }

            extensions.configure<ApplicationExtension> {
                configureKotlinAndroid(this)
                defaultConfig.targetSdk = 36
                buildTypes {
                    debug {
                        applicationIdSuffix = AppBuildType.Debug.applicationIdSuffix
                    }
                    release {
                        isMinifyEnabled = true
                        applicationIdSuffix = AppBuildType.Release.applicationIdSuffix
                        proguardFiles(
                            getDefaultProguardFile("proguard-android-optimize.txt"),
                            "proguard-rules.pro",
                        )
                    }
                }
            }
        }
    }
}
```

### Convention Plugin: `AndroidApplicationComposeConventionPlugin.kt`

```kotlin
import com.android.build.api.dsl.ApplicationExtension
import com.myapp.buildlogic.configureAndroidCompose
import org.gradle.api.Plugin
import org.gradle.api.Project
import org.gradle.kotlin.dsl.configure

class AndroidApplicationComposeConventionPlugin : Plugin<Project> {
    override fun apply(target: Project) {
        with(target) {
            pluginManager.apply("org.jetbrains.kotlin.plugin.compose")
            extensions.configure<ApplicationExtension> {
                configureAndroidCompose(this)
            }
        }
    }
}
```

### Convention Plugin: `AndroidLibraryConventionPlugin.kt`

```kotlin
import com.android.build.api.dsl.LibraryExtension
import com.myapp.buildlogic.configureKotlinAndroid
import com.myapp.buildlogic.libs
import org.gradle.api.Plugin
import org.gradle.api.Project
import org.gradle.kotlin.dsl.configure
import org.gradle.kotlin.dsl.dependencies

class AndroidLibraryConventionPlugin : Plugin<Project> {
    override fun apply(target: Project) {
        with(target) {
            with(pluginManager) {
                apply("com.android.library")
                apply("org.jetbrains.kotlin.android")
            }

            extensions.configure<LibraryExtension> {
                configureKotlinAndroid(this)
                defaultConfig.targetSdk = 36

                // Auto-prefix resources with module path to avoid clashes
                // e.g., :core:database → prefix "core_database_"
                val resourcePrefix = path
                    .split(":")
                    .drop(1) // remove empty first element
                    .joinToString(separator = "_")
                    .lowercase() + "_"
                this.resourcePrefix = resourcePrefix
            }

            dependencies {
                "testImplementation"(kotlin("test"))
                "androidTestImplementation"(kotlin("test"))
            }
        }
    }
}
```

### Convention Plugin: `AndroidLibraryComposeConventionPlugin.kt`

```kotlin
import com.android.build.api.dsl.LibraryExtension
import com.myapp.buildlogic.configureAndroidCompose
import org.gradle.api.Plugin
import org.gradle.api.Project
import org.gradle.kotlin.dsl.configure

class AndroidLibraryComposeConventionPlugin : Plugin<Project> {
    override fun apply(target: Project) {
        with(target) {
            pluginManager.apply("org.jetbrains.kotlin.plugin.compose")
            extensions.configure<LibraryExtension> {
                configureAndroidCompose(this)
            }
        }
    }
}
```

### Convention Plugin: `AndroidFeatureImplConventionPlugin.kt`

This is the most important plugin for feature modules. It automatically adds navigation, Hilt, and lifecycle dependencies.

```kotlin
import com.myapp.buildlogic.libs
import org.gradle.api.Plugin
import org.gradle.api.Project
import org.gradle.kotlin.dsl.dependencies

class AndroidFeatureImplConventionPlugin : Plugin<Project> {
    override fun apply(target: Project) {
        with(target) {
            pluginManager.apply("myapp.android.library")
            pluginManager.apply("myapp.hilt")

            dependencies {
                // UI fundamentals every feature needs
                "implementation"(project(":core:ui"))
                "implementation"(project(":core:designsystem"))

                // Lifecycle
                "implementation"(libs.findLibrary("androidx-lifecycle-runtimeCompose").get())
                "implementation"(libs.findLibrary("androidx-lifecycle-viewModelCompose").get())

                // Navigation
                "implementation"(libs.findLibrary("androidx-hilt-navigation-compose").get())

                // Hilt
                "implementation"(libs.findLibrary("hilt-android").get())
            }
        }
    }
}
```

### Convention Plugin: `AndroidFeatureApiConventionPlugin.kt`

```kotlin
import org.gradle.api.Plugin
import org.gradle.api.Project

class AndroidFeatureApiConventionPlugin : Plugin<Project> {
    override fun apply(target: Project) {
        with(target) {
            pluginManager.apply("myapp.android.library")
            pluginManager.apply("org.jetbrains.kotlin.plugin.serialization")
        }
    }
}
```

### Convention Plugin: `HiltConventionPlugin.kt`

```kotlin
import com.myapp.buildlogic.libs
import org.gradle.api.Plugin
import org.gradle.api.Project
import org.gradle.kotlin.dsl.dependencies

class HiltConventionPlugin : Plugin<Project> {
    override fun apply(target: Project) {
        with(target) {
            pluginManager.apply("com.google.devtools.ksp")

            dependencies {
                "ksp"(libs.findLibrary("hilt-compiler").get())
            }

            pluginManager.withPlugin("com.android.base") {
                pluginManager.apply("com.google.dagger.hilt.android")
                dependencies {
                    "implementation"(libs.findLibrary("hilt-android").get())
                }
            }
        }
    }
}
```

### Convention Plugin: `AndroidRoomConventionPlugin.kt`

```kotlin
import androidx.room.gradle.RoomExtension
import com.myapp.buildlogic.libs
import org.gradle.api.Plugin
import org.gradle.api.Project
import org.gradle.kotlin.dsl.configure
import org.gradle.kotlin.dsl.dependencies

class AndroidRoomConventionPlugin : Plugin<Project> {
    override fun apply(target: Project) {
        with(target) {
            pluginManager.apply("androidx.room")
            pluginManager.apply("com.google.devtools.ksp")

            extensions.configure<RoomExtension> {
                schemaDirectory("$projectDir/schemas")
            }

            dependencies {
                "implementation"(libs.findLibrary("androidx-room-runtime").get())
                "implementation"(libs.findLibrary("androidx-room-ktx").get())
                "ksp"(libs.findLibrary("androidx-room-compiler").get())
            }
        }
    }
}
```

### Convention Plugin: `JvmLibraryConventionPlugin.kt`

Used for pure Kotlin modules that have no Android dependency (e.g., `:core:model`).

```kotlin
import com.myapp.buildlogic.configureKotlinJvm
import org.gradle.api.Plugin
import org.gradle.api.Project

class JvmLibraryConventionPlugin : Plugin<Project> {
    override fun apply(target: Project) {
        with(target) {
            pluginManager.apply("org.jetbrains.kotlin.jvm")
            configureKotlinJvm()
        }
    }
}
```

### Helper: `ProjectExtensions.kt`

```kotlin
// build-logic/convention/src/main/kotlin/com/myapp/buildlogic/ProjectExtensions.kt
package com.myapp.buildlogic

import org.gradle.api.Project
import org.gradle.api.artifacts.VersionCatalog
import org.gradle.api.artifacts.VersionCatalogsExtension
import org.gradle.kotlin.dsl.getByType

val Project.libs: VersionCatalog
    get() = extensions.getByType<VersionCatalogsExtension>().named("libs")
```

---

## 6. Core Modules

### `:core:model` — Pure data models (no Android framework)

```kotlin
// core/model/build.gradle.kts
plugins {
    alias(libs.plugins.myapp.jvm.library)
}

dependencies {
    api(libs.kotlinx.datetime)
}
```

```kotlin
// core/model/src/main/kotlin/com/myapp/core/model/Track.kt
data class Track(
    val id: String,
    val title: String,
    val artist: String,
    val durationMs: Long,
    val addedAt: Instant,
)
```

### `:core:common` — Shared Kotlin utilities

```kotlin
// core/common/build.gradle.kts
plugins {
    alias(libs.plugins.myapp.jvm.library)
}

dependencies {
    api(libs.kotlinx.coroutines.core)
}
```

```kotlin
// core/common/src/main/kotlin/com/myapp/core/common/network/Dispatcher.kt
import javax.inject.Qualifier

@Qualifier
@Retention(AnnotationRetention.RUNTIME)
annotation class Dispatcher(val niaDispatcher: MyAppDispatchers)

enum class MyAppDispatchers { Default, IO }
```

```kotlin
// core/common/src/main/kotlin/com/myapp/core/common/network/di/DispatchersModule.kt
@Module
@InstallIn(SingletonComponent::class)
object DispatchersModule {
    @Provides
    @Dispatcher(MyAppDispatchers.IO)
    fun providesIODispatcher(): CoroutineDispatcher = Dispatchers.IO

    @Provides
    @Dispatcher(MyAppDispatchers.Default)
    fun providesDefaultDispatcher(): CoroutineDispatcher = Dispatchers.Default
}
```

### `:core:network` — Retrofit clients

```kotlin
// core/network/build.gradle.kts
plugins {
    alias(libs.plugins.myapp.android.library)
    alias(libs.plugins.myapp.hilt)
    alias(libs.plugins.kotlin.serialization)
}

android {
    namespace = "com.myapp.core.network"
    buildFeatures { buildConfig = true }
}

dependencies {
    api(libs.kotlinx.datetime)
    api(projects.core.common)
    api(projects.core.model)

    implementation(libs.kotlinx.serialization.json)
    implementation(libs.okhttp.logging)
    implementation(libs.retrofit.core)
    implementation(libs.retrofit.kotlin.serialization)
}
```

```kotlin
// core/network/src/main/kotlin/com/myapp/core/network/MyAppNetworkDataSource.kt
interface MyAppNetworkDataSource {
    suspend fun getTracks(): List<NetworkTrack>
}
```

```kotlin
// core/network/src/main/kotlin/com/myapp/core/network/retrofit/RetrofitMyAppNetwork.kt
@Singleton
class RetrofitMyAppNetwork @Inject constructor(
    networkJson: Json,
    @Dispatcher(MyAppDispatchers.IO) private val ioDispatcher: CoroutineDispatcher,
) : MyAppNetworkDataSource {

    private val networkApi = Retrofit.Builder()
        .baseUrl(BuildConfig.BACKEND_URL)
        .addConverterFactory(networkJson.asConverterFactory("application/json".toMediaType()))
        .build()
        .create(MyAppRetrofitApi::class.java)

    override suspend fun getTracks(): List<NetworkTrack> =
        withContext(ioDispatcher) { networkApi.getTracks() }
}
```

### `:core:database` — Room database

```kotlin
// core/database/build.gradle.kts
plugins {
    alias(libs.plugins.myapp.android.library)
    alias(libs.plugins.myapp.android.room)
    alias(libs.plugins.myapp.hilt)
}

android {
    namespace = "com.myapp.core.database"
}

dependencies {
    api(projects.core.model)
    implementation(libs.kotlinx.datetime)
}
```

```kotlin
// core/database/src/main/kotlin/com/myapp/core/database/MyAppDatabase.kt
@Database(
    entities = [TrackEntity::class],
    version = 1,
)
abstract class MyAppDatabase : RoomDatabase() {
    abstract fun trackDao(): TrackDao
}
```

```kotlin
// core/database/src/main/kotlin/com/myapp/core/database/dao/TrackDao.kt
@Dao
interface TrackDao {
    @Query("SELECT * FROM tracks ORDER BY added_at DESC")
    fun getTracks(): Flow<List<TrackEntity>>

    @Upsert
    suspend fun upsertTracks(tracks: List<TrackEntity>)

    @Query("DELETE FROM tracks WHERE id IN (:ids)")
    suspend fun deleteTracks(ids: List<String>)
}
```

```kotlin
// core/database/src/main/kotlin/com/myapp/core/database/di/DatabaseModule.kt
@Module
@InstallIn(SingletonComponent::class)
object DatabaseModule {
    @Provides
    @Singleton
    fun providesMyAppDatabase(@ApplicationContext context: Context): MyAppDatabase =
        Room.databaseBuilder(context, MyAppDatabase::class.java, "myapp-database").build()

    @Provides
    fun providesTrackDao(database: MyAppDatabase): TrackDao = database.trackDao()
}
```

### `:core:data` — Repository implementations

```kotlin
// core/data/build.gradle.kts
plugins {
    alias(libs.plugins.myapp.android.library)
    alias(libs.plugins.myapp.hilt)
}

android {
    namespace = "com.myapp.core.data"
}

dependencies {
    api(projects.core.common)
    api(projects.core.database)
    api(projects.core.network)
    api(projects.core.model)

    testImplementation(libs.kotlinx.coroutines.test)
    testImplementation(projects.core.testing)
}
```

```kotlin
// core/data/src/main/kotlin/com/myapp/core/data/repository/TrackRepository.kt
interface TrackRepository {
    fun getTracks(): Flow<List<Track>>
    suspend fun syncWith(synchronizer: Synchronizer): Boolean
}
```

```kotlin
// core/data/src/main/kotlin/com/myapp/core/data/repository/DefaultTrackRepository.kt
@Singleton
class DefaultTrackRepository @Inject constructor(
    private val trackDao: TrackDao,
    private val network: MyAppNetworkDataSource,
    @Dispatcher(MyAppDispatchers.IO) private val ioDispatcher: CoroutineDispatcher,
) : TrackRepository {

    override fun getTracks(): Flow<List<Track>> =
        trackDao.getTracks().map { it.map(TrackEntity::asExternalModel) }

    override suspend fun syncWith(synchronizer: Synchronizer): Boolean =
        withContext(ioDispatcher) {
            val networkTracks = network.getTracks()
            trackDao.upsertTracks(networkTracks.map(NetworkTrack::asEntity))
            true
        }
}
```

```kotlin
// core/data/src/main/kotlin/com/myapp/core/data/di/DataModule.kt
@Module
@InstallIn(SingletonComponent::class)
abstract class DataModule {
    @Binds
    abstract fun bindsTrackRepository(impl: DefaultTrackRepository): TrackRepository
}
```

### `:core:domain` — Use cases

```kotlin
// core/domain/build.gradle.kts
plugins {
    alias(libs.plugins.myapp.android.library)
}

android {
    namespace = "com.myapp.core.domain"
}

dependencies {
    api(projects.core.data)
    api(projects.core.model)

    implementation(libs.javax.inject)

    testImplementation(projects.core.testing)
}
```

```kotlin
// core/domain/src/main/kotlin/com/myapp/core/domain/GetTracksUseCase.kt

// Use cases are plain classes with a single operator fun invoke().
// They are injected with @Inject constructor; no @Singleton — one per ViewModel.
class GetTracksUseCase @Inject constructor(
    private val trackRepository: TrackRepository,
) {
    operator fun invoke(): Flow<List<Track>> =
        trackRepository.getTracks()
}
```

### `:core:designsystem` — Theme, typography, icons

```kotlin
// core/designsystem/build.gradle.kts
plugins {
    alias(libs.plugins.myapp.android.library)
    alias(libs.plugins.myapp.android.library.compose)
}

android {
    namespace = "com.myapp.core.designsystem"
}

dependencies {
    api(libs.androidx.compose.foundation)
    api(libs.androidx.compose.foundation.layout)
    api(libs.androidx.compose.material.iconsExtended)
    api(libs.androidx.compose.material3)
    api(libs.androidx.compose.runtime)
    api(libs.androidx.compose.ui.util)

    implementation(libs.coil.kt.compose)
}
```

```kotlin
// core/designsystem/src/main/kotlin/com/myapp/core/designsystem/theme/Theme.kt
@Composable
fun MyAppTheme(
    darkTheme: Boolean = isSystemInDarkTheme(),
    dynamicColor: Boolean = true,
    content: @Composable () -> Unit,
) {
    val colorScheme = when {
        dynamicColor && Build.VERSION.SDK_INT >= Build.VERSION_CODES.S -> {
            val context = LocalContext.current
            if (darkTheme) dynamicDarkColorScheme(context) else dynamicLightColorScheme(context)
        }
        darkTheme -> DarkColorScheme
        else -> LightColorScheme
    }

    MaterialTheme(
        colorScheme = colorScheme,
        typography = MyAppTypography,
        content = content,
    )
}
```

### `:core:ui` — Reusable Compose components

```kotlin
// core/ui/build.gradle.kts
plugins {
    alias(libs.plugins.myapp.android.library)
    alias(libs.plugins.myapp.android.library.compose)
}

android {
    namespace = "com.myapp.core.ui"
}

dependencies {
    api(projects.core.designsystem)
    api(projects.core.model)

    implementation(libs.coil.kt)
    implementation(libs.coil.kt.compose)
}
```

### `:core:navigation` — Navigation keys and destinations

```kotlin
// core/navigation/build.gradle.kts
plugins {
    alias(libs.plugins.myapp.android.library)
    alias(libs.plugins.kotlin.serialization)
}

android {
    namespace = "com.myapp.core.navigation"
}

dependencies {
    api(libs.androidx.navigation3.runtime)
}
```

### `:core:testing` — Shared test utilities and fakes

```kotlin
// core/testing/build.gradle.kts
plugins {
    alias(libs.plugins.myapp.android.library)
    alias(libs.plugins.myapp.hilt)
}

android {
    namespace = "com.myapp.core.testing"
}

dependencies {
    api(libs.kotlinx.coroutines.test)
    api(projects.core.common)
    api(projects.core.data)
    api(projects.core.model)

    implementation(libs.hilt.android.testing)
    implementation(libs.kotlinx.datetime)
    implementation(libs.turbine)
    implementation(libs.truth)
    implementation(libs.junit)
}
```

```kotlin
// core/testing/src/main/kotlin/com/myapp/core/testing/repository/TestTrackRepository.kt

// Fake repositories expose a MutableStateFlow so tests can inject data.
class TestTrackRepository : TrackRepository {
    private val tracksFlow = MutableStateFlow<List<Track>>(emptyList())

    override fun getTracks(): Flow<List<Track>> = tracksFlow

    fun sendTracks(tracks: List<Track>) {
        tracksFlow.value = tracks
    }

    override suspend fun syncWith(synchronizer: Synchronizer) = true
}
```

---

## 7. Feature Modules

Every feature is two Gradle modules:

| Module | Plugin | Purpose |
|--------|--------|---------|
| `:feature:home:api` | `myapp.android.feature.api` | Public navigation key, no implementation details |
| `:feature:home:impl` | `myapp.android.feature.impl` + compose | Screen, ViewModel, navigation entry provider |

### `:feature:home:api`

```kotlin
// feature/home/api/build.gradle.kts
plugins {
    alias(libs.plugins.myapp.android.feature.api)
}

android {
    namespace = "com.myapp.feature.home.api"
}

dependencies {
    api(projects.core.navigation)
}
```

```kotlin
// feature/home/api/src/main/kotlin/com/myapp/feature/home/api/navigation/HomeNavKey.kt
@Serializable
data object HomeNavKey
```

```xml
<!-- feature/home/api/src/main/AndroidManifest.xml -->
<manifest />
```

### `:feature:home:impl`

```kotlin
// feature/home/impl/build.gradle.kts
plugins {
    alias(libs.plugins.myapp.android.feature.impl)
    alias(libs.plugins.myapp.android.library.compose)
}

android {
    namespace = "com.myapp.feature.home.impl"
    testOptions.unitTests.isIncludeAndroidResources = true
}

dependencies {
    implementation(projects.core.domain)
    implementation(projects.feature.home.api)

    testImplementation(libs.hilt.android.testing)
    testImplementation(libs.robolectric)
    testImplementation(projects.core.testing)

    androidTestImplementation(libs.bundles.androidx.compose.ui.test)
    androidTestImplementation(projects.core.testing)
}
```

**Directory layout:**

```
feature/home/impl/src/
├── main/
│   ├── AndroidManifest.xml           (empty <manifest />)
│   └── kotlin/com/myapp/feature/home/impl/
│       ├── HomeScreen.kt             (Composable — stateless, takes UiState + callbacks)
│       ├── HomeViewModel.kt          (ViewModel — produces UiState StateFlow)
│       ├── HomeUiState.kt            (sealed interface or data class)
│       └── navigation/
│           └── HomeEntryProvider.kt  (registers this feature with the nav graph)
├── test/
│   └── kotlin/com/myapp/feature/home/impl/
│       ├── HomeViewModelTest.kt
│       └── HomeScreenScreenshotTests.kt
└── androidTest/
    └── kotlin/com/myapp/feature/home/impl/
        └── HomeScreenTest.kt
```

---

## 8. App Module

```kotlin
// app/build.gradle.kts
plugins {
    alias(libs.plugins.myapp.android.application)
    alias(libs.plugins.myapp.android.application.compose)
    alias(libs.plugins.myapp.android.application.flavors)
    alias(libs.plugins.myapp.hilt)
    alias(libs.plugins.kotlin.serialization)
}

android {
    namespace = "com.myapp"
    defaultConfig {
        applicationId = "com.myapp"
        versionCode = 1
        versionName = "1.0.0"
        testInstrumentationRunner = "com.myapp.core.testing.MyAppTestRunner"
    }

    packaging {
        resources {
            excludes.add("/META-INF/{AL2.0,LGPL2.1}")
        }
    }
}

dependencies {
    // Features
    implementation(projects.feature.home.api)
    implementation(projects.feature.home.impl)
    implementation(projects.feature.settings.impl)

    // Core
    implementation(projects.core.common)
    implementation(projects.core.data)
    implementation(projects.core.designsystem)
    implementation(projects.core.model)
    implementation(projects.core.ui)

    // AndroidX
    implementation(libs.androidx.activity.compose)
    implementation(libs.androidx.core.ktx)
    implementation(libs.androidx.core.splashscreen)
    implementation(libs.androidx.lifecycle.runtimeCompose)
    implementation(libs.androidx.navigation3.ui)
    implementation(libs.kotlinx.serialization.json)

    testImplementation(libs.kotlin.test)
    testImplementation(projects.core.testing)

    androidTestImplementation(libs.androidx.test.espresso.core)
    androidTestImplementation(libs.androidx.compose.ui.test)
    androidTestImplementation(libs.hilt.android.testing)
    androidTestImplementation(projects.core.testing)
}
```

```xml
<!-- app/src/main/AndroidManifest.xml -->
<?xml version="1.0" encoding="utf-8"?>
<manifest xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:tools="http://schemas.android.com/tools">

    <uses-permission android:name="android.permission.INTERNET" />

    <application
        android:name=".MyApplication"
        android:allowBackup="true"
        android:icon="@mipmap/ic_launcher"
        android:label="@string/app_name"
        android:roundIcon="@mipmap/ic_launcher_round"
        android:supportsRtl="true"
        android:theme="@style/Theme.MyApp.Splash">

        <activity
            android:name=".MainActivity"
            android:exported="true"
            android:windowSoftInputMode="adjustResize">
            <intent-filter>
                <action android:name="android.intent.action.MAIN" />
                <category android:name="android.intent.category.LAUNCHER" />
            </intent-filter>
        </activity>

    </application>
</manifest>
```

```kotlin
// app/src/main/kotlin/com/myapp/MyApplication.kt
@HiltAndroidApp
class MyApplication : Application()
```

```kotlin
// app/src/main/kotlin/com/myapp/MainActivity.kt
@AndroidEntryPoint
class MainActivity : ComponentActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        val splashScreen = installSplashScreen()
        super.onCreate(savedInstanceState)

        enableEdgeToEdge()

        setContent {
            MyAppTheme {
                MyApp()
            }
        }
    }
}
```

```kotlin
// app/src/main/kotlin/com/myapp/ui/MyApp.kt
@Composable
fun MyApp(
    appState: MyAppState = rememberMyAppState(),
) {
    Scaffold(
        bottomBar = {
            MyAppBottomBar(
                destinations = appState.topLevelDestinations,
                currentDestination = appState.currentDestination,
                onNavigateToDestination = appState::navigateToTopLevelDestination,
            )
        }
    ) { padding ->
        NavDisplay(
            backStack = appState.backStack,
            onBack = appState::onBack,
            entryProviders = entryProviders,
            modifier = Modifier.padding(padding),
        )
    }
}
```

---

## 9. Navigation Architecture

Navigation 3 (`androidx.navigation3`) with serializable nav keys.

### Nav Key (in `feature:X:api`)

```kotlin
// feature/home/api/src/main/kotlin/com/myapp/feature/home/api/navigation/HomeNavKey.kt
@Serializable
data object HomeNavKey

// For destinations with arguments:
@Serializable
data class TrackDetailNavKey(val trackId: String)
```

### Entry Provider (in `feature:X:impl`)

```kotlin
// feature/home/impl/src/main/kotlin/com/myapp/feature/home/impl/navigation/HomeEntryProvider.kt
val homeEntryProvider = entryProvider<HomeNavKey> {
    HomeScreen(
        onTrackClick = { trackId ->
            // Navigation handled via lambda — no direct nav dependency in impl
        }
    )
}
```

### Wiring in `:app`

```kotlin
// app/src/main/kotlin/com/myapp/navigation/MyAppNavigation.kt
val entryProviders = entryProviders {
    addEntryProvider(homeEntryProvider)
    addEntryProvider(settingsEntryProvider)
}
```

---

## 10. Compose UI Patterns

### Screen Composable (stateless — receives UiState and callbacks)

```kotlin
// feature/home/impl/src/main/kotlin/com/myapp/feature/home/impl/HomeScreen.kt

// Route: connects ViewModel to the stateless screen
@Composable
internal fun HomeRoute(
    onTrackClick: (String) -> Unit,
    viewModel: HomeViewModel = hiltViewModel(),
) {
    val uiState by viewModel.uiState.collectAsStateWithLifecycle()
    HomeScreen(
        uiState = uiState,
        onTrackClick = onTrackClick,
        onRefresh = viewModel::refresh,
    )
}

// Screen: pure Compose, no ViewModel, fully testable
@Composable
internal fun HomeScreen(
    uiState: HomeUiState,
    onTrackClick: (String) -> Unit,
    onRefresh: () -> Unit,
    modifier: Modifier = Modifier,
) {
    when (uiState) {
        HomeUiState.Loading -> LoadingWheel(modifier = modifier)
        is HomeUiState.Success -> TrackList(
            tracks = uiState.tracks,
            onTrackClick = onTrackClick,
            modifier = modifier,
        )
        is HomeUiState.Error -> ErrorMessage(
            message = uiState.message,
            onRetry = onRefresh,
            modifier = modifier,
        )
    }
}
```

### UiState (sealed interface)

```kotlin
// feature/home/impl/src/main/kotlin/com/myapp/feature/home/impl/HomeUiState.kt
sealed interface HomeUiState {
    data object Loading : HomeUiState
    data class Success(val tracks: List<Track>) : HomeUiState
    data class Error(val message: String) : HomeUiState
}
```

### ViewModel

```kotlin
// feature/home/impl/src/main/kotlin/com/myapp/feature/home/impl/HomeViewModel.kt
@HiltViewModel
class HomeViewModel @Inject constructor(
    private val getTracksUseCase: GetTracksUseCase,
) : ViewModel() {

    val uiState: StateFlow<HomeUiState> = getTracksUseCase()
        .map<List<Track>, HomeUiState>(HomeUiState::Success)
        .catch { emit(HomeUiState.Error(it.message ?: "Unknown error")) }
        .stateIn(
            scope = viewModelScope,
            started = SharingStarted.WhileSubscribed(5_000),
            initialValue = HomeUiState.Loading,
        )

    fun refresh() {
        viewModelScope.launch {
            // trigger refresh
        }
    }
}
```

### Recomposition optimization

```kotlin
// Mark data classes passed across module boundaries as @Stable or @Immutable
// so Compose can skip recomposition when content hasn't changed

@Immutable
data class TrackUiModel(
    val id: String,
    val title: String,
    val artistName: String,
    val durationFormatted: String,
)

// Use derivedStateOf when computing values from other state:
@Composable
fun TrackList(tracks: List<Track>) {
    val sortedTracks by remember(tracks) {
        derivedStateOf { tracks.sortedBy { it.title } }
    }
    LazyColumn {
        items(items = sortedTracks, key = { it.id }) { track ->
            TrackItem(track = track)
        }
    }
}
```

---

## 11. Adding a New Feature

Follow these steps exactly when adding a new feature (e.g., `:feature:playlist`).

### Step 1: Create the api module

```
feature/playlist/
└── api/
    ├── build.gradle.kts
    └── src/main/
        ├── AndroidManifest.xml
        └── kotlin/com/myapp/feature/playlist/api/
            └── navigation/
                └── PlaylistNavKey.kt
```

```kotlin
// feature/playlist/api/build.gradle.kts
plugins {
    alias(libs.plugins.myapp.android.feature.api)
}

android {
    namespace = "com.myapp.feature.playlist.api"
}

dependencies {
    api(projects.core.navigation)
}
```

```kotlin
// PlaylistNavKey.kt
@Serializable
data object PlaylistNavKey
```

### Step 2: Create the impl module

```
feature/playlist/
└── impl/
    ├── build.gradle.kts
    └── src/main/
        ├── AndroidManifest.xml
        └── kotlin/com/myapp/feature/playlist/impl/
            ├── PlaylistScreen.kt
            ├── PlaylistViewModel.kt
            ├── PlaylistUiState.kt
            └── navigation/
                └── PlaylistEntryProvider.kt
```

```kotlin
// feature/playlist/impl/build.gradle.kts
plugins {
    alias(libs.plugins.myapp.android.feature.impl)
    alias(libs.plugins.myapp.android.library.compose)
}

android {
    namespace = "com.myapp.feature.playlist.impl"
    testOptions.unitTests.isIncludeAndroidResources = true
}

dependencies {
    implementation(projects.core.domain)
    implementation(projects.feature.playlist.api)

    testImplementation(libs.hilt.android.testing)
    testImplementation(libs.robolectric)
    testImplementation(projects.core.testing)
}
```

### Step 3: Register in `settings.gradle.kts`

```kotlin
include(":feature:playlist:api")
include(":feature:playlist:impl")
```

### Step 4: Wire into `:app`

```kotlin
// app/build.gradle.kts — add dependencies
implementation(projects.feature.playlist.api)
implementation(projects.feature.playlist.impl)
```

```kotlin
// app/.../navigation/MyAppNavigation.kt — add entry provider
val entryProviders = entryProviders {
    addEntryProvider(homeEntryProvider)
    addEntryProvider(playlistEntryProvider) // ← add this
    addEntryProvider(settingsEntryProvider)
}
```

---

## 12. Adding a New Core Module

Example: adding `:core:analytics`.

### Step 1: Create directory and files

```
core/analytics/
├── build.gradle.kts
└── src/main/
    ├── AndroidManifest.xml
    └── kotlin/com/myapp/core/analytics/
        ├── AnalyticsHelper.kt           (interface)
        ├── NoOpAnalyticsHelper.kt       (demo/test stub)
        └── di/
            └── AnalyticsModule.kt
```

### Step 2: Write the build file

```kotlin
// core/analytics/build.gradle.kts
plugins {
    alias(libs.plugins.myapp.android.library)
    alias(libs.plugins.myapp.hilt)
}

android {
    namespace = "com.myapp.core.analytics"
}

// If it has Compose components:
// alias(libs.plugins.myapp.android.library.compose)
```

### Step 3: Register in `settings.gradle.kts`

```kotlin
include(":core:analytics")
```

### Step 4: Use in other modules

```kotlin
// In any module's build.gradle.kts that needs analytics:
implementation(projects.core.analytics)
// or:
api(projects.core.analytics)  // if consumers of this module also need it
```

### `api` vs `implementation` rule

| Keyword | When to use |
|---------|-------------|
| `api` | The module's public API exposes this dependency's types in its own public interfaces or return types |
| `implementation` | The dependency is only used internally; consumers don't need to know about it |
| `testImplementation` | Only needed in `src/test/` |
| `androidTestImplementation` | Only needed in `src/androidTest/` |

---

## 13. Testing Patterns

### Unit test (ViewModel)

```kotlin
// feature/home/impl/src/test/kotlin/.../HomeViewModelTest.kt
@HiltAndroidTest
class HomeViewModelTest {

    private val testRepository = TestTrackRepository()

    @Test
    fun `uiState emits Success when tracks are available`() = runTest {
        val viewModel = HomeViewModel(
            getTracksUseCase = GetTracksUseCase(testRepository),
        )

        testRepository.sendTracks(listOf(testTrack()))

        viewModel.uiState.test {
            assertIs<HomeUiState.Loading>(awaitItem())
            assertIs<HomeUiState.Success>(awaitItem())
            cancelAndIgnoreRemainingEvents()
        }
    }
}
```

### Compose UI test

```kotlin
// feature/home/impl/src/androidTest/kotlin/.../HomeScreenTest.kt
@HiltAndroidTest
class HomeScreenTest {

    @get:Rule
    val composeTestRule = createAndroidComposeRule<ComponentActivity>()

    @Test
    fun loading_state_shows_wheel() {
        composeTestRule.setContent {
            HomeScreen(
                uiState = HomeUiState.Loading,
                onTrackClick = {},
                onRefresh = {},
            )
        }
        composeTestRule.onNodeWithTag("loading_wheel").assertIsDisplayed()
    }
}
```

### Screenshot test (Roborazzi)

```kotlin
// feature/home/impl/src/test/kotlin/.../HomeScreenScreenshotTests.kt
@RunWith(ParameterizedRobolectricTestRunner::class)
@GraphicsMode(GraphicsMode.Mode.NATIVE)
class HomeScreenScreenshotTests(private val screenshotParameters: DefaultTestDevices) {

    @get:Rule
    val composeTestRule = createComposeRule()

    @Test
    fun homeScreen_success_state() {
        composeTestRule.setContent {
            MyAppTheme {
                HomeScreen(
                    uiState = HomeUiState.Success(tracks = listOf(testTrack())),
                    onTrackClick = {},
                    onRefresh = {},
                )
            }
        }
        composeTestRule.onRoot().captureRoboImage()
    }
}
```

### No mocking rule

Never use `mockk {}` or `Mockito.mock()` for repository or data source layers. Always use a `TestXxx` fake that implements the real interface with a `MutableStateFlow`.

```kotlin
// WRONG — do not do this:
val repo = mockk<TrackRepository>()
every { repo.getTracks() } returns flowOf(emptyList())

// CORRECT — use a fake:
val repo = TestTrackRepository()
repo.sendTracks(emptyList())
```

---

## 14. Code Style & Naming Rules

### Module namespace pattern

```
com.<company>.<app_name>.core.<module_name>
com.<company>.<app_name>.feature.<feature_name>.api
com.<company>.<app_name>.feature.<feature_name>.impl
```

Example for an app named "Pulse" by company "Acme":
- `:core:data` → `com.acme.pulse.core.data`
- `:feature:home:impl` → `com.acme.pulse.feature.home.impl`

### File naming

| Type | Name pattern | Example |
|------|-------------|---------|
| Screen composable | `{Feature}Screen.kt` | `HomeScreen.kt` |
| ViewModel | `{Feature}ViewModel.kt` | `HomeViewModel.kt` |
| UiState | `{Feature}UiState.kt` | `HomeUiState.kt` |
| Nav key | `{Feature}NavKey.kt` | `HomeNavKey.kt` |
| Entry provider | `{Feature}EntryProvider.kt` | `HomeEntryProvider.kt` |
| Repository interface | `{Entity}Repository.kt` | `TrackRepository.kt` |
| Repository impl | `Default{Entity}Repository.kt` | `DefaultTrackRepository.kt` |
| Use case | `Get{Entity}UseCase.kt` / `{Verb}{Entity}UseCase.kt` | `GetTracksUseCase.kt` |
| Room DAO | `{Entity}Dao.kt` | `TrackDao.kt` |
| Room entity | `{Entity}Entity.kt` | `TrackEntity.kt` |
| Retrofit API | `{App}RetrofitApi.kt` | `MyAppRetrofitApi.kt` |
| Network DTO | `Network{Entity}.kt` | `NetworkTrack.kt` |
| Mapper | Extension fun on the source type | `fun NetworkTrack.asEntity()` |
| Hilt module | `{Purpose}Module.kt` | `DatabaseModule.kt`, `DataModule.kt` |
| Fake for tests | `Test{Interface}.kt` | `TestTrackRepository.kt` |

### Kotlin idioms

```kotlin
// Data holders → data class
data class Track(val id: String, val title: String)

// State machines → sealed interface
sealed interface HomeUiState {
    data object Loading : HomeUiState
    data class Success(val tracks: List<Track>) : HomeUiState
}

// Booleans → is/has/can prefix
val isPlaying: Boolean
val hasError: Boolean

// Extension functions over utility classes
fun NetworkTrack.asEntity() = TrackEntity(id = id, title = title)

// Scope functions — judiciously
val client = OkHttpClient.Builder()
    .apply { if (BuildConfig.DEBUG) addInterceptor(HttpLoggingInterceptor()) }
    .build()
```

### Resource naming

Resources are auto-prefixed by the convention plugin using the module path.

| Module | Auto-prefix | Example resource |
|--------|------------|-----------------|
| `:core:designsystem` | `core_designsystem_` | `core_designsystem_ic_check.xml` |
| `:feature:home:impl` | `feature_home_impl_` | `feature_home_impl_strings.xml` |

---

## 15. Checklist for New Projects

Use this checklist when starting a new project or verifying an AI-generated baseline.

### Project setup
- [ ] `build-logic/` is an included build with a `settings.gradle.kts` that shares `libs.versions.toml`
- [ ] `settings.gradle.kts` uses `TYPESAFE_PROJECT_ACCESSORS` feature preview
- [ ] Root `build.gradle.kts` declares all plugins with `apply false`
- [ ] JDK 17 check is present in `settings.gradle.kts`
- [ ] `gradle.properties` has `-Xmx4g`, parallel builds, configuration cache enabled
- [ ] `libs.versions.toml` is the only place versions are defined

### build-logic convention plugins
- [ ] `AndroidApplicationConventionPlugin` — sets compileSdk=36, targetSdk=36, minSdk≥23, Java 11 source/target
- [ ] `AndroidLibraryConventionPlugin` — sets resource prefix from module path automatically
- [ ] `AndroidFeatureImplConventionPlugin` — auto-adds core:ui, core:designsystem, lifecycle, hilt-navigation-compose
- [ ] `AndroidFeatureApiConventionPlugin` — applies serialization plugin
- [ ] `HiltConventionPlugin` — applies KSP + hilt.android conditionally (Android vs JVM)
- [ ] `AndroidRoomConventionPlugin` — configures schema directory + adds room deps
- [ ] `JvmLibraryConventionPlugin` — for pure Kotlin modules with no Android framework
- [ ] `ProjectExtensions.kt` provides `val Project.libs` helper

### Module structure
- [ ] `:core:model` is JVM-only (no Android framework)
- [ ] `:core:common` is JVM-only, provides dispatcher qualifiers
- [ ] `:core:network` has `buildConfig = true` for `BACKEND_URL`
- [ ] `:core:database` uses `myapp.android.room` plugin and has a `schemas/` directory
- [ ] `:core:data` `api()` exposes common, database, network, model; `implementation()` for analytics
- [ ] `:core:domain` uses only `javax.inject`, no Hilt (use cases aren't `@Singleton`)
- [ ] `:core:testing` exports fake repositories; all test modules use it
- [ ] Every feature has separate `:feature:X:api` and `:feature:X:impl` modules
- [ ] Feature api only exposes nav key + `core:navigation`
- [ ] Feature impl `AndroidManifest.xml` is empty `<manifest />`

### Architecture
- [ ] Single `MainActivity` with `@AndroidEntryPoint`
- [ ] `@HiltAndroidApp` on `Application` class
- [ ] Every ViewModel uses `stateIn(viewModelScope, WhileSubscribed(5000), initialValue)`
- [ ] UI state collected with `collectAsStateWithLifecycle()` (not `collectAsState()`)
- [ ] Screen composables are stateless — all state hoisted to `*Route` composable
- [ ] `UiState` is a sealed interface per screen
- [ ] Repository interface in `:core:data`; `Default*` impl bound via `@Binds` in `DataModule`
- [ ] Use cases have single `operator fun invoke()`, no `@Singleton`
- [ ] Dispatchers are injected, never hardcoded

### Testing
- [ ] `TestXxx` fakes in `:core:testing` for every repository
- [ ] No `mockk {}` / `Mockito.mock()` for repository layer
- [ ] ViewModel tests use `runTest` + Turbine `.test {}`
- [ ] Compose tests use `createAndroidComposeRule<ComponentActivity>()`

### Code quality
- [ ] No comments that restate what the code does
- [ ] No hardcoded user-facing strings (all in `res/values/strings.xml`)
- [ ] No hardcoded design tokens (colors, text sizes) — use design system
- [ ] No `GlobalScope` usage
- [ ] No `viewModelScope.launch` without a clear cancellation path
- [ ] Resource files prefixed with module path (handled automatically by convention plugin)
- [ ] Sensitive data (tokens, keys) stored in `EncryptedSharedPreferences` or Keystore
