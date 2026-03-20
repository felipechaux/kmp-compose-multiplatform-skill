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

## R8 / ProGuard Configuration (Android)

Keep rules are required for Ktor, Room, and Koin in release builds. Add to `android/proguard-rules.pro`:

```proguard
# Ktor — keep serialization metadata
-keep class io.ktor.** { *; }
-keep class kotlinx.serialization.** { *; }
-keepattributes *Annotation*, InnerClasses
-dontnote kotlinx.serialization.AnnotationsKt
-keepclassmembers class ** {
    @kotlinx.serialization.Serializable *;
}

# Room — keep generated _Impl classes
-keep class * extends androidx.room.RoomDatabase
-keep @androidx.room.Entity class *
-dontwarn androidx.room.paging.**

# Koin — keep module declarations
-keep class org.koin.** { *; }
-keepnames class * extends org.koin.core.module.Module

# Kotlin reflection (used by Koin)
-keep class kotlin.Metadata { *; }
-keepclassmembers class ** {
    @kotlin.jvm.JvmField *;
    @kotlin.jvm.JvmStatic *;
}

# DataStore
-keep class androidx.datastore.** { *; }

# Coroutines
-keepnames class kotlinx.coroutines.internal.MainDispatcherFactory {}
-keepnames class kotlinx.coroutines.CoroutineExceptionHandler {}
```

Enable R8 full mode for better size reduction (requires more explicit keeps):

```kotlin
// android/build.gradle.kts
android {
    buildTypes {
        release {
            isMinifyEnabled = true
            isShrinkResources = true
            proguardFiles(
                getDefaultProguardFile("proguard-android-optimize.txt"),
                "proguard-rules.pro"
            )
        }
    }
}
```

```properties
# gradle.properties — enable R8 full mode
android.enableR8.fullMode=true
```

---

## Publishing to Maven (Local and Remote)

### Local Maven (for local multi-repo development)

```kotlin
// shared/build.gradle.kts
plugins {
    id("maven-publish")
}

afterEvaluate {
    publishing {
        publications {
            create<MavenPublication>("release") {
                groupId = "com.example"
                artifactId = "shared"
                version = "1.0.0"
                from(components["kotlin"])
            }
        }
        repositories {
            maven {
                name = "LocalMaven"
                url = uri("${rootProject.buildDir}/local-maven")
            }
        }
    }
}
```

```bash
# Publish to local Maven repo
./gradlew :shared:publishReleasePublicationToLocalMavenRepository

# Consume from local Maven in another project
repositories {
    maven { url = uri("/path/to/local-maven") }
}
```

### Publishing to GitHub Packages (CI)

```kotlin
// shared/build.gradle.kts
publishing {
    repositories {
        maven {
            name = "GitHubPackages"
            url = uri("https://maven.pkg.github.com/ORG/REPO")
            credentials {
                username = System.getenv("GITHUB_ACTOR")
                password = System.getenv("GITHUB_TOKEN")
            }
        }
    }
}
```

```yaml
# .github/workflows/publish.yml
- name: Publish to GitHub Packages
  run: ./gradlew :shared:publish
  env:
    GITHUB_ACTOR: ${{ github.actor }}
    GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
```

---

## CI Gradle Daemon Configuration

Disable the Gradle daemon in CI to avoid warm-up overhead and avoid orphaned daemon processes:

```properties
# gradle.properties (CI override via environment or -P flag)
# In CI, set via: ./gradlew -Dorg.gradle.daemon=false
org.gradle.daemon=false
```

Or set in CI workflow:

```yaml
# GitHub Actions — disable daemon and configure memory for CI
env:
  GRADLE_OPTS: "-Dorg.gradle.daemon=false -Dkotlin.incremental=false -Dorg.gradle.jvmargs=-Xmx4g"
```

Use Gradle's built-in caching in GitHub Actions:

```yaml
- name: Setup Gradle
  uses: gradle/actions/setup-gradle@v4
  with:
    cache-encryption-key: ${{ secrets.GRADLE_ENCRYPTION_KEY }}

- name: Build
  run: ./gradlew :shared:assembleRelease
```

This caches the Gradle home directory, build cache, and configuration cache between runs — significantly speeds up CI.

---

## Build Performance Tips

1. **Enable build cache** — `org.gradle.caching=true` in `gradle.properties`
2. **Enable configuration cache** — `org.gradle.configuration-cache=true`
3. **Parallel builds** — `org.gradle.parallel=true`
4. **Avoid `implementation` in `api`** — Use `api()` only for types exposed in public signatures
5. **Use `compileOnly`** for annotation processors that don't need to be on runtime classpath
6. **Prefer `testImplementation`** over `implementation` for test-only deps
7. **Disable daemon in CI** — `org.gradle.daemon=false` via `GRADLE_OPTS` environment variable
8. **Use `--no-configuration-cache` selectively** — some third-party plugins are not configuration-cache compatible yet; add them to the incompatible task list rather than disabling globally

Check build health with:
```bash
./gradlew buildHealth          # dependency analysis
./gradlew :shared:dependencies # inspect dependency tree
./gradlew --profile            # generate HTML build performance report
```
