# Architecture Guide — KMP + Compose Multiplatform

Based on [Now in Android](https://github.com/android/nowinandroid) and [Android Architecture Guide](https://developer.android.com/topic/architecture).

## Overview

This guide defines the architecture used in KMP + Compose Multiplatform projects. The architecture follows three primary layers with strict dependency rules.

```
UI (Compose) → Domain (Use Cases) → Data (Repository) → Sources (API/DB)
```

Dependencies only flow **inward** — domain never depends on UI, data never depends on UI.

---

## Layer Definitions

### Presentation Layer

Responsibility: Display state and capture user events.

**Components:**
- `Screen` composables — stateless UI
- `ViewModel` — holds `StateFlow<UiState>`, exposes events
- `UiState` — sealed class/data class representing screen state

**Rules:**
- ViewModels may only depend on use cases (never repositories directly)
- Composables may only depend on state + callbacks (never ViewModels directly below the screen level)
- State is always `StateFlow`, never `LiveData`
- Never launch coroutines from composables — use `LaunchedEffect` sparingly or ViewModel events

### Domain Layer

Responsibility: Business logic and abstraction contracts.

**Components:**
- `UseCase` classes — single `operator fun invoke()`, one responsibility each
- Repository `interface` declarations
- Domain `model` classes (pure Kotlin data classes)

**Rules:**
- Zero Android or platform dependencies — pure Kotlin only
- Each use case does exactly one thing
- Repository interfaces live here, implementations in data layer
- Domain models are never database entities or DTOs

### Data Layer

Responsibility: Data access, transformation, and persistence.

**Components:**
- Repository `Impl` classes
- Remote data sources (`ApiService` via Ktor)
- Local data sources (Room DAOs, DataStore)
- `Mapper` functions — DTO ↔ Domain, Entity ↔ Domain
- DTO and Entity data classes

**Rules:**
- Data models (DTOs, entities) never escape this layer
- All repository methods return `Resource<T>` or `Flow<Resource<T>>`
- Mappers are pure functions, never classes

---

## Module Structure

### Single-Module KMP (recommended for shared libraries)

```
shared/
└── src/
    ├── commonMain/kotlin/com/example/shared/
    │   ├── core/
    │   │   ├── data/          # AppDatabase, DataStore setup
    │   │   ├── di/            # Core Koin module
    │   │   └── util/          # Resource, extensions
    │   ├── feature/
    │   │   ├── auth/          # Auth feature
    │   │   ├── home/          # Home feature
    │   │   └── settings/      # Settings feature
    │   ├── presentation/
    │   │   └── ui/
    │   │       ├── navigation/ # AppNavigation, Screen
    │   │       ├── theme/      # AppTheme, colors, typography
    │   │       └── components/ # Shared composables
    │   ├── di/
    │   │   └── AppModule.kt   # getAllModules() aggregator
    │   └── KoinInitializer.kt
    ├── androidMain/kotlin/
    └── iosMain/kotlin/
```

### Multi-Module KMP (for larger apps — Now in Android pattern)

```
root/
├── app/                       # Android entry point
├── iosApp/                    # iOS entry point
├── build-logic/               # Convention plugins
│   └── convention/
│       └── src/main/kotlin/
│           ├── KmpLibraryPlugin.kt
│           ├── ComposePlugin.kt
│           └── KoinPlugin.kt
├── core/
│   ├── data/                  # Base repository classes
│   ├── database/              # Room setup
│   ├── network/               # Ktor client
│   ├── datastore/             # DataStore
│   ├── ui/                    # Shared composables
│   └── designsystem/          # Theme, colors, typography
└── feature/
    ├── auth/
    │   ├── api/               # Navigation contract + interfaces
    │   └── impl/              # Full feature implementation
    └── home/
        ├── api/
        └── impl/
```

---

## Dependency Rules (Multi-Module)

```
app → feature:*:impl, feature:*:api, core:*
feature:impl → feature:*:api (never other impls), core:*
feature:api → core:designsystem (for navigation types only)
core:data → core:network, core:database, core:datastore
core:network → (external only)
core:database → (external only)
core:designsystem → (external + compose only)
```

**Never:**
- `feature:api` → other `feature` modules
- `core:*` → `feature:*`
- `core:*` → `app`
- Domain layer → Data layer

---

## State Management

### Unidirectional Data Flow (UDF)

```
User Action → ViewModel Event → Repository → UseCase → ViewModel State → UI
```

### UiState Pattern

```kotlin
// Simple state
data class HomeUiState(
    val isLoading: Boolean = false,
    val items: List<Item> = emptyList(),
    val error: String? = null
)

// OR sealed class for mutually exclusive states
sealed class HomeUiState {
    data object Loading : HomeUiState()
    data class Success(val items: List<Item>) : HomeUiState()
    data class Error(val message: String) : HomeUiState()
    data object Idle : HomeUiState()
}
```

### Collecting State in Compose

```kotlin
// Use collectAsStateWithLifecycle for lifecycle-aware collection
val uiState by viewModel.uiState.collectAsStateWithLifecycle()
```

---

## Navigation Architecture

Navigation is defined at the top-level (`AppNavigation.kt`) and uses type-safe sealed class routes.

### Route Definitions

```kotlin
sealed class Screen(val route: String) {
    data object Splash : Screen("splash")
    data object Home : Screen("home")
    data object Detail : Screen("detail/{id}") {
        fun createRoute(id: String) = "detail/$id"
        const val ARG_ID = "id"
    }
}
```

### NavHost Setup

```kotlin
@Composable
fun AppNavigation(
    startDestination: String = Screen.Splash.route,
    navController: NavHostController = rememberNavController()
) {
    NavHost(
        navController = navController,
        startDestination = startDestination
    ) {
        composable(
            route = Screen.Splash.route,
            exitTransition = { fadeOut() }
        ) {
            SplashScreen(
                onNavigateToHome = {
                    navController.navigate(Screen.Home.route) {
                        popUpTo(Screen.Splash.route) { inclusive = true }
                    }
                }
            )
        }

        composable(Screen.Home.route) {
            HomeScreen(navController = navController)
        }

        composable(
            route = Screen.Detail.route,
            arguments = listOf(navArgument(Screen.Detail.ARG_ID) { type = NavType.StringType })
        ) { backStackEntry ->
            val id = backStackEntry.arguments?.getString(Screen.Detail.ARG_ID) ?: return@composable
            DetailScreen(id = id, onBack = navController::popBackStack)
        }
    }
}
```

---

## Testing Philosophy

Based on Now in Android's approach:

> "The app does not use any mocking libraries. This is a deliberate choice to ensure tests exercise real code paths."

### Test Doubles (Fakes over Mocks)

```kotlin
// Interface (in domain layer)
interface UserRepository {
    suspend fun getUser(id: String): Resource<User>
    fun observeUsers(): Flow<List<User>>
}

// Fake (in commonTest)
class FakeUserRepository : UserRepository {
    private val users = mutableMapOf<String, User>()
    private val usersFlow = MutableStateFlow<List<User>>(emptyList())

    fun addUser(user: User) {
        users[user.id] = user
        usersFlow.value = users.values.toList()
    }

    override suspend fun getUser(id: String): Resource<User> =
        users[id]?.let { Resource.Success(it) } ?: Resource.Error("User $id not found")

    override fun observeUsers(): Flow<List<User>> = usersFlow
}
```

### Test Structure

```
commonTest/
├── feature/
│   └── home/
│       ├── domain/usecase/GetHomeDataUseCaseTest.kt
│       └── presentation/viewmodel/HomeViewModelTest.kt
└── fake/
    ├── FakeUserRepository.kt
    └── FakeNetworkClient.kt
```

---

## iOS SPM Integration

### Package.swift — Wrapper Pattern

KMP projects expose a compiled XCFramework to iOS via Swift Package Manager. The key pattern is a **Wrapper target** that bridges the binary with Swift-only dependencies:

```
Package.swift
├── products
│   └── MyShared (library) → MySharedWrapper
├── targets
│   ├── MySharedBinary (binaryTarget) ← XCFramework zip from GitHub Releases
│   └── MySharedWrapper (target)
│       ├── depends on: MySharedBinary
│       └── depends on: PurchasesHybridCommon (or other Swift packages)
```

**Why not expose the binaryTarget directly?**
`binaryTarget` cannot declare Swift package dependencies. The wrapper solves this — it's an empty Swift target whose only job is to re-export the binary alongside Swift dependencies.

### Local vs Remote SPM

| Mode | When | How |
|------|------|-----|
| Local | Development | `File → Add Package Dependencies → Add Local` → select repo root |
| Remote | Production | `File → Add Package Dependencies` → enter GitHub repo URL |

The `Package.swift` is **auto-generated by CI** on every release — never edit it manually. The release workflow computes the SHA256 checksum of the XCFramework zip and writes it into `Package.swift` automatically.

---

## Feature Flags

Use a `FeatureFlagRepository` backed by local config (BuildKonfig/hardcoded) and optionally remote config (Firebase Remote Config) to gate features at runtime:

```kotlin
// commonMain/core/featureflags/FeatureFlag.kt
enum class FeatureFlag(val key: String, val defaultValue: Boolean) {
    NEW_HOME_UI("new_home_ui", false),
    DARK_MODE_V2("dark_mode_v2", true),
    PAYMENT_REDESIGN("payment_redesign", false)
}

interface FeatureFlagRepository {
    fun isEnabled(flag: FeatureFlag): Boolean
    fun observe(flag: FeatureFlag): Flow<Boolean>
}
```

```kotlin
// Local implementation (always available — no network dependency)
class LocalFeatureFlagRepository : FeatureFlagRepository {
    // Overrides from build config or hardcoded for testing
    private val overrides = mutableMapOf<String, Boolean>()

    override fun isEnabled(flag: FeatureFlag): Boolean =
        overrides[flag.key] ?: flag.defaultValue

    override fun observe(flag: FeatureFlag): Flow<Boolean> =
        flowOf(isEnabled(flag))

    fun override(flag: FeatureFlag, enabled: Boolean) {
        overrides[flag.key] = enabled
    }
}
```

```kotlin
// androidMain — Firebase Remote Config backed (production)
class RemoteFeatureFlagRepository(
    private val remoteConfig: FirebaseRemoteConfig,
    private val local: LocalFeatureFlagRepository
) : FeatureFlagRepository {

    override fun isEnabled(flag: FeatureFlag): Boolean =
        remoteConfig.getBoolean(flag.key)  // falls back to defaultValue

    override fun observe(flag: FeatureFlag): Flow<Boolean> = flow {
        emit(isEnabled(flag))
        remoteConfig.fetchAndActivate().addOnCompleteListener {
            // Re-emit after remote fetch
        }
    }
}
```

Gate UI at the Composable level — never in domain layer:

```kotlin
@Composable
fun HomeScreen(
    viewModel: HomeViewModel = koinViewModel(),
    featureFlags: FeatureFlagRepository = get()
) {
    val showNewUI by featureFlags.observe(FeatureFlag.NEW_HOME_UI)
        .collectAsStateWithLifecycle(initialValue = false)

    if (showNewUI) NewHomeContent() else LegacyHomeContent()
}
```

---

## Inter-Feature Communication

Features must never depend on each other's implementations. Use one of two patterns:

### Pattern 1: Shared Domain Event via SharedFlow (for cross-feature events)

Define events in `core:domain` — a module every feature depends on:

```kotlin
// core/domain/events/AppEvents.kt
sealed class AppEvent {
    data class UserLoggedOut(val reason: String) : AppEvent()
    data class CartItemAdded(val itemId: String) : AppEvent()
    data object SessionExpired : AppEvent()
}

interface AppEventBus {
    val events: SharedFlow<AppEvent>
    suspend fun emit(event: AppEvent)
}
```

```kotlin
// core/data/events/AppEventBusImpl.kt
class AppEventBusImpl : AppEventBus {
    private val _events = MutableSharedFlow<AppEvent>(
        replay = 0,
        extraBufferCapacity = 64,
        onBufferOverflow = BufferOverflow.DROP_OLDEST
    )
    override val events: SharedFlow<AppEvent> = _events.asSharedFlow()
    override suspend fun emit(event: AppEvent) = _events.emit(event)
}
```

Any feature ViewModel subscribes in `init {}`:

```kotlin
class CartViewModel(
    private val eventBus: AppEventBus,
    private val clearCartUseCase: ClearCartUseCase
) : ViewModel() {
    init {
        viewModelScope.launch {
            eventBus.events.filterIsInstance<AppEvent.UserLoggedOut>()
                .collect { clearCartUseCase() }
        }
    }
}
```

### Pattern 2: Feature API Contract (for navigation/callbacks)

Each feature exposes a public API in its `:api` module — other features depend only on the API:

```kotlin
// feature/auth/api/AuthFeatureApi.kt  (in feature:auth:api)
interface AuthFeatureApi {
    fun loginRoute(): String
    fun onLoginSuccess(navController: NavController)
}
```

```kotlin
// feature/auth/impl  implements AuthFeatureApi
class AuthFeatureApiImpl : AuthFeatureApi {
    override fun loginRoute() = Screen.Login.route
    override fun onLoginSuccess(navController: NavController) {
        navController.navigate(Screen.Home.route) {
            popUpTo(0) { inclusive = true }
        }
    }
}
```

Register in DI, inject where needed — no direct impl dependency required.

---

## Proto DataStore (Typed Preferences)

Use Proto DataStore for typed, schema-versioned preferences. Prefer over `Preferences DataStore` when the data model is complex:

```protobuf
// commonMain/proto/user_preferences.proto
syntax = "proto3";
option java_package = "com.example.shared.datastore";

message UserPreferences {
  string theme = 1;          // "light" | "dark" | "system"
  bool notifications_enabled = 2;
  string language_code = 3;
  int32 schema_version = 4;
}
```

```kotlin
// commonMain
object UserPreferencesSerializer : Serializer<UserPreferences> {
    override val defaultValue: UserPreferences = UserPreferences(
        theme = "system",
        notificationsEnabled = true,
        languageCode = "en",
        schemaVersion = 1
    )
    override suspend fun readFrom(input: InputStream): UserPreferences =
        UserPreferences.ADAPTER.decode(input)
    override suspend fun writeTo(t: UserPreferences, output: OutputStream) =
        t.encode(output)
}

class UserPreferencesRepository(private val dataStore: DataStore<UserPreferences>) {
    val preferences: Flow<UserPreferences> = dataStore.data

    suspend fun setTheme(theme: String) {
        dataStore.updateData { it.copy(theme = theme) }
    }
}
```

---

## Official Resources

| Resource | URL |
|----------|-----|
| Now in Android | https://github.com/android/nowinandroid |
| Android Architecture Guide | https://developer.android.com/topic/architecture |
| Compose Architecture | https://developer.android.com/develop/ui/compose/architecture |
| Compose State | https://developer.android.com/develop/ui/compose/state |
| Modularization Guide | https://developer.android.com/topic/modularization |
| KMP Getting Started | https://www.jetbrains.com/help/kotlin-multiplatform-dev/multiplatform-getting-started.html |
| Room KMP | https://developer.android.com/kotlin/multiplatform/room |
