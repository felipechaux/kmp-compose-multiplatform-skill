# Convention Plugins (build-logic)

Based on [Now in Android build-logic](https://github.com/android/nowinandroid/tree/main/build-logic) and [Gradle convention plugins best practices](https://docs.gradle.org/current/samples/sample_convention_plugins.html).

## Overview

Convention plugins centralize shared Gradle configuration. Each plugin has a single responsibility and modules combine them as needed.

## Directory Structure

```
build-logic/
└── convention/
    ├── build.gradle.kts
    └── src/main/kotlin/
        ├── KmpLibraryPlugin.kt       # KMP library base config
        ├── KmpComposablePlugin.kt    # Adds Compose Multiplatform
        ├── AndroidLibraryPlugin.kt   # Android library config
        ├── AndroidApplicationPlugin.kt
        └── KoinPlugin.kt             # Koin dependency setup
```

## settings.gradle.kts (root)

```kotlin
pluginManagement {
    includeBuild("build-logic")
    repositories {
        google()
        mavenCentral()
        gradlePluginPortal()
    }
}
```

## KmpLibraryPlugin.kt

```kotlin
class KmpLibraryPlugin : Plugin<Project> {
    override fun apply(target: Project) = with(target) {
        with(pluginManager) {
            apply("org.jetbrains.kotlin.multiplatform")
            apply("com.android.library")
        }

        extensions.configure<KotlinMultiplatformExtension> {
            androidTarget {
                compilations.all {
                    compileTaskProvider.configure {
                        compilerOptions { jvmTarget.set(JvmTarget.JVM_17) }
                    }
                }
            }

            listOf(iosX64(), iosArm64(), iosSimulatorArm64()).forEach {
                it.binaries.framework { baseName = project.name; isStatic = true }
            }
        }

        extensions.configure<LibraryExtension> {
            compileSdk = 35
            defaultConfig.minSdk = 24
            compileOptions {
                sourceCompatibility = JavaVersion.VERSION_17
                targetCompatibility = JavaVersion.VERSION_17
            }
        }
    }
}
```

## KmpComposablePlugin.kt

```kotlin
class KmpComposablePlugin : Plugin<Project> {
    override fun apply(target: Project) = with(target) {
        pluginManager.apply("org.jetbrains.compose")
        pluginManager.apply("org.jetbrains.kotlin.plugin.compose")

        extensions.configure<KotlinMultiplatformExtension> {
            sourceSets {
                commonMain.dependencies {
                    implementation(compose.runtime)
                    implementation(compose.foundation)
                    implementation(compose.material3)
                    implementation(compose.ui)
                    implementation(compose.components.resources)
                }
            }
        }
    }
}
```

## Module build.gradle.kts (using plugins)

```kotlin
plugins {
    id("convention.kmp-library")      // base KMP + Android
    id("convention.kmp-composable")   // adds Compose
    id("convention.koin")             // adds Koin
    alias(libs.plugins.ksp)
    alias(libs.plugins.room)
    alias(libs.plugins.kotlin.serialization)
}

kotlin {
    sourceSets {
        commonMain.dependencies {
            // Feature-specific deps only
            implementation(libs.ktor.client.core)
        }
    }
}

room {
    schemaDirectory("$projectDir/schemas")
}
```
