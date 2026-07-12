# Convention Plugins

Convention plugins are how you standardize 30+ modules without copy-pasting build config into every `build.gradle.kts`. This is a strong senior signal: it shows you treat build logic as production code, understand Gradle's configuration model, and care about build-time feedback loops. NowInAndroid made this the reference pattern.

## The problem they solve

Without them, every module repeats the same block:

```kotlin
// Repeated in 30+ build.gradle.kts files — a maintenance disaster.
android {
    compileSdk = 35
    defaultConfig { minSdk = 24 }
    compileOptions {
        sourceCompatibility = JavaVersion.VERSION_17
        targetCompatibility = JavaVersion.VERSION_17
    }
    buildFeatures { compose = true }
}
kotlin { jvmToolchain(17) }
dependencies {
    implementation(platform(libs.androidx.compose.bom))
    // ...hilt, compose, test wiring...
}
```

Bump `compileSdk`? Edit 30 files. Add a lint baseline? 30 files. Convention plugins collapse all of it to a single applied plugin id per module.

## Why `build-logic` beats `buildSrc`

Both let you write reusable Gradle logic in Kotlin. The difference is **build invalidation**, and it's the whole reason to prefer `build-logic`.

| | `buildSrc` | `build-logic` (included build) |
|---|---|---|
| Change invalidation | Any edit invalidates the **entire build's** script-compilation cache and can force reconfiguration of every module | Only modules that **apply the changed plugin** reconfigure |
| Configuration cache | Plays poorly; frequent cache misses | First-class; cache survives unrelated build-logic changes |
| Coupling | Implicitly on every project's classpath | Explicit — you `apply` what you need |
| Publishing | Not publishable | Plugins are real, versionable, publishable artifacts |

!!! warning "The one-liner that lands the point"
    "`buildSrc` is an implicit classpath dependency of the whole build, so touching one helper recompiles all build logic and busts caches. `build-logic` is an included build exposing discrete plugins, so a change only reconfigures modules that apply that plugin. On a 30-module project that's the difference between a 5-second and a 90-second feedback loop after a build-logic edit."

## Structuring `build-logic/convention`

```
build-logic/
├── settings.gradle.kts          # standalone: declares version catalog for build-logic
└── convention/
    ├── build.gradle.kts         # registers the plugins
    └── src/main/kotlin/
        ├── AndroidApplicationConventionPlugin.kt
        ├── AndroidLibraryConventionPlugin.kt
        ├── AndroidFeatureConventionPlugin.kt
        ├── AndroidHiltConventionPlugin.kt
        ├── AndroidComposeConventionPlugin.kt
        └── KotlinAndroid.kt      # shared helper, not a plugin
```

Wire it into the root `settings.gradle.kts`:

```kotlin
pluginManagement {
    includeBuild("build-logic")   // makes the convention plugins available
    repositories { google(); mavenCentral(); gradlePluginPortal() }
}
```

The `build-logic/settings.gradle.kts` re-declares the same version catalog so plugins can read `libs.*`:

```kotlin
dependencyResolutionManagement {
    versionCatalogs {
        create("libs") { from(files("../gradle/libs.versions.toml")) }
    }
}
```

## Registering plugins

In `build-logic/convention/build.gradle.kts`, register each `Plugin<Project>` class with an id. Those ids are what modules apply.

```kotlin
plugins {
    `kotlin-dsl`
}

dependencies {
    // Compile-only so the AGP/Kotlin versions used at BUILD time match the app's.
    compileOnly(libs.android.gradlePlugin)
    compileOnly(libs.kotlin.gradlePlugin)
    compileOnly(libs.ksp.gradlePlugin)
}

gradlePlugin {
    plugins {
        register("androidApplication") {
            id = "myapp.android.application"
            implementationClass = "AndroidApplicationConventionPlugin"
        }
        register("androidLibrary") {
            id = "myapp.android.library"
            implementationClass = "AndroidLibraryConventionPlugin"
        }
        register("androidFeature") {
            id = "myapp.android.feature"
            implementationClass = "AndroidFeatureConventionPlugin"
        }
        register("androidHilt") {
            id = "myapp.android.hilt"
            implementationClass = "AndroidHiltConventionPlugin"
        }
        register("androidCompose") {
            id = "myapp.android.compose"
            implementationClass = "AndroidComposeConventionPlugin"
        }
    }
}
```

Apply them in a module — no `apply plugin`, no version numbers, just intent:

```kotlin
plugins {
    id("myapp.android.feature")
    id("myapp.android.compose")
}
```

## A real `AndroidFeatureConventionPlugin.kt`

A feature module is just an Android library that also gets Compose, Hilt, and the standard feature dependencies. So the feature plugin **composes** other conventions rather than duplicating them.

```kotlin
import com.android.build.gradle.LibraryExtension
import org.gradle.api.Plugin
import org.gradle.api.Project
import org.gradle.api.artifacts.VersionCatalogsExtension
import org.gradle.kotlin.dsl.dependencies
import org.gradle.kotlin.dsl.getByType

class AndroidFeatureConventionPlugin : Plugin<Project> {
    override fun apply(target: Project) = with(target) {
        // Compose one convention out of others — do NOT re-declare their config.
        pluginManager.apply("myapp.android.library")
        pluginManager.apply("myapp.android.hilt")
        pluginManager.apply("myapp.android.compose")

        // Read the shared version catalog from inside the plugin.
        val libs = extensions.getByType<VersionCatalogsExtension>().named("libs")

        extensions.configure<LibraryExtension> {
            defaultConfig {
                testInstrumentationRunner =
                    "androidx.test.runner.AndroidJUnitRunner"
            }
        }

        dependencies {
            // Every feature gets these for free — the whole point.
            add("implementation", project(":core:ui"))
            add("implementation", project(":core:designsystem"))
            add("implementation", project(":domain"))

            add("implementation", libs.findLibrary("androidx.hilt.navigation.compose").get())
            add("implementation", libs.findLibrary("androidx.lifecycle.viewModelCompose").get())

            add("androidTestImplementation", project(":core:testing"))
        }
    }
}
```

## The `library` convention + a shared `android {}` helper

Keep the actual `android {}` configuration in a plain function so both the application and library plugins call it. This is the single place `compileSdk`, `minSdk`, and Java/Kotlin targets live.

```kotlin
// AndroidLibraryConventionPlugin.kt
import com.android.build.gradle.LibraryExtension
import org.gradle.api.Plugin
import org.gradle.api.Project
import org.gradle.kotlin.dsl.configure

class AndroidLibraryConventionPlugin : Plugin<Project> {
    override fun apply(target: Project) = with(target) {
        pluginManager.apply("com.android.library")
        pluginManager.apply("org.jetbrains.kotlin.android")

        extensions.configure<LibraryExtension> {
            configureKotlinAndroid(this)      // shared helper below
            defaultConfig.targetSdk = 35
        }
    }
}
```

```kotlin
// KotlinAndroid.kt  — NOT a plugin, just a shared function.
import com.android.build.api.dsl.CommonExtension
import org.gradle.api.JavaVersion
import org.gradle.api.Project
import org.gradle.api.artifacts.VersionCatalogsExtension
import org.gradle.kotlin.dsl.getByType
import org.jetbrains.kotlin.gradle.dsl.KotlinAndroidProjectExtension

internal fun Project.configureKotlinAndroid(
    commonExtension: CommonExtension<*, *, *, *, *, *>,
) {
    val libs = extensions.getByType<VersionCatalogsExtension>().named("libs")

    commonExtension.apply {
        compileSdk = 35
        defaultConfig { minSdk = 24 }
        compileOptions {
            sourceCompatibility = JavaVersion.VERSION_17
            targetCompatibility = JavaVersion.VERSION_17
        }
    }

    extensions.configure<KotlinAndroidProjectExtension> {
        compilerOptions {
            // e.g. jvmTarget, allWarningsAsErrors, opt-ins — set once, everywhere.
        }
    }
}
```

## The Hilt convention plugin

```kotlin
// AndroidHiltConventionPlugin.kt
import org.gradle.api.Plugin
import org.gradle.api.Project
import org.gradle.api.artifacts.VersionCatalogsExtension
import org.gradle.kotlin.dsl.dependencies
import org.gradle.kotlin.dsl.getByType

class AndroidHiltConventionPlugin : Plugin<Project> {
    override fun apply(target: Project) = with(target) {
        val libs = extensions.getByType<VersionCatalogsExtension>().named("libs")

        pluginManager.apply("com.google.devtools.ksp")
        pluginManager.apply("com.google.dagger.hilt.android")

        dependencies {
            add("implementation", libs.findLibrary("hilt.android").get())
            add("ksp", libs.findLibrary("hilt.compiler").get())
        }
    }
}
```

!!! tip "Reading the version catalog from a plugin"
    Inside a `Plugin<Project>` you can't use the generated `libs.retrofit` accessor — that's only available in `.gradle.kts` scripts. Use the `VersionCatalogsExtension` and `libs.findLibrary("name").get()` / `libs.findPlugin("name").get()` / `libs.findVersion("name").get()` API, with the catalog name (e.g. `retrofit`) as the dotted string.

## What this buys you across 30+ modules

- **Single source of truth**: bump `compileSdk` once in `KotlinAndroid.kt`, not in 30 files.
- **Consistency by construction**: it's impossible for a module to forget Hilt's KSP processor or Compose's compiler — applying `myapp.android.feature` guarantees them.
- **Readable build files**: a module's `build.gradle.kts` states *what it is* (`id("myapp.android.feature")`) and *what it needs* (its `dependencies`), nothing else.
- **Fast feedback**: editing a convention only reconfigures modules that apply it, and the config cache survives unrelated build-logic edits.
- **Onboarding**: a new engineer creating a module copies one `plugins {}` line, not 40 lines of tribal knowledge.

!!! example "The interview summary"
    "I model build logic as first-class code in a `build-logic` included build. Each convention is a `Plugin<Project>` registered with an id via `gradlePlugin { plugins { register(...) } }`, and modules opt in with `plugins { id(\"myapp.android.feature\") }`. Feature plugins compose library + compose + hilt conventions so there's zero duplication. It reads the same `libs.versions.toml` the app uses, so versions never drift. That's how you keep 30+ modules identical and builds fast."
