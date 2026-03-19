<div align="center">

<img src="https://avatars.githubusercontent.com/u/878437?s=120&v=4" alt="JetBrains" width="80" />

# KMP + Compose Multiplatform Agent Skill

**Make your AI coding tool actually understand Kotlin-Compose Multiplatform.**
Backed by real JetBrains and Google architecture docs — not vibes.

![Setup time](https://img.shields.io/badge/setup-5%20min-brightgreen)
![License](https://img.shields.io/badge/license-MIT-blue)
![Compose Multiplatform](https://img.shields.io/badge/Compose%20Multiplatform-1.8.0%2B-7F52FF?logo=jetbrains)
![Kotlin](https://img.shields.io/badge/Kotlin-2.1.0%2B-orange?logo=kotlin)

</div>

---

A **Claude Code skill** that turns any Claude agent into an expert in **Kotlin Multiplatform (KMP)** and **Compose Multiplatform** development — following Google's official architecture guidelines and JetBrains best practices.

Without this skill, Claude gives you generic Android advice. With it, Claude understands shared `commonMain` source sets, `expect`/`actual` declarations, Koin multiplatform DI, Room KMP setup, Ktor client configuration, and the full Clean Architecture layer separation that makes KMP projects maintainable across Android and iOS.

## What This Skill Covers

| Area | Details |
|------|---------|
| **Architecture** | Clean Architecture with feature-based modularization (Now in Android pattern) |
| **UI** | Compose Multiplatform, Material 3, state hoisting, previews |
| **Navigation** | Type-safe sealed class routes, NavHostController |
| **DI** | Koin Multiplatform — modules, ViewModels, platform DI |
| **Networking** | Ktor client — setup, interceptors, error handling |
| **Persistence** | Room KMP + DataStore KMP — expect/actual setup |
| **State** | UDF pattern, StateFlow, Resource sealed class |
| **Build System** | Version catalog, BuildKonfig, KSP, convention plugins |
| **iOS** | SPM integration, ComposeUIViewController, expect/actual |
| **Testing** | Fakes over mocks, commonTest, coroutine testing |

## Installation

### Option 1: Project-level (recommended for KMP projects)

```bash
git clone https://github.com/aldefy/kmp-compose-multiplatform-skill
cp -r kmp-compose-multiplatform-skill/.claude/skills/kmp-compose-multiplatform .claude/skills/
```

### Option 2: Global (available in all projects)

```bash
git clone https://github.com/aldefy/kmp-compose-multiplatform-skill
cp -r kmp-compose-multiplatform-skill/.claude/skills/kmp-compose-multiplatform ~/.claude/skills/
```

### Option 3: Git submodule

```bash
git submodule add https://github.com/aldefy/kmp-compose-multiplatform-skill .claude/kmp-skill
cp -r .claude/kmp-skill/.claude/skills/kmp-compose-multiplatform .claude/skills/
```

> **That's it.** Claude Code automatically picks up skill directories from `.claude/skills/`. No configuration required.

## Usage

Once installed, invoke the skill in Claude Code:

```
/kmp-compose-multiplatform
```

Or reference it inline:

```
Using the kmp-compose-multiplatform skill, create a new feature module for user authentication
```

```
Using the kmp-compose-multiplatform skill, review my ViewModel for UDF violations
```

```
Using the kmp-compose-multiplatform skill, set up Room KMP for both Android and iOS
```

```
Using the kmp-compose-multiplatform skill, configure Ktor client with auth interceptor and error handling
```

```
Using the kmp-compose-multiplatform skill, generate a Koin module for my data layer
```

## What Changes With This Skill

| Without skill | With skill |
|--------------|-----------|
| Generic Android ViewModel patterns | UDF pattern with `StateFlow` and `UiState` sealed classes |
| Android-only Room setup | Room KMP with `expect`/`actual` database builders |
| `Hilt` / `Dagger` suggestions | Koin Multiplatform with platform-specific modules |
| Single-platform Retrofit | Ktor client configured for `commonMain` |
| Mixed architecture advice | Strict Clean Architecture — domain never imports data |
| Guessed iOS integration | SPM wrapper pattern, `ComposeUIViewController`, Swift interop |

## Skill Content Overview

The skill ([`.claude/skills/kmp-compose-multiplatform/`](.claude/skills/kmp-compose-multiplatform/)) is structured as a precise instruction set Claude follows when invoked:

1. **Core Principles** — Foundational rules for KMP development
2. **Project Structure** — Module and package layout conventions
3. **Architecture Guidelines** — Layer responsibilities, dependency rules, code examples
4. **Compose Best Practices** — State hoisting, Material 3, modifiers, previews
5. **Navigation** — Type-safe routing with NavHostController
6. **KMP Patterns** — `expect`/`actual`, source sets, platform configuration
7. **Dependency Injection** — Koin module structure and initialization
8. **Build System** — Version catalog, BuildKonfig, KSP setup
9. **Data Persistence** — Room KMP and DataStore cross-platform setup
10. **Networking** — Ktor client configuration and error handling
11. **Testing** — Fakes over mocks, coroutine testing patterns
12. **iOS Integration** — SPM, `ComposeUIViewController`, Swift interop
13. **Common Pitfalls** — 10 mistakes to avoid in KMP projects
14. **Official References** — Links to authoritative documentation

## Architecture Reference

This skill implements the architecture pattern shown below, directly inspired by [Now in Android](https://github.com/android/nowinandroid):

```
┌─────────────────────────────────────────────────────────┐
│                    Presentation Layer                    │
│         ViewModel → UiState → Composable Screen         │
└─────────────────────┬───────────────────────────────────┘
                      │ uses
┌─────────────────────▼───────────────────────────────────┐
│                     Domain Layer                         │
│          UseCase → Repository Interface → Model          │
└─────────────────────┬───────────────────────────────────┘
                      │ implements
┌─────────────────────▼───────────────────────────────────┐
│                      Data Layer                          │
│      Repository Impl → Remote/Local Source → DTO         │
└─────────────────────────────────────────────────────────┘
```

### Feature Module Structure

```
feature/
└── auth/
    ├── data/
    │   ├── local/         ← Room entities, DAOs
    │   ├── remote/        ← Ktor API services
    │   ├── repository/    ← Repository implementations
    │   └── mapper/        ← DTO ↔ Domain mappers
    ├── domain/
    │   ├── model/         ← Pure Kotlin domain models
    │   ├── repository/    ← Repository interfaces
    │   └── usecase/       ← Business logic (one action per class)
    ├── presentation/
    │   ├── ui/            ← Composable screens
    │   ├── viewmodel/     ← State holders
    │   └── state/         ← UiState sealed classes
    └── di/                ← Koin module
```

### Source Set Layout

```
src/
├── commonMain/            ← Shared business logic, UI, ViewModels
├── androidMain/           ← Android-specific implementations
├── iosMain/               ← iOS-specific implementations
├── commonTest/            ← Shared unit tests with fakes
├── androidUnitTest/       ← Android-specific tests
└── iosTest/               ← iOS-specific tests
```

## iOS Integration via Swift Package Manager

The recommended pattern for distributing the KMP shared module to an iOS app is a **SPM Wrapper target** — this is required because a `binaryTarget` (the compiled XCFramework) cannot declare Swift package dependencies on its own.

### Package.swift Structure

```swift
// swift-tools-version: 5.9
import PackageDescription

let package = Package(
    name: "MyShared",
    platforms: [.iOS(.v16)],
    products: [
        // Expose the wrapper, not the binary directly
        .library(name: "MyShared", targets: ["MySharedWrapper"]),
    ],
    dependencies: [
        // Add Swift-only dependencies here (RevenueCat, Firebase, etc.)
        .package(url: "https://github.com/RevenueCat/purchases-hybrid-common.git", exact: "17.32.0"),
    ],
    targets: [
        // Binary target — the compiled XCFramework from Kotlin/Native
        .binaryTarget(
            name: "MySharedBinary",
            url: "https://github.com/org/repo/releases/download/v1.0.0/MyShared.xcframework.zip",
            checksum: "abc123..."  // sha256 — auto-computed in CI
        ),
        // Wrapper target — bridges the binary with Swift dependencies
        .target(
            name: "MySharedWrapper",
            dependencies: [
                "MySharedBinary",
                .product(name: "PurchasesHybridCommon", package: "purchases-hybrid-common"),
            ]
        )
    ]
)
```

**Why the Wrapper pattern?**
- `binaryTarget` cannot declare Swift package dependencies directly
- The wrapper is an empty Swift target whose only job is to re-export the binary alongside any Swift-only dependencies
- Your iOS app only needs to `import MyShared` — no knowledge of the wrapper internals

### Integrating in Xcode

**Remote (production)** — add published releases via SPM:
1. `File → Add Package Dependencies`
2. Enter your GitHub repo URL
3. Select version rule → Add Package
4. `import MyShared` in your Swift files

**Local (development)** — use your local build directly:
1. `File → Add Package Dependencies → Add Local`
2. Select the root folder containing `Package.swift`
3. No checksum needed — uses the local XCFramework build

### iOS Entry Point

```kotlin
// iosMain/MainViewController.kt
fun MainViewController(): UIViewController = ComposeUIViewController {
    initKoin()
    AppTheme { AppNavigation() }
}
```

```swift
// ContentView.swift
import SwiftUI
import MyShared

struct ContentView: View {
    var body: some View {
        ComposeView().ignoresSafeArea(.all)
    }
}

struct ComposeView: UIViewControllerRepresentable {
    func makeUIViewController(context: Context) -> UIViewController {
        MainViewControllerKt.MainViewController()
    }
    func updateUIViewController(_ uiViewController: UIViewController, context: Context) {}
}
```

### CI/CD — Auto-generating Package.swift

Your GitHub Actions release workflow should:

```yaml
- name: Build XCFramework
  run: ./gradlew :shared:assembleReleaseXCFramework

- name: Zip XCFramework
  run: zip -r MyShared.xcframework.zip shared/build/XCFrameworks/release/MyShared.xcframework

- name: Compute checksum
  run: echo "CHECKSUM=$(swift package compute-checksum MyShared.xcframework.zip)" >> $GITHUB_ENV

- name: Update Package.swift
  run: |
    sed -i '' "s|url: \".*xcframework.zip\"|url: \"https://github.com/${{ github.repository }}/releases/download/${{ github.ref_name }}/MyShared.xcframework.zip\"|" Package.swift
    sed -i '' "s|checksum: \".*\"|checksum: \"$CHECKSUM\"|" Package.swift

- name: Commit Package.swift
  run: |
    git config user.email "ci@github.com"
    git config user.name "GitHub Actions"
    git add Package.swift
    git commit -m "chore: update Package.swift for ${{ github.ref_name }} [skip ci]"
    git push
```

> Never edit `Package.swift` manually — it should be generated by CI on every release.

## Recommended Tech Stack

| Concern | Library | Version |
|---------|---------|---------|
| UI | Compose Multiplatform | 1.8.0+ |
| DI | Koin | 4.1.1+ |
| Networking | Ktor | 3.0.3+ |
| DB | Room KMP | 2.8.4+ |
| Prefs | DataStore KMP | 1.1.1+ |
| Nav | Navigation Compose | 2.9.1+ |
| Images | Coil 3 | 3.0.4+ |
| DateTime | kotlinx-datetime | 0.6.1+ |
| Serialization | kotlinx-serialization | 1.7.3+ |
| Config | BuildKonfig | 0.17.1+ |
| Code gen | KSP | 2.1.0+ |

## Examples

See the [`examples/`](examples/) directory for ready-to-use scaffolds:

- [`examples/feature-template/`](examples/feature-template/) — Complete feature module with all layers wired
- [`examples/build-logic/`](examples/build-logic/) — Convention plugin examples for multi-module builds
- [`examples/core-template/`](examples/core-template/) — Core module setup (network, database, DI)

## Reference Architecture

This skill is grounded in official, production-tested documentation:

- **[Now in Android](https://github.com/android/nowinandroid)** — Google's official Android architecture reference app
- **[Android Architecture Guide](https://developer.android.com/topic/architecture)** — Official Google architecture documentation
- **[KMP Documentation](https://www.jetbrains.com/help/kotlin-multiplatform-dev/)** — JetBrains official KMP docs
- **[Compose Multiplatform](https://www.jetbrains.com/compose-multiplatform/)** — JetBrains official CMP docs
- Real-world production KMP project patterns

## Official Documentation Links

- [Kotlin Multiplatform](https://www.jetbrains.com/help/kotlin-multiplatform-dev/)
- [Compose Multiplatform](https://www.jetbrains.com/compose-multiplatform/)
- [Android Architecture Guide](https://developer.android.com/topic/architecture)
- [Now in Android](https://github.com/android/nowinandroid)
- [Room KMP](https://developer.android.com/kotlin/multiplatform/room)
- [DataStore KMP](https://developer.android.com/topic/libraries/architecture/datastore)
- [Koin Multiplatform](https://insert-koin.io/docs/reference/koin-mp/kmp)
- [Ktor Client](https://ktor.io/docs/client-create-multiplatform-application.html)
- [Navigation Compose](https://developer.android.com/guide/navigation/design/kotlin-dsl)
- [Compose State Guide](https://developer.android.com/develop/ui/compose/state)
- [Compose Layouts](https://developer.android.com/develop/ui/compose/layouts)

## Contributing

Contributions that improve the skill are welcome:

1. Fork the repository
2. Create a branch: `git checkout -b improve/feature-name`
3. Update the skill file with accurate, tested patterns
4. Reference official documentation for any new patterns added
5. Submit a pull request with a clear description

**What makes a good contribution:** new `expect`/`actual` patterns, corrected library APIs, additional common pitfalls, updated version references.

## License

MIT License — see [LICENSE](LICENSE)
