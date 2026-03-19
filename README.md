# KMP + Compose Multiplatform — Claude Code Skill

A **Claude Code skill** that turns any Claude agent into an expert in **Kotlin Multiplatform (KMP)** and **Compose Multiplatform** development — following Google's official architecture guidelines and JetBrains best practices.

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

## Reference Architecture

This skill is based on:

- **[Now in Android](https://github.com/android/nowinandroid)** — Google's official Android architecture reference app
- **[Android Architecture Guide](https://developer.android.com/topic/architecture)** — Official Google architecture documentation
- **[KMP Documentation](https://www.jetbrains.com/help/kotlin-multiplatform-dev/)** — JetBrains official KMP docs
- **[Compose Multiplatform](https://www.jetbrains.com/compose-multiplatform/)** — JetBrains official CMP docs
- Real-world production KMP project patterns

## Installation

### Option 1: Project-level (recommended for KMP projects)

Copy the skill into your project:

```bash
mkdir -p .claude/skills
cp kmp-compose.md .claude/skills/
```

Or clone directly:

```bash
git clone https://github.com/YOUR_USERNAME/kmp-compose-multiplatform-skill
cp kmp-compose-multiplatform-skill/.claude/skills/kmp-compose.md .claude/skills/
```

### Option 2: Global (available in all projects)

```bash
mkdir -p ~/.claude/skills
cp .claude/skills/kmp-compose.md ~/.claude/skills/
```

### Option 3: Use as a submodule

```bash
git submodule add https://github.com/YOUR_USERNAME/kmp-compose-multiplatform-skill .claude/kmp-skill
ln -s ../.claude/kmp-skill/.claude/skills/kmp-compose.md .claude/skills/kmp-compose.md
```

## Usage

Once installed, invoke the skill in Claude Code:

```
/kmp-compose
```

Or reference it in your prompt:

```
Using the kmp-compose skill, help me create a new feature module for user authentication
```

```
Using the kmp-compose skill, review my ViewModel for architecture violations
```

```
Using the kmp-compose skill, set up Room database for both Android and iOS
```

## Skill Content Overview

### [`kmp-compose.md`](.claude/skills/kmp-compose.md)

The main skill file contains:

1. **Core Principles** — Foundational rules for KMP development
2. **Project Structure** — Module and package layout conventions
3. **Architecture Guidelines** — Layer responsibilities, patterns, and examples
4. **Compose Best Practices** — State hoisting, Material 3, modifiers, previews
5. **Navigation** — Type-safe routing with NavHostController
6. **KMP Patterns** — expect/actual, source sets, platform configuration
7. **Dependency Injection** — Koin module structure and initialization
8. **Build System** — Version catalog, BuildKonfig, KSP setup
9. **Data Persistence** — Room and DataStore cross-platform setup
10. **Networking** — Ktor client configuration
11. **Testing** — Fakes over mocks, coroutine testing patterns
12. **iOS Integration** — SPM, ComposeUIViewController, Swift interop
13. **Common Pitfalls** — 10 mistakes to avoid
14. **Official References** — Links to authoritative docs

## Examples

See the [`examples/`](examples/) directory for:

- [`examples/feature-template/`](examples/feature-template/) — Complete feature module scaffold
- [`examples/build-logic/`](examples/build-logic/) — Convention plugin examples
- [`examples/core-template/`](examples/core-template/) — Core module setup

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
    │   ├── local/     ← Room entities, DAOs
    │   ├── remote/    ← Ktor API services
    │   ├── repository/← Repository implementations
    │   └── mapper/    ← DTO ↔ Domain mappers
    ├── domain/
    │   ├── model/     ← Pure Kotlin domain models
    │   ├── repository/← Repository interfaces
    │   └── usecase/   ← Business logic (one action per class)
    ├── presentation/
    │   ├── ui/        ← Composable screens
    │   ├── viewmodel/ ← State holders
    │   └── state/     ← UiState sealed classes
    └── di/            ← Koin module
```

## Recommended Tech Stack

| Concern | Library | Version |
|---------|---------|---------|
| UI | Compose Multiplatform | 1.10.1+ |
| DI | Koin | 4.1.1+ |
| Networking | Ktor | 3.0.3+ |
| DB | Room KMP | 2.8.4+ |
| Prefs | DataStore KMP | 1.1.1+ |
| Nav | Navigation Compose | 2.9.1+ |
| Images | Coil 3 | 3.0.4+ |
| DateTime | kotlinx-datetime | 0.6.1+ |
| Serialization | kotlinx-serialization | 1.7.3+ |
| Config | BuildKonfig | 0.17.1+ |
| Code gen | KSP | 2.3.0+ |

## Contributing

Contributions that improve the skill are welcome:

1. Fork the repository
2. Create a branch: `git checkout -b improve/feature-name`
3. Update the skill file with accurate, tested patterns
4. Reference official documentation for any new patterns added
5. Submit a pull request with a clear description

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

## License

MIT License — see [LICENSE](LICENSE)
