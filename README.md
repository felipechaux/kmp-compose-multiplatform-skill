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
mkdir -p .claude/skills
curl -o .claude/skills/kmp-compose.md \
  https://raw.githubusercontent.com/aldefy/kmp-compose-multiplatform-skill/main/.claude/skills/kmp-compose.md
```

Or clone and copy:

```bash
git clone https://github.com/aldefy/kmp-compose-multiplatform-skill
cp kmp-compose-multiplatform-skill/.claude/skills/kmp-compose.md .claude/skills/
```

### Option 2: Global (available in all projects)

```bash
mkdir -p ~/.claude/skills
cp .claude/skills/kmp-compose.md ~/.claude/skills/
```

### Option 3: Git submodule

```bash
git submodule add https://github.com/aldefy/kmp-compose-multiplatform-skill .claude/kmp-skill
ln -s ../.claude/kmp-skill/.claude/skills/kmp-compose.md .claude/skills/kmp-compose.md
```

> **That's it.** Claude Code automatically picks up skill files from `.claude/skills/`. No configuration required.

## Usage

Once installed, invoke the skill in Claude Code:

```
/kmp-compose
```

Or reference it inline:

```
Using the kmp-compose skill, create a new feature module for user authentication
```

```
Using the kmp-compose skill, review my ViewModel for UDF violations
```

```
Using the kmp-compose skill, set up Room KMP for both Android and iOS
```

```
Using the kmp-compose skill, configure Ktor client with auth interceptor and error handling
```

```
Using the kmp-compose skill, generate a Koin module for my data layer
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

The main skill file [`kmp-compose.md`](.claude/skills/kmp-compose.md) is structured as a precise instruction set Claude follows when invoked:

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
