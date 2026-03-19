# Build System — KMP + Compose Multiplatform

References: [Convention Plugins](https://developer.android.com/topic/modularization/build-logic) | [Version Catalog](https://docs.gradle.org/current/userguide/platforms.html) | [KSP](https://kotlinlang.org/docs/ksp-overview.html)

---

## gradle.properties

Essential settings for KMP builds:

```properties
# gradle.properties

# Kotlin
kotlin.code.style=official
kotlin.mpp.androidSourceSetLayoutVersion=2

# Android
android.useAndroidX=true
android.nonTransitiveRClass=true

# Build performance
org.gradle.jvmargs=-Xmx4g -XX:+UseParallelGC
org.gradle.parallel=true
org.gradle.caching=true
org.gradle.configuration-cache=true

# Compose
org.jetbrains.compose.experimental.uikit.enabled=true
```

---

## Gradle Wrapper

Always pin the Gradle version in `gradle/wrapper/gradle-wrapper.properties`:

```properties
distributionUrl=https\://services.gradle.org/distributions/gradle-8.10-bin.zip
```

Use Gradle 8.10+ with AGP 8.8+ and Kotlin 2.x.

---

## Convention Plugins (build-logic)

For multi-module projects, extract all build logic into convention plugins to keep module `build.gradle.kts` files minimal and consistent.

### build-logic structure

```
build-logic/
└── convention/
    ├── build.gradle.kts
    └── src/main/kotlin/
        ├── KmpLibraryPlugin.kt
        ├── KmpLibraryComposePlugin.kt
        ├── AndroidApplicationPlugin.kt
        └── KoinPlugin.kt
```

### build-logic/convention/build.gradle.kts

```kotlin
plugins {
    `kotlin-dsl`
}

dependencies {
    compileOnly(libs.android.gradlePlugin)
    compileOnly(libs.kotlin.gradlePlugin)
    compileOnly(libs.compose.gradlePlugin)
    compileOnly(libs.ksp.gradlePlugin)
}

// Register all convention plugins
gradlePlugin {
    plugins {
        register("kmpLibrary") {
            id = "convention.kmp.library"
            implementationClass = "KmpLibraryPlugin"
        }
        register("kmpLibraryCompose") {
            id = "convention.kmp.library.compose"
            implementationClass = "KmpLibraryComposePlugin"
        }
        register("androidApplication") {
            id = "convention.android.application"
            implementationClass = "AndroidApplicationPlugin"
        }
        register("koin") {
            id = "convention.koin"
            implementationClass = "KoinPlugin"
        }
    }
}
```

### KmpLibraryPlugin.kt

```kotlin
import com.android.build.gradle.LibraryExtension
import org.gradle.api.Plugin
import org.gradle.api.Project
import org.gradle.kotlin.dsl.configure
import org.gradle.kotlin.dsl.getByType
import org.jetbrains.kotlin.gradle.dsl.KotlinMultiplatformExtension

class KmpLibraryPlugin : Plugin<Project> {
    override fun apply(target: Project) {
        with(target) {
            with(pluginManager) {
                apply("org.jetbrains.kotlin.multiplatform")
                apply("com.android.library")
            }

            extensions.configure<LibraryExtension> {
                compileSdk = 35
                defaultConfig {
                    minSdk = 26
                }
                compileOptions {
                    sourceCompatibility = JavaVersion.VERSION_17
                    targetCompatibility = JavaVersion.VERSION_17
                }
            }

            extensions.configure<KotlinMultiplatformExtension> {
                androidTarget {
                    compilations.all {
                        compileTaskProvider.configure {
                            compilerOptions {
                                jvmTarget.set(org.jetbrains.kotlin.gradle.dsl.JvmTarget.JVM_17)
                            }
                        }
                    }
                }

                listOf(iosX64(), iosArm64(), iosSimulatorArm64()).forEach { target ->
                    target.binaries.framework {
                        baseName = project.name
                        isStatic = true
                    }
                }

                sourceSets.commonMain.dependencies {
                    implementation(libs.findLibrary("kotlinx-coroutines-core").get())
                }
            }
        }
    }
}
```

### KmpLibraryComposePlugin.kt

```kotlin
import org.gradle.api.Plugin
import org.gradle.api.Project
import org.gradle.kotlin.dsl.configure
import org.jetbrains.compose.ComposeExtension
import org.jetbrains.kotlin.gradle.dsl.KotlinMultiplatformExtension

class KmpLibraryComposePlugin : Plugin<Project> {
    override fun apply(target: Project) {
        with(target) {
            pluginManager.apply("convention.kmp.library")
            pluginManager.apply("org.jetbrains.compose")
            pluginManager.apply("org.jetbrains.kotlin.plugin.compose")

            extensions.configure<KotlinMultiplatformExtension> {
                sourceSets.commonMain.dependencies {
                    val compose = extensions.getByType<ComposeExtension>().dependencies
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

### KoinPlugin.kt

```kotlin
import org.gradle.api.Plugin
import org.gradle.api.Project
import org.gradle.kotlin.dsl.configure
import org.jetbrains.kotlin.gradle.dsl.KotlinMultiplatformExtension

class KoinPlugin : Plugin<Project> {
    override fun apply(target: Project) {
        with(target) {
            extensions.configure<KotlinMultiplatformExtension> {
                sourceSets.commonMain.dependencies {
                    implementation(libs.findLibrary("koin-core").get())
                    implementation(libs.findLibrary("koin-compose").get())
                    implementation(libs.findLibrary("koin-compose-viewmodel").get())
                }
                sourceSets.named("androidMain") {
                    dependencies {
                        implementation(libs.findLibrary("koin-android").get())
                    }
                }
            }
        }
    }
}
```

### Using Convention Plugins in a Feature Module

```kotlin
// feature/home/build.gradle.kts
plugins {
    alias(libs.plugins.convention.kmp.library.compose)
    alias(libs.plugins.convention.koin)
    alias(libs.plugins.kotlin.serialization)
}

kotlin {
    sourceSets {
        commonMain.dependencies {
            implementation(projects.core.domain)
            implementation(projects.core.data)
            implementation(libs.navigation.compose)
        }
    }
}
```

---

## KSP Configuration for Room

```kotlin
// shared/build.gradle.kts
plugins {
    alias(libs.plugins.ksp)
    alias(libs.plugins.room)
}

kotlin {
    sourceSets {
        commonMain.dependencies {
            implementation(libs.room.runtime)
            implementation(libs.room.ktx)
        }
    }
}

// KSP — add Room processor for each platform target
dependencies {
    add("kspAndroid", libs.room.compiler)
    add("kspIosX64", libs.room.compiler)
    add("kspIosArm64", libs.room.compiler)
    add("kspIosSimulatorArm64", libs.room.compiler)
}

// Room — schema output directory for migration validation
room {
    schemaDirectory("$projectDir/schemas")
}
```

Enable incremental processing in `gradle.properties`:

```properties
ksp.incremental=true
```

---

## Version Catalog — Build Plugin Dependencies

Add build plugin deps to the catalog so convention plugins can use `libs`:

```toml
# gradle/libs.versions.toml
[libraries]
android-gradlePlugin = { module = "com.android.tools.build:gradle", version.ref = "agp" }
kotlin-gradlePlugin = { module = "org.jetbrains.kotlin:kotlin-gradle-plugin", version.ref = "kotlin" }
compose-gradlePlugin = { module = "org.jetbrains.compose:compose-gradle-plugin", version.ref = "compose-multiplatform" }
ksp-gradlePlugin = { module = "com.google.devtools.ksp:symbol-processing-gradle-plugin", version.ref = "ksp" }

[plugins]
convention-kmp-library = { id = "convention.kmp.library", version = "unspecified" }
convention-kmp-library-compose = { id = "convention.kmp.library.compose", version = "unspecified" }
convention-android-application = { id = "convention.android.application", version = "unspecified" }
convention-koin = { id = "convention.koin", version = "unspecified" }
```

---

## Settings.gradle.kts (Multi-Module)

```kotlin
// settings.gradle.kts
pluginManagement {
    includeBuild("build-logic")   // include convention plugins
    repositories {
        google()
        mavenCentral()
        gradlePluginPortal()
    }
}

dependencyResolutionManagement {
    repositoriesMode.set(RepositoriesMode.FAIL_ON_PROJECT_REPOS)
    repositories {
        google()
        mavenCentral()
    }
}

rootProject.name = "MyApp"

// Feature modules
include(":app")
include(":iosApp")
include(":shared")
include(":core:domain")
include(":core:data")
include(":core:database")
include(":core:network")
include(":core:designsystem")
include(":feature:home:api")
include(":feature:home:impl")
include(":feature:auth:api")
include(":feature:auth:impl")
```

---

## BuildKonfig — Full Configuration

```kotlin
// shared/build.gradle.kts
buildkonfig {
    packageName = "com.example.shared"

    defaultConfigs {
        buildConfigField(STRING, "ENVIRONMENT", "staging")
        buildConfigField(BOOLEAN, "IS_DEBUG", "true")
        buildConfigField(BOOLEAN, "ENABLE_LOGGING", "true")
        buildConfigField(STRING, "API_BASE_URL", "https://api.staging.example.com")
        buildConfigField(STRING, "APP_VERSION", project.version.toString())
    }

    // Production overrides
    targetConfigs("prod") {
        buildConfigField(STRING, "ENVIRONMENT", "production")
        buildConfigField(BOOLEAN, "IS_DEBUG", "false")
        buildConfigField(BOOLEAN, "ENABLE_LOGGING", "false")
        buildConfigField(STRING, "API_BASE_URL", "https://api.example.com")
    }
}
```

Access from Swift:

```swift
import shared

let apiUrl = BuildKonfig.shared.API_BASE_URL
let isDebug = BuildKonfig.shared.IS_DEBUG
```

Access from Kotlin:

```kotlin
val apiUrl = BuildKonfig.API_BASE_URL
val isDebug = BuildKonfig.IS_DEBUG
```

---

## Build Performance Tips

1. **Enable build cache** — `org.gradle.caching=true` in `gradle.properties`
2. **Enable configuration cache** — `org.gradle.configuration-cache=true`
3. **Parallel builds** — `org.gradle.parallel=true`
4. **Avoid `implementation` in `api`** — Use `api()` only for types exposed in public signatures
5. **Use `compileOnly`** for annotation processors that don't need to be on runtime classpath
6. **Prefer `testImplementation`** over `implementation` for test-only deps

Check build health with:
```bash
./gradlew buildHealth          # dependency analysis
./gradlew :shared:dependencies # inspect dependency tree
```
