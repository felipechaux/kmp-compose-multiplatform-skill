---
name: kmp-compose-multiplatform
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

Always use a sealed class for async results with **typed domain errors** (never raw strings):

```kotlin
sealed class Resource<out T> {
    data class Success<out T>(val data: T) : Resource<T>()
    data class Error(val error: AppError) : Resource<Nothing>()
    data object Loading : Resource<Nothing>()
}
```

See `references/error-handling.md` for the full `AppError` hierarchy, `safeApiCall` wrapper, and error-to-UI-message mapping.

### Use Case Pattern

```kotlin
class GetUserUseCase(private val repository: UserRepository) {
    suspend operator fun invoke(userId: String): Resource<User> {
        return repository.getUser(userId)
    }
}
```

### ViewModel Pattern

Use `data class` UiState (not sealed class) for composable state with `_uiState.update { }`. Expose navigation events via a separate `SharedFlow`:

```kotlin
class HomeViewModel(
    private val getItemsUseCase: GetItemsUseCase
) : ViewModel() {

    private val _uiState = MutableStateFlow(HomeUiState())
    val uiState: StateFlow<HomeUiState> = _uiState.asStateFlow()

    // One-time navigation/event channel — never put navigation in UiState
    private val _events = MutableSharedFlow<HomeEvent>()
    val events: SharedFlow<HomeEvent> = _events.asSharedFlow()

    fun loadItems() {
        viewModelScope.launch {
            _uiState.update { it.copy(isLoading = true, errorMessage = null) }
            getItemsUseCase()
                .onSuccess { items ->
                    _uiState.update { it.copy(isLoading = false, items = items) }
                }
                .onError { error ->
                    _uiState.update { it.copy(isLoading = false, errorMessage = error.toUserMessage()) }
                }
        }
    }

    fun onItemClicked(id: String) {
        viewModelScope.launch {
            _events.emit(HomeEvent.NavigateToDetail(id))
        }
    }
}

// Flat data class — preferred over sealed class for composable state
data class HomeUiState(
    val isLoading: Boolean = false,
    val items: List<Item> = emptyList(),
    val errorMessage: String? = null   // human-readable, never AppError
)

// One-time events — navigation, toasts, analytics
sealed class HomeEvent {
    data class NavigateToDetail(val id: String) : HomeEvent()
    data object ShowUndoSnackbar : HomeEvent()
}
```

Collect events in the screen composable:

```kotlin
@Composable
fun HomeScreen(
    navController: NavHostController,
    viewModel: HomeViewModel = koinViewModel()
) {
    val uiState by viewModel.uiState.collectAsStateWithLifecycle()

    // Collect one-time events
    LaunchedEffect(Unit) {
        viewModel.events.collect { event ->
            when (event) {
                is HomeEvent.NavigateToDetail -> navController.navigate(Screen.Detail.createRoute(event.id))
                is HomeEvent.ShowUndoSnackbar -> { /* show snackbar */ }
            }
        }
    }

    HomeContent(uiState = uiState, onItemClick = viewModel::onItemClicked)
}
```

### StateFlow from Repository Flow

Use `stateIn()` to convert a repository `Flow` into a ViewModel `StateFlow`:

```kotlin
val uiState: StateFlow<HomeUiState> = itemsRepository.observeItems()
    .map { items -> HomeUiState(items = items) }
    .stateIn(
        scope = viewModelScope,
        started = SharingStarted.WhileSubscribed(5_000),
        initialValue = HomeUiState(isLoading = true)
    )
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

### Scopes — Feature-Scoped Dependencies

Use Koin scopes for dependencies that should live only as long as a feature/screen is active (e.g., a shopping cart, a multi-step form):

```kotlin
// Define a scope qualifier
val CartScope = named("CartScope")

val cartModule = module {
    // Scoped — one instance per CartScope lifecycle
    scope(CartScope) {
        scoped { CartRepository(get()) }
        scoped { CartViewModel(get()) }
    }
}

// Open scope when entering the feature
val cartScope = getKoin().createScope("cart_session", CartScope)
val cartViewModel = cartScope.get<CartViewModel>()

// Close scope when leaving — instance is garbage collected
cartScope.close()
```

### Lazy Injection

Use `inject()` (lazy delegation) instead of `get()` (eager) when the dependency may not be needed immediately:

```kotlin
class HomeViewModel : ViewModel() {
    private val analyticsService: AnalyticsService by inject()   // lazy
    private val repository: HomeRepository = get()               // eager
}
```

### Named Qualifiers

Use `named()` qualifiers when you need multiple instances of the same type in the same module — a common pattern for multiple API clients or dispatchers:

```kotlin
val networkModule = module {
    // Two HTTP clients with different base URLs, distinguished by name
    single<HttpClient>(named("main")) {
        provideHttpClient(baseUrl = BuildKonfig.API_BASE_URL, tokenProvider = get())
    }
    single<HttpClient>(named("auth")) {
        provideHttpClient(baseUrl = BuildKonfig.AUTH_BASE_URL, tokenProvider = get())
    }

    // Multiple dispatchers
    single<CoroutineDispatcher>(named("io")) { Dispatchers.IO }
    single<CoroutineDispatcher>(named("main")) { Dispatchers.Main }
}

// Inject by name
class UserRepository(
    private val mainClient: HttpClient = get(named("main")),
    private val authClient: HttpClient = get(named("auth"))
)
```

### ViewModel with SavedStateHandle

Bind `SavedStateHandle` in Koin using `viewModelOf` or the `params` API:

```kotlin
// Using viewModelOf — automatically injects SavedStateHandle
val featureModule = module {
    viewModelOf(::DetailViewModel)  // SavedStateHandle injected automatically
}

// Or manually via params
val featureModule = module {
    viewModel { params ->
        DetailViewModel(
            savedStateHandle = params.get(),
            getItemUseCase = get()
        )
    }
}
```

```kotlin
class DetailViewModel(
    savedStateHandle: SavedStateHandle,
    private val getItemUseCase: GetItemUseCase
) : ViewModel() {
    private val itemId: String = checkNotNull(savedStateHandle[Screen.Detail.ARG_ID])
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
@Database(entities = [UserEntity::class], version = 2)
abstract class AppDatabase : RoomDatabase() {
    abstract fun userDao(): UserDao
}
```

```kotlin
// androidMain — actual
actual class DatabaseBuilder actual constructor(private val context: Any?) {
    actual fun build(): AppDatabase = Room.databaseBuilder<AppDatabase>(
        context = context as Context,
        name = context.getDatabasePath("app.db").absolutePath
    )
    .addMigrations(MIGRATION_1_2)
    .build()
}

val MIGRATION_1_2 = object : Migration(1, 2) {
    override fun migrate(db: SupportSQLiteDatabase) {
        db.execSQL("ALTER TABLE users ADD COLUMN avatar_url TEXT")
    }
}
```

```kotlin
// iosMain — actual
actual class DatabaseBuilder actual constructor(context: Any?) {
    actual fun build(): AppDatabase = Room.databaseBuilder<AppDatabase>(
        name = NSHomeDirectory() + "/app.db"
    )
    .addMigrations(MIGRATION_1_2)
    .build()
}
```

### Room DAO — Reactive Queries

Always use `Flow<List<T>>` for queries that the UI observes — never return a raw `List`:

```kotlin
@Dao
interface UserDao {
    // Reactive — emits whenever the table changes
    @Query("SELECT * FROM users ORDER BY name ASC")
    fun observeAll(): Flow<List<UserEntity>>

    // One-shot suspend for writes
    @Upsert
    suspend fun upsert(user: UserEntity)

    @Delete
    suspend fun delete(user: UserEntity)

    @Query("SELECT * FROM users WHERE id = :id")
    suspend fun getById(id: String): UserEntity?

    // Transaction for atomic multi-step operations
    @Transaction
    suspend fun replaceAll(users: List<UserEntity>) {
        deleteAll()
        insertAll(users)
    }

    @Query("DELETE FROM users")
    suspend fun deleteAll()

    @Insert(onConflict = OnConflictStrategy.IGNORE)
    suspend fun insertAll(users: List<UserEntity>)
}
```

### Room Pagination with Paging 3

For large datasets use `PagingSource` — never load everything into memory:

```toml
# libs.versions.toml
paging = "3.3.6"
[libraries]
paging-runtime = { module = "androidx.paging:paging-runtime", version.ref = "paging" }
paging-compose = { module = "androidx.paging:paging-compose", version.ref = "paging" }
paging-testing = { module = "androidx.paging:paging-testing", version.ref = "paging" }
```

```kotlin
// DAO — return PagingSource instead of List
@Dao
interface ItemDao {
    @Query("SELECT * FROM items ORDER BY created_at DESC")
    fun pagingSource(): PagingSource<Int, ItemEntity>
}

// Repository
fun observeItemsPaged(): Flow<PagingData<Item>> = Pager(
    config = PagingConfig(pageSize = 20, enablePlaceholders = false),
    pagingSourceFactory = { itemDao.pagingSource() }
).flow.map { pagingData -> pagingData.map { it.toDomain() } }

// ViewModel
val pagedItems: Flow<PagingData<Item>> = itemsRepository
    .observeItemsPaged()
    .cachedIn(viewModelScope)

// Composable
@Composable
fun ItemListScreen(viewModel: HomeViewModel = koinViewModel()) {
    val items = viewModel.pagedItems.collectAsLazyPagingItems()

    LazyColumn {
        items(count = items.itemCount, key = items.itemKey { it.id }) { index ->
            items[index]?.let { ItemCard(item = it) }
        }
        item {
            when (items.loadState.append) {
                is LoadState.Loading -> CircularProgressIndicator()
                is LoadState.Error -> RetryButton(onClick = { items.retry() })
                else -> Unit
            }
        }
    }
}
```

### Room Full-Text Search (FTS)

```kotlin
@Fts4(contentEntity = ItemEntity::class)
@Entity(tableName = "items_fts")
data class ItemFtsEntity(
    @PrimaryKey @ColumnInfo(name = "rowid") val rowId: Int = 0,
    val title: String,
    val description: String
)

@Dao
interface ItemSearchDao {
    @Query("SELECT * FROM items WHERE rowid IN (SELECT rowid FROM items_fts WHERE items_fts MATCH :query)")
    fun search(query: String): Flow<List<ItemEntity>>
}
```

### DataStore Setup

```kotlin
// commonMain
expect fun createDataStore(producePath: () -> String): DataStore<Preferences>

internal const val DATASTORE_FILE = "app_prefs.preferences_pb"
```

---

## Networking with Ktor

```kotlin
// commonMain
class ApiService(private val client: HttpClient) {

    suspend fun getUser(id: String): UserDto =
        client.get("/users/$id").body()

    suspend fun createUser(request: CreateUserRequest): UserDto =
        client.post("/users") {
            contentType(ContentType.Application.Json)
            setBody(request)
        }.body()
}

// HTTP client setup (in DI module) — with auth, logging, and retry
fun provideHttpClient(
    baseUrl: String,
    tokenProvider: TokenProvider
): HttpClient = HttpClient {
    install(ContentNegotiation) {
        json(Json {
            ignoreUnknownKeys = true
            isLenient = true
        })
    }
    install(HttpTimeout) {
        requestTimeoutMillis = 30_000
        connectTimeoutMillis = 15_000
        socketTimeoutMillis = 30_000
    }
    install(Logging) {
        logger = Logger.DEFAULT
        level = if (BuildKonfig.IS_DEBUG) LogLevel.HEADERS else LogLevel.NONE
    }
    // Automatic retry for transient failures
    install(HttpRequestRetry) {
        retryOnServerErrors(maxRetries = 3)
        retryOnException(maxRetries = 3, retryOnTimeout = true)
        exponentialDelay(base = 2.0, maxDelayMs = 10_000)
    }
    // Auth token injection
    install(Auth) {
        bearer {
            loadTokens { BearerTokens(tokenProvider.getAccessToken(), tokenProvider.getRefreshToken()) }
            refreshTokens {
                val newTokens = tokenProvider.refresh()
                BearerTokens(newTokens.accessToken, newTokens.refreshToken)
            }
        }
    }
    defaultRequest {
        url(baseUrl)
        header(HttpHeaders.ContentType, ContentType.Application.Json)
    }
}
```

Always wrap API calls with `safeApiCall` to map Ktor exceptions to domain errors — see `references/error-handling.md`.

### Exponential Backoff with Jitter

Add randomization to retry delays to prevent thundering-herd problems when many clients retry simultaneously:

```kotlin
install(HttpRequestRetry) {
    retryOnServerErrors(maxRetries = 3)
    retryOnException(maxRetries = 3, retryOnTimeout = true)
    // Add jitter: randomize delay within ±500ms of calculated backoff
    exponentialDelay(base = 2.0, maxDelayMs = 10_000, randomizationMs = 500)
}
```

### Certificate Pinning (Android)

For high-security apps, pin the server's certificate to prevent MITM attacks:

```kotlin
// androidMain — OkHttp CertificatePinner
actual fun createHttpClient(baseUrl: String, tokenProvider: TokenProvider): HttpClient =
    HttpClient(OkHttp) {
        engine {
            config {
                certificatePinner(
                    CertificatePinner.Builder()
                        .add("api.example.com", "sha256/AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA=")
                        .add("api.example.com", "sha256/BBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBB=") // backup pin
                        .build()
                )
            }
        }
        // ... other config
    }
```

> Keep two pins active at all times (primary + backup) to allow certificate rotation without a forced update.

### SharedFlow Buffer Strategy

Choose buffer size and overflow behavior explicitly when emitting from multiple coroutines:

```kotlin
// One-time UI events (navigation, toasts) — no replay, drop oldest if consumer is slow
private val _events = MutableSharedFlow<HomeEvent>(
    replay = 0,
    extraBufferCapacity = 64,
    onBufferOverflow = BufferOverflow.DROP_OLDEST
)

// App-wide events (logout, session expiry) — replay 1 so late subscribers catch the event
private val _appEvents = MutableSharedFlow<AppEvent>(
    replay = 1,
    extraBufferCapacity = 16,
    onBufferOverflow = BufferOverflow.DROP_OLDEST
)
```

Never use `replay > 0` for navigation events — a screen re-subscribing would navigate again.

### HTTP Caching

Enable response caching in the Ktor client to reduce network calls and support offline reading:

```kotlin
// androidMain — OkHttp cache
actual fun createHttpClient(baseUrl: String, tokenProvider: TokenProvider): HttpClient =
    HttpClient(OkHttp) {
        engine {
            config {
                cache(Cache(
                    directory = context.cacheDir.resolve("http_cache"),
                    maxSize = 10L * 1024 * 1024  // 10 MB
                ))
            }
        }
        // ... other plugins
    }
```

For stale-while-revalidate behaviour, add headers in repository calls:

```kotlin
suspend fun getItems(): Resource<List<ItemDto>> = safeApiCall {
    client.get("/items") {
        header(HttpHeaders.CacheControl, "max-age=300")   // fresh for 5 min
    }.body()
}
```

### OAuth 2.0 Token Refresh

The Ktor `Auth` plugin handles token rotation automatically. Ensure the refresh call itself is unauthenticated to avoid infinite loops:

```kotlin
install(Auth) {
    bearer {
        loadTokens {
            BearerTokens(tokenStorage.accessToken, tokenStorage.refreshToken)
        }
        refreshTokens {
            // markAsRefreshTokenRequest() prevents Auth plugin re-intercepting this call
            val response = client.post("/auth/refresh") {
                markAsRefreshTokenRequest()
                setBody(RefreshRequest(oldTokens?.refreshToken ?: ""))
            }.body<TokenResponse>()

            tokenStorage.save(response.accessToken, response.refreshToken)
            BearerTokens(response.accessToken, response.refreshToken)
        }
        sendWithoutRequest { request ->
            request.url.host == "api.example.com"   // only attach token to your API
        }
    }
}

---

## Internationalization (i18n)

All user-facing strings must use Compose Multiplatform's resource system. Never hardcode text:

```kotlin
// GOOD — uses generated Res.string references
Text(text = stringResource(Res.string.home_title, userName))
Button(onClick = onRetry) { Text(text = stringResource(Res.string.action_retry)) }

// BAD — hardcoded, not translatable
Text(text = "Welcome, $userName")
```

Key rules:
- Define all strings in `commonMain/composeResources/values/strings.xml`
- Add locale folders (`values-es/`, `values-ar/`) for each supported language
- Use `pluralStringResource()` for quantities — never `if (count == 1)` string branching
- Use `start`/`end` padding (not `left`/`right`) for RTL language support
- Use `Icons.AutoMirrored.*` for directional icons that should flip in RTL
- Test with `@Preview(locale = "ar")` to verify RTL layouts

See `references/i18n.md` for plurals, RTL testing, dynamic locale change, and locale-aware number/currency formatting.

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

**Rules**:
- Never use Mockito or MockK — use fakes/test doubles
- All shared tests go in `commonTest`
- Platform-specific tests in `androidTest`/`iosTest`
- Use `runTest` from `kotlinx-coroutines-test` for coroutine testing

---

## Logging

Use `expect`/`actual` for platform logging — **never use `println()`** in production code. Never log sensitive data (tokens, passwords, PII).

```kotlin
// commonMain — log levels matching platform conventions
enum class LogLevel { DEBUG, INFO, WARN, ERROR }

expect fun logDebug(tag: String, message: String)
expect fun logInfo(tag: String, message: String)
expect fun logWarn(tag: String, message: String)
expect fun logError(tag: String, message: String, throwable: Throwable? = null)
```

```kotlin
// androidMain — use Timber for structured Android logging
actual fun logDebug(tag: String, message: String) = Timber.tag(tag).d(message)
actual fun logInfo(tag: String, message: String) = Timber.tag(tag).i(message)
actual fun logWarn(tag: String, message: String) = Timber.tag(tag).w(message)
actual fun logError(tag: String, message: String, throwable: Throwable?) =
    Timber.tag(tag).e(throwable, message)
```

Initialize Timber in `Application.onCreate()` — plant a `DebugTree` for debug builds and a **Crashlytics reporting tree** for production:

```kotlin
class MyApp : Application() {
    override fun onCreate() {
        super.onCreate()
        if (BuildKonfig.IS_DEBUG) {
            Timber.plant(Timber.DebugTree())
        } else {
            Timber.plant(CrashlyticsTree())
        }
    }
}

// Production tree — routes WARN/ERROR to Firebase Crashlytics
class CrashlyticsTree : Timber.Tree() {
    override fun log(priority: Int, tag: String?, message: String, t: Throwable?) {
        // Only forward warnings and errors to crash reporting
        if (priority < Log.WARN) return
        // Never log sensitive data — scrub before sending
        val safeMessage = message.redactSensitivePatterns()
        FirebaseCrashlytics.getInstance().log("[$tag] $safeMessage")
        if (t != null) FirebaseCrashlytics.getInstance().recordException(t)
    }
}

// Redact tokens, emails, phone numbers before sending to crash services
private fun String.redactSensitivePatterns(): String = this
    .replace(Regex("Bearer [A-Za-z0-9\\-._~+/]+=*"), "Bearer [REDACTED]")
    .replace(Regex("[a-zA-Z0-9._%+\\-]+@[a-zA-Z0-9.\\-]+\\.[a-zA-Z]{2,}"), "[EMAIL REDACTED]")
```

```kotlin
// iosMain — use os_log (not NSLog, which is deprecated for structured logging)
import platform.Foundation.NSLog
import platform.darwin.*

actual fun logDebug(tag: String, message: String) {
    os_log_with_type(OS_LOG_DEFAULT, OS_LOG_TYPE_DEBUG, "[$tag] $message")
}
actual fun logInfo(tag: String, message: String) {
    os_log_with_type(OS_LOG_DEFAULT, OS_LOG_TYPE_INFO, "[$tag] $message")
}
actual fun logWarn(tag: String, message: String) {
    os_log_with_type(OS_LOG_DEFAULT, OS_LOG_TYPE_ERROR, "[$tag] WARN: $message")
}
actual fun logError(tag: String, message: String, throwable: Throwable?) {
    os_log_with_type(OS_LOG_DEFAULT, OS_LOG_TYPE_FAULT, "[$tag] ERROR: $message ${throwable?.message ?: ""}")
}
```

---

## Common Pitfalls to Avoid

1. **Never put Android/iOS imports in `commonMain`** — use expect/actual
2. **Never expose Flow from Room directly to UI** — map through repository to domain models first
3. **Never use `LiveData` in KMP** — use `StateFlow`/`Flow` only
4. **Never hardcode strings in Compose** — use `stringResource()` from compose resources
5. **Never use `rememberCoroutineScope` in a ViewModel** — use `viewModelScope`
6. **Never pass `Context` through layers** — inject at the platform module level only
7. **Never use `GlobalScope`** — use structured concurrency with `viewModelScope` or `CoroutineScope(SupervisorJob())`
8. **Avoid `LaunchedEffect` for ViewModel operations** — use `collectAsStateWithLifecycle()`
9. **Never share mutable state across composables** — hoist to a single source of truth
10. **Do not skip the domain layer** — even for simple features, maintain the abstraction
11. **Never use raw `String` for errors in `Resource.Error`** — use typed `AppError` sealed class
12. **Never put navigation calls in `UiState`** — use a separate `SharedFlow<Event>` for one-time events
13. **Never call `stopKoin()` in production code** — only in test teardown
14. **Never use `println()` for logging** — use `expect/actual` log functions
15. **Never skip Room migrations** — always add a `Migration` object when bumping the schema version
16. **Never omit `contentDescription` on meaningful images/icons** — required for accessibility (TalkBack, VoiceOver)
17. **Never log sensitive data** — redact tokens, emails, and PII before sending to Crashlytics or any log aggregation service
18. **Never load unbounded lists** — use `PagingSource` + `Pager` for large datasets
19. **Never hardcode user-facing strings** — always use `stringResource()` from compose resources
20. **Never use `left`/`right` padding in Composables** — use `start`/`end` for RTL language support
21. **Never use `Icons.Default.ArrowBack` for navigation** — use `Icons.AutoMirrored.Filled.ArrowBack` to mirror in RTL
22. **Never use `reply > 0` on navigation event SharedFlows** — late subscribers would trigger navigation again
23. **Never keep only one certificate pin** — always pin primary + backup to allow rotation without forcing an update
24. **Never use `left`/`right` in column/row alignment** — prefer `Start`/`End` which respect layout direction

---

## Reference Files

- `references/architecture.md` — detailed architecture guide, module structures, feature flags, inter-feature communication, proto DataStore
- `references/compose-best-practices.md` — composable design, `@Stable`, state hoisting, Material 3, focus management, text field accessibility, dynamic type, previews, performance
- `references/error-handling.md` — `AppError` hierarchy, `safeApiCall`, recoverable vs fatal, 429 handling, error analytics/breadcrumbs, retry logic
- `references/testing.md` — fakes, ViewModel tests with Turbine, SharedFlow event testing, Paging tests, screenshot/golden tests, Compose UI tests, Room in-memory
- `references/ios-interop.md` — Swift naming conventions, SKIE sealed class edge cases, Kotlin/Native memory model, iOS performance, nullability bridging, coroutines↔Swift Concurrency
- `references/navigation.md` — deep links, cross-module navigation contracts, predictive back, deep link validation, nested nav, bottom navigation, back handling, transitions, `SavedStateHandle`
- `references/build-system.md` — convention plugins, R8/ProGuard, publishing to Maven, CI Gradle daemon, KSP config, `gradle.properties`, build performance
- `references/i18n.md` — string resources, plurals, RTL support, dynamic locale change, locale-aware number/currency formatting

## Official References

- [KMP Documentation](https://www.jetbrains.com/help/kotlin-multiplatform-dev/)
- [Compose Multiplatform](https://www.jetbrains.com/compose-multiplatform/)
- [Android Architecture Guide](https://developer.android.com/topic/architecture)
- [Now in Android (Reference App)](https://github.com/android/nowinandroid)
- [Room KMP](https://developer.android.com/kotlin/multiplatform/room)
- [DataStore KMP](https://developer.android.com/topic/libraries/architecture/datastore)
- [Koin Multiplatform](https://insert-koin.io/docs/reference/koin-mp/kmp)
- [Ktor Client](https://ktor.io/docs/client-create-multiplatform-application.html)
- [Navigation Compose](https://developer.android.com/guide/navigation/design/kotlin-dsl)
- [Compose Localization](https://www.jetbrains.com/help/kotlin-multiplatform-dev/compose-multiplatform-resources-usage.html)
- [Predictive Back](https://developer.android.com/guide/navigation/custom-back/predictive-back-gesture)
- [Paging 3](https://developer.android.com/topic/libraries/architecture/paging/v3-overview)
