---
description: Expert Kotlin Multiplatform (KMP) and Compose Multiplatform development guidance. Use when creating, reviewing, or modifying KMP projects with Compose UI, clean architecture, and multi-platform targets (Android, iOS, Desktop, Web).
---

# Kotlin Multiplatform + Compose Multiplatform Skill

You are an expert in **Kotlin Multiplatform (KMP)** and **Compose Multiplatform** development. You follow Google's official architecture guidelines (as demonstrated in Now in Android), JetBrains Compose best practices, and the KMP community standards.

## Core Principles

1. **Maximize shared code** — write once in `commonMain`, use everywhere
2. **Clean Architecture** — strict layer separation: Data → Domain → Presentation
3. **Feature-based modularization** — organize by feature, not by layer
4. **Unidirectional Data Flow (UDF)** — state flows down, events flow up
5. **Interface-first design** — define contracts, inject implementations
6. **Platform parity** — same behavior on Android and iOS unless explicitly platform-specific

---

## Project Structure

### Recommended Module Layout

```
root/
├── app/                          # Android app entry point
├── iosApp/                       # iOS app entry point (Xcode project)
├── shared/                       # KMP shared module (or multi-module)
│   └── src/
│       ├── commonMain/           # Shared code for all platforms
│       ├── androidMain/          # Android-specific implementations
│       ├── iosMain/              # iOS-specific implementations
│       └── commonTest/           # Shared tests
├── build-logic/                  # Convention plugins (if multi-module)
│   └── convention/               # Gradle convention plugins
└── gradle/
    └── libs.versions.toml        # Version catalog (ALWAYS use this)
```

### Feature Module Layout (inside commonMain)

Each feature must follow this exact structure:

```
feature/
└── [feature-name]/
    ├── data/
    │   ├── local/
    │   │   ├── dao/              # Room DAOs
    │   │   └── entity/           # Room entities
    │   ├── remote/               # API services
    │   ├── repository/           # Repository implementations
    │   └── mapper/               # Data ↔ Domain mappers
    ├── domain/
    │   ├── model/                # Domain models (pure Kotlin)
    │   ├── repository/           # Repository interfaces
    │   └── usecase/              # Use cases (one action per class)
    ├── presentation/
    │   ├── ui/                   # Composable screens and components
    │   ├── viewmodel/            # ViewModels
    │   └── state/                # UI state data classes
    └── di/                       # Koin module for this feature
```

---

## Architecture Guidelines

### Layer Responsibilities

**Data Layer**
- Implements repository interfaces from domain
- Maps data models to/from domain models
- Handles network requests (Ktor) and local persistence (Room/DataStore)
- Never exposes data models to domain or presentation

**Domain Layer**
- Pure Kotlin — NO Android/platform dependencies
- Repository interfaces (abstractions)
- Use cases: single public function `operator fun invoke()`
- Domain models (not database entities, not DTOs)

**Presentation Layer**
- ViewModels hold `StateFlow<UiState>` — never expose mutable state
- UI State is a sealed class or data class
- Composables receive state + callbacks (no direct ViewModel access in nested composables)
- Navigation handled at screen level only

### Resource/Result Pattern

Always use a sealed class for async results:

```kotlin
sealed class Resource<out T> {
    data class Success<out T>(val data: T) : Resource<T>()
    data class Error(val message: String, val throwable: Throwable? = null) : Resource<Nothing>()
    data object Loading : Resource<Nothing>()
}
```

### Use Case Pattern

```kotlin
class GetUserUseCase(private val repository: UserRepository) {
    suspend operator fun invoke(userId: String): Resource<User> {
        return repository.getUser(userId)
    }
}
```

### ViewModel Pattern

```kotlin
class HomeViewModel(
    private val generateIcebreakerUseCase: GenerateIcebreakerUseCase
) : ViewModel() {

    private val _uiState = MutableStateFlow<HomeUiState>(HomeUiState.Idle)
    val uiState: StateFlow<HomeUiState> = _uiState.asStateFlow()

    fun onGenerateClicked() {
        viewModelScope.launch {
            _uiState.value = HomeUiState.Loading
            generateIcebreakerUseCase().collect { resource ->
                _uiState.value = when (resource) {
                    is Resource.Success -> HomeUiState.Success(resource.data)
                    is Resource.Error -> HomeUiState.Error(resource.message)
                    is Resource.Loading -> HomeUiState.Loading
                }
            }
        }
    }
}

sealed class HomeUiState {
    data object Idle : HomeUiState()
    data object Loading : HomeUiState()
    data class Success(val data: IcebreakerResult) : HomeUiState()
    data class Error(val message: String) : HomeUiState()
}
```

---

## Compose Multiplatform Best Practices

### State Hoisting

Always hoist state to the screen level. Composables should be stateless:

```kotlin
// CORRECT — stateless composable
@Composable
fun HomeContent(
    uiState: HomeUiState,
    onGenerateClick: () -> Unit,
    modifier: Modifier = Modifier
) { ... }

// CORRECT — screen composable connects to ViewModel
@Composable
fun HomeScreen(
    viewModel: HomeViewModel = koinViewModel()
) {
    val uiState by viewModel.uiState.collectAsStateWithLifecycle()
    HomeContent(
        uiState = uiState,
        onGenerateClick = viewModel::onGenerateClicked
    )
}
```

### Design System

- Use **Material 3** (`MaterialTheme`) everywhere
- Define a central `AppTheme.kt` in `presentation/ui/theme/`
- Dark/light mode via `isSystemInDarkTheme()`
- Color palette as `object ColorPalette { val Primary = Color(...) }`
- Typography scale via `Typography { ... }`
- Never hardcode colors or text styles in composables — always use `MaterialTheme.colorScheme.*` and `MaterialTheme.typography.*`

### Modifiers

- Always accept `modifier: Modifier = Modifier` as last parameter before lambdas
- Apply modifier as the first thing inside the root layout
- Never apply padding inside a composable that accepts a modifier — let the caller decide spacing

### Previews

Always provide previews for composables:

```kotlin
@Preview(showBackground = true)
@Preview(showBackground = true, uiMode = UI_MODE_NIGHT_YES)
@Composable
private fun HomeContentPreview() {
    AppTheme {
        HomeContent(uiState = HomeUiState.Idle, onGenerateClick = {})
    }
}
```

### Navigation

Use type-safe sealed class routes:

```kotlin
sealed class Screen(val route: String) {
    data object Home : Screen("home")
    data object Detail : Screen("detail/{id}") {
        fun createRoute(id: String) = "detail/$id"
    }
}
```

Navigation setup:

```kotlin
@Composable
fun AppNavigation(navController: NavHostController = rememberNavController()) {
    NavHost(
        navController = navController,
        startDestination = Screen.Home.route,
        enterTransition = { fadeIn(animationSpec = tween(300)) },
        exitTransition = { fadeOut(animationSpec = tween(300)) }
    ) {
        composable(Screen.Home.route) { HomeScreen(navController) }
        composable(Screen.Detail.route) { backStackEntry ->
            val id = backStackEntry.arguments?.getString("id") ?: return@composable
            DetailScreen(id = id, navController = navController)
        }
    }
}
```

---

## Kotlin Multiplatform Patterns

### Expect/Actual Pattern

Use expect/actual for platform-specific implementations:

```kotlin
// commonMain
expect fun getPlatformName(): String

expect class DatabaseBuilder(context: Any?) {
    fun build(): AppDatabase
}
```

```kotlin
// androidMain
actual fun getPlatformName(): String = "Android"

actual class DatabaseBuilder actual constructor(private val context: Any?) {
    actual fun build(): AppDatabase =
        Room.databaseBuilder(context as Context, AppDatabase::class.java, "app.db").build()
}
```

```kotlin
// iosMain
actual fun getPlatformName(): String = "iOS"

actual class DatabaseBuilder actual constructor(context: Any?) {
    actual fun build(): AppDatabase {
        val dbFilePath = NSHomeDirectory() + "/app.db"
        return Room.databaseBuilder<AppDatabase>(name = dbFilePath).build()
    }
}
```

### Source Set Configuration (build.gradle.kts)

```kotlin
kotlin {
    androidTarget {
        compilations.all {
            compileTaskProvider.configure {
                compilerOptions {
                    jvmTarget.set(JvmTarget.JVM_17)
                }
            }
        }
    }

    listOf(iosX64(), iosArm64(), iosSimulatorArm64()).forEach { target ->
        target.binaries.framework {
            baseName = "shared"
            isStatic = true
        }
    }

    sourceSets {
        commonMain.dependencies {
            // Compose Multiplatform
            implementation(compose.runtime)
            implementation(compose.foundation)
            implementation(compose.material3)
            implementation(compose.ui)
            implementation(compose.components.resources)

            // Navigation
            implementation(libs.navigation.compose)

            // Koin
            implementation(libs.koin.core)
            implementation(libs.koin.compose)
            implementation(libs.koin.compose.viewmodel)

            // Ktor
            implementation(libs.ktor.client.core)
            implementation(libs.ktor.client.content.negotiation)
            implementation(libs.ktor.serialization.kotlinx.json)

            // Room
            implementation(libs.room.runtime)
            implementation(libs.room.ktx)

            // DataStore
            implementation(libs.datastore.preferences)

            // DateTime
            implementation(libs.kotlinx.datetime)

            // Serialization
            implementation(libs.kotlinx.serialization.json)

            // Coroutines
            implementation(libs.kotlinx.coroutines.core)
        }

        androidMain.dependencies {
            implementation(libs.ktor.client.okhttp)
            implementation(libs.koin.android)
            implementation(libs.kotlinx.coroutines.android)
        }

        iosMain.dependencies {
            implementation(libs.ktor.client.darwin)
        }
    }
}
```

---

## Dependency Injection with Koin

### Module Structure

```kotlin
// feature/home/di/HomeModule.kt
val homeModule = module {
    single<HomeRepository> { HomeRepositoryImpl(get(), get()) }
    factory { GetHomeDataUseCase(get()) }
    viewModel { HomeViewModel(get()) }
}
```

### Central Module Aggregator

```kotlin
// di/AppModule.kt
fun getAllModules() = listOf(
    platformModule(),
    coreModule,
    authModule,
    homeModule,
    // ... other feature modules
)

// Platform-specific (expect/actual)
expect fun platformModule(): Module
```

### Koin Initialization

```kotlin
// KoinInitializer.kt
fun initKoin(appDeclaration: KoinAppDeclaration = {}) {
    startKoin {
        appDeclaration()
        modules(getAllModules())
    }
}
```

Android entry (Application class):
```kotlin
class MyApp : Application() {
    override fun onCreate() {
        super.onCreate()
        initKoin {
            androidContext(this@MyApp)
        }
    }
}
```

iOS entry (Swift):
```swift
KoinInitializerKt.doInitKoin()
```

---

## Build System

### Version Catalog (gradle/libs.versions.toml)

Always use the version catalog. Never hardcode versions in build files:

```toml
[versions]
kotlin = "2.3.0"
compose-multiplatform = "1.10.1"
agp = "8.8.0"
koin = "4.1.1"
ktor = "3.0.3"
room = "2.8.4"
datastore = "1.1.1"
navigation-compose = "2.9.1"
kotlinx-coroutines = "1.10.2"
kotlinx-serialization = "1.7.3"
kotlinx-datetime = "0.6.1"
ksp = "2.3.0-1.0.32"
buildkonfig = "0.17.1"
coil = "3.0.4"

[libraries]
# Compose Multiplatform (accessed via compose.* in sourceSets)
# Koin
koin-core = { module = "io.insert-koin:koin-core", version.ref = "koin" }
koin-android = { module = "io.insert-koin:koin-android", version.ref = "koin" }
koin-compose = { module = "io.insert-koin:koin-compose", version.ref = "koin" }
koin-compose-viewmodel = { module = "io.insert-koin:koin-compose-viewmodel", version.ref = "koin" }
# Ktor
ktor-client-core = { module = "io.ktor:ktor-client-core", version.ref = "ktor" }
ktor-client-okhttp = { module = "io.ktor:ktor-client-okhttp", version.ref = "ktor" }
ktor-client-darwin = { module = "io.ktor:ktor-client-darwin", version.ref = "ktor" }
ktor-client-content-negotiation = { module = "io.ktor:ktor-client-content-negotiation", version.ref = "ktor" }
ktor-serialization-kotlinx-json = { module = "io.ktor:ktor-serialization-kotlinx-json", version.ref = "ktor" }
# Room
room-runtime = { module = "androidx.room:room-runtime", version.ref = "room" }
room-ktx = { module = "androidx.room:room-ktx", version.ref = "room" }
room-compiler = { module = "androidx.room:room-compiler", version.ref = "room" }
# DataStore
datastore-preferences = { module = "androidx.datastore:datastore-preferences-core", version.ref = "datastore" }
# Navigation
navigation-compose = { module = "org.jetbrains.androidx.navigation:navigation-compose", version.ref = "navigation-compose" }
# KotlinX
kotlinx-coroutines-core = { module = "org.jetbrains.kotlinx:kotlinx-coroutines-core", version.ref = "kotlinx-coroutines" }
kotlinx-coroutines-android = { module = "org.jetbrains.kotlinx:kotlinx-coroutines-android", version.ref = "kotlinx-coroutines" }
kotlinx-serialization-json = { module = "org.jetbrains.kotlinx:kotlinx-serialization-json", version.ref = "kotlinx-serialization" }
kotlinx-datetime = { module = "org.jetbrains.kotlinx:kotlinx-datetime", version.ref = "kotlinx-datetime" }
# Coil
coil-compose = { module = "io.coil-kt.coil3:coil-compose", version.ref = "coil" }
coil-network-ktor = { module = "io.coil-kt.coil3:coil-network-ktor3", version.ref = "coil" }

[plugins]
kotlin-multiplatform = { id = "org.jetbrains.kotlin.multiplatform", version.ref = "kotlin" }
compose-multiplatform = { id = "org.jetbrains.compose", version.ref = "compose-multiplatform" }
compose-compiler = { id = "org.jetbrains.kotlin.plugin.compose", version.ref = "kotlin" }
android-library = { id = "com.android.library", version.ref = "agp" }
android-application = { id = "com.android.application", version.ref = "agp" }
kotlin-serialization = { id = "org.jetbrains.kotlin.plugin.serialization", version.ref = "kotlin" }
ksp = { id = "com.google.devtools.ksp", version.ref = "ksp" }
room = { id = "androidx.room", version.ref = "room" }
buildkonfig = { id = "com.codingfeline.buildkonfig", version.ref = "buildkonfig" }
```

### BuildKonfig for Environment Configuration

```kotlin
// build.gradle.kts
buildkonfig {
    packageName = "com.example.shared"

    defaultConfigs {
        buildConfigField(STRING, "ENVIRONMENT", "stage")
        buildConfigField(BOOLEAN, "IS_DEBUG", "true")
        buildConfigField(STRING, "API_BASE_URL", "https://api.stage.example.com")
    }

    targetConfigs("prod") {
        buildConfigField(STRING, "ENVIRONMENT", "prod")
        buildConfigField(BOOLEAN, "IS_DEBUG", "false")
        buildConfigField(STRING, "API_BASE_URL", "https://api.example.com")
    }
}
```

---

## Data Persistence

### Room Database Setup

```kotlin
// commonMain
@Database(entities = [UserEntity::class], version = 1)
abstract class AppDatabase : RoomDatabase() {
    abstract fun userDao(): UserDao
}
```

```kotlin
// commonMain — expect
expect class DatabaseBuilder(context: Any?) {
    fun build(): AppDatabase
}
```

```kotlin
// androidMain — actual
actual class DatabaseBuilder actual constructor(private val context: Any?) {
    actual fun build(): AppDatabase = Room.databaseBuilder<AppDatabase>(
        context = context as Context,
        name = context.getDatabasePath("app.db").absolutePath
    ).build()
}
```

```kotlin
// iosMain — actual
actual class DatabaseBuilder actual constructor(context: Any?) {
    actual fun build(): AppDatabase = Room.databaseBuilder<AppDatabase>(
        name = NSHomeDirectory() + "/app.db"
    ).build()
}
```

### DataStore Setup

```kotlin
// commonMain
expect fun createDataStore(producePath: () -> String): DataStore<Preferences>

internal const val DATASTORE_FILE = "app_prefs.preferences_pb"
```

```kotlin
// androidMain
actual fun createDataStore(producePath: () -> String): DataStore<Preferences> =
    PreferenceDataStoreFactory.createWithPath(produceFile = { producePath().toPath() })
```

```kotlin
// iosMain
actual fun createDataStore(producePath: () -> String): DataStore<Preferences> =
    PreferenceDataStoreFactory.createWithPath(produceFile = { producePath().toPath() })
```

---

## Networking with Ktor

```kotlin
// commonMain
class ApiService(private val client: HttpClient) {

    suspend fun getUser(id: String): UserDto {
        return client.get("/users/$id").body()
    }

    suspend fun createUser(request: CreateUserRequest): UserDto {
        return client.post("/users") {
            contentType(ContentType.Application.Json)
            setBody(request)
        }.body()
    }
}

// HTTP client setup (in DI module)
fun provideHttpClient(baseUrl: String): HttpClient = HttpClient {
    install(ContentNegotiation) {
        json(Json { ignoreUnknownKeys = true })
    }
    install(HttpTimeout) {
        requestTimeoutMillis = 30_000
        connectTimeoutMillis = 15_000
    }
    defaultRequest {
        url(baseUrl)
        header(HttpHeaders.ContentType, ContentType.Application.Json)
    }
}
```

---

## Testing Strategy

### Unit Tests (commonTest)

```kotlin
class GetUserUseCaseTest {

    private val repository = FakeUserRepository()
    private val useCase = GetUserUseCase(repository)

    @Test
    fun `returns success when repository succeeds`() = runTest {
        repository.setUser(testUser)
        val result = useCase("user-123")
        assertIs<Resource.Success<User>>(result)
        assertEquals(testUser, result.data)
    }
}

// Fake (not mock) — real implementation of the interface
class FakeUserRepository : UserRepository {
    private var user: User? = null

    fun setUser(user: User) { this.user = user }

    override suspend fun getUser(id: String): Resource<User> =
        user?.let { Resource.Success(it) } ?: Resource.Error("Not found")
}
```

### ViewModel Tests

```kotlin
class HomeViewModelTest {

    @Test
    fun `uiState emits Loading then Success`() = runTest {
        val viewModel = HomeViewModel(FakeGenerateIcebreakerUseCase())

        val states = mutableListOf<HomeUiState>()
        val job = launch { viewModel.uiState.toList(states) }

        viewModel.onGenerateClicked()
        advanceUntilIdle()

        assertTrue(states.any { it is HomeUiState.Loading })
        assertTrue(states.last() is HomeUiState.Success)
        job.cancel()
    }
}
```

**Rules**:
- Never use Mockito or MockK — use fakes/test doubles
- All shared tests go in `commonTest`
- Platform-specific tests in `androidTest`/`iosTest`
- Use `runTest` from `kotlinx-coroutines-test` for coroutine testing

---

## iOS Integration

### Swift Package Manager

```swift
// Package.swift
let package = Package(
    name: "SharedPackage",
    platforms: [.iOS(.v16)],
    products: [
        .library(name: "shared", targets: ["shared"])
    ],
    targets: [
        .binaryTarget(
            name: "shared",
            url: "https://github.com/org/repo/releases/download/v1.0.0/shared.xcframework.zip",
            checksum: "abc123..."
        )
    ]
)
```

### iOS Entry Point

```kotlin
// iosMain/MainViewController.kt
fun MainViewController(): UIViewController = ComposeUIViewController {
    initKoin()
    AppTheme {
        AppNavigation()
    }
}
```

### Calling from Swift

```swift
// ContentView.swift
import SwiftUI
import shared

struct ContentView: View {
    var body: some View {
        ComposeView()
            .ignoresSafeArea(.all)
    }
}

struct ComposeView: UIViewControllerRepresentable {
    func makeUIViewController(context: Context) -> UIViewController {
        return MainViewControllerKt.MainViewController()
    }
    func updateUIViewController(_ uiViewController: UIViewController, context: Context) {}
}
```

---

## Common Pitfalls to Avoid

1. **Never put Android/iOS imports in `commonMain`** — use expect/actual
2. **Never expose Flow from Room directly to UI** — map through repository to domain models
3. **Never use `LiveData` in KMP** — use `StateFlow`/`Flow` only
4. **Never hardcode strings in Compose** — use `stringResource()` from compose resources
5. **Never use `rememberCoroutineScope` in a ViewModel** — use `viewModelScope`
6. **Never pass `Context` through layers** — inject at the platform module level only
7. **Never use `GlobalScope`** — use structured concurrency with `viewModelScope` or `CoroutineScope(SupervisorJob())`
8. **Avoid `LaunchedEffect` for ViewModel operations** — use `collectAsStateWithLifecycle()`
9. **Never share mutable state across composables** — hoist to a single source of truth
10. **Do not skip the domain layer** — even for simple features, maintain the abstraction

---

## Official References

- [KMP Documentation](https://www.jetbrains.com/help/kotlin-multiplatform-dev/)
- [Compose Multiplatform](https://www.jetbrains.com/compose-multiplatform/)
- [Android Architecture Guide](https://developer.android.com/topic/architecture)
- [Now in Android (Reference App)](https://github.com/android/nowinandroid)
- [Compose Layouts](https://developer.android.com/develop/ui/compose/layouts)
- [Compose State](https://developer.android.com/develop/ui/compose/state)
- [Room KMP](https://developer.android.com/kotlin/multiplatform/room)
- [DataStore KMP](https://developer.android.com/topic/libraries/architecture/datastore)
- [Koin Multiplatform](https://insert-koin.io/docs/reference/koin-mp/kmp)
- [Ktor Client](https://ktor.io/docs/client-create-multiplatform-application.html)
- [Navigation Compose](https://developer.android.com/guide/navigation/design/kotlin-dsl)
