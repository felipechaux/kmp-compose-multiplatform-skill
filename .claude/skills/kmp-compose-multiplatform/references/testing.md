# Testing — KMP + Compose Multiplatform

References: [Now in Android Testing](https://github.com/android/nowinandroid) | [Compose Testing](https://developer.android.com/develop/ui/compose/testing) | [Coroutines Testing](https://kotlinlang.org/api/kotlinx.coroutines/kotlinx-coroutines-test/)

---

## Test Dependencies

```toml
# gradle/libs.versions.toml
[versions]
kotlin-test = "2.3.0"
kotlinx-coroutines-test = "1.10.2"
turbine = "1.2.0"
androidx-test-junit = "1.2.1"
compose-ui-test = "1.8.0"

[libraries]
kotlin-test = { module = "org.jetbrains.kotlin:kotlin-test", version.ref = "kotlin-test" }
kotlinx-coroutines-test = { module = "org.jetbrains.kotlinx:kotlinx-coroutines-test", version.ref = "kotlinx-coroutines-test" }
turbine = { module = "app.cash.turbine:turbine", version.ref = "turbine" }
androidx-test-junit = { module = "androidx.test.ext:junit", version.ref = "androidx-test-junit" }
compose-ui-test-junit = { module = "androidx.compose.ui:ui-test-junit4", version.ref = "compose-ui-test" }
compose-ui-test-manifest = { module = "androidx.compose.ui:ui-test-manifest", version.ref = "compose-ui-test" }
```

```kotlin
// shared/build.gradle.kts
sourceSets {
    commonTest.dependencies {
        implementation(libs.kotlin.test)
        implementation(libs.kotlinx.coroutines.test)
        implementation(libs.turbine)
    }
    androidUnitTest.dependencies {
        implementation(libs.androidx.test.junit)
    }
    androidInstrumentedTest.dependencies {
        implementation(libs.compose.ui.test.junit)
        debugImplementation(libs.compose.ui.test.manifest)
    }
}
```

---

## Test Doubles (Fakes over Mocks)

Never use Mockito or MockK. Write fakes — real implementations of interfaces that are controllable in tests.

### Base Fake Pattern

```kotlin
// commonTest/fake/FakeUserRepository.kt
class FakeUserRepository : UserRepository {

    // Controllable state
    private val users = mutableMapOf<String, User>()
    private val usersFlow = MutableStateFlow<List<User>>(emptyList())
    var shouldReturnError: AppError? = null

    // Test setup helpers
    fun addUser(user: User) {
        users[user.id] = user
        usersFlow.value = users.values.toList()
    }

    fun setError(error: AppError) {
        shouldReturnError = error
    }

    // Interface implementation
    override suspend fun getUser(id: String): Resource<User> {
        shouldReturnError?.let { return Resource.Error(it) }
        return users[id]?.let { Resource.Success(it) }
            ?: Resource.Error(AppError.Database.NotFound)
    }

    override fun observeUsers(): Flow<List<User>> = usersFlow
}
```

---

## Use Case Tests

```kotlin
// commonTest/feature/user/domain/usecase/GetUserUseCaseTest.kt
class GetUserUseCaseTest {

    private val repository = FakeUserRepository()
    private val useCase = GetUserUseCase(repository)

    @Test
    fun `returns success when user exists`() = runTest {
        val expected = User(id = "1", name = "Alice")
        repository.addUser(expected)

        val result = useCase("1")

        assertIs<Resource.Success<User>>(result)
        assertEquals(expected, result.data)
    }

    @Test
    fun `returns not found error when user missing`() = runTest {
        val result = useCase("unknown-id")

        assertIs<Resource.Error>(result)
        assertIs<AppError.Database.NotFound>(result.error)
    }

    @Test
    fun `propagates repository error`() = runTest {
        repository.setError(AppError.Network.NoConnection)

        val result = useCase("1")

        assertIs<Resource.Error>(result)
        assertIs<AppError.Network.NoConnection>(result.error)
    }
}
```

---

## ViewModel Tests

Use [Turbine](https://github.com/cashapp/turbine) for Flow testing:

```kotlin
// commonTest/feature/home/presentation/viewmodel/HomeViewModelTest.kt
class HomeViewModelTest {

    private val repository = FakeUserRepository()
    private val getItemsUseCase = GetItemsUseCase(repository)
    private lateinit var viewModel: HomeViewModel

    @BeforeTest
    fun setup() {
        Dispatchers.setMain(UnconfinedTestDispatcher())
        viewModel = HomeViewModel(getItemsUseCase)
    }

    @AfterTest
    fun tearDown() {
        Dispatchers.resetMain()
    }

    @Test
    fun `initial state is empty and not loading`() = runTest {
        val state = viewModel.uiState.value
        assertFalse(state.isLoading)
        assertTrue(state.items.isEmpty())
        assertNull(state.errorMessage)
    }

    @Test
    fun `loading then success flow`() = runTest {
        repository.addUser(User(id = "1", name = "Alice"))

        viewModel.uiState.test {
            // Initial idle state
            val idle = awaitItem()
            assertFalse(idle.isLoading)

            viewModel.loadItems()

            // Loading state
            val loading = awaitItem()
            assertTrue(loading.isLoading)

            // Success state
            val success = awaitItem()
            assertFalse(success.isLoading)
            assertEquals(1, success.items.size)
            assertNull(success.errorMessage)
        }
    }

    @Test
    fun `error state set on failure`() = runTest {
        repository.setError(AppError.Network.NoConnection)

        viewModel.uiState.test {
            awaitItem() // initial

            viewModel.loadItems()
            awaitItem() // loading

            val error = awaitItem()
            assertFalse(error.isLoading)
            assertNotNull(error.errorMessage)
            assertTrue(error.items.isEmpty())
        }
    }
}
```

---

## Repository Integration Tests (commonTest)

Test the repository with a fake data source — not the real network, but a real local implementation where possible:

```kotlin
// commonTest/feature/user/data/repository/UserRepositoryImplTest.kt
class UserRepositoryImplTest {

    private val fakeRemote = FakeUserRemoteDataSource()
    private val fakeLocal = FakeUserLocalDataSource()
    private val repository = UserRepositoryImpl(fakeRemote, fakeLocal)

    @Test
    fun `fetches from remote and caches locally on success`() = runTest {
        val remoteUser = UserDto(id = "1", name = "Alice")
        fakeRemote.setUser(remoteUser)

        val result = repository.getUser("1")

        assertIs<Resource.Success<User>>(result)
        assertEquals("Alice", result.data.name)
        // Verify local cache was populated
        assertNotNull(fakeLocal.getUser("1"))
    }

    @Test
    fun `returns cached data on network failure`() = runTest {
        fakeLocal.addUser(UserEntity(id = "1", name = "Alice (cached)"))
        fakeRemote.setError(AppError.Network.NoConnection)

        val result = repository.getUser("1")

        assertIs<Resource.Success<User>>(result)
        assertEquals("Alice (cached)", result.data.name)
    }
}
```

---

## Room In-Memory Database Tests (androidUnitTest)

```kotlin
// androidUnitTest/data/local/UserDaoTest.kt
@RunWith(AndroidJUnit4::class)
class UserDaoTest {

    private lateinit var database: AppDatabase
    private lateinit var userDao: UserDao

    @Before
    fun setup() {
        database = Room.inMemoryDatabaseBuilder(
            context = ApplicationProvider.getApplicationContext(),
            klass = AppDatabase::class.java
        ).allowMainThreadQueries().build()
        userDao = database.userDao()
    }

    @After
    fun tearDown() {
        database.close()
    }

    @Test
    fun insertAndRetrieveUser() = runTest {
        val entity = UserEntity(id = "1", name = "Alice", email = "alice@example.com")
        userDao.insert(entity)

        val result = userDao.getById("1")
        assertEquals(entity, result)
    }

    @Test
    fun observeUsersEmitsOnInsert() = runTest {
        userDao.observeAll().test {
            assertEquals(emptyList<UserEntity>(), awaitItem())

            userDao.insert(UserEntity(id = "1", name = "Alice", email = ""))
            val updated = awaitItem()
            assertEquals(1, updated.size)
        }
    }
}
```

---

## Compose UI Tests (androidInstrumentedTest)

```kotlin
// androidInstrumentedTest/feature/home/HomeScreenTest.kt
@RunWith(AndroidJUnit4::class)
class HomeScreenTest {

    @get:Rule
    val composeTestRule = createComposeRule()

    @Test
    fun showsLoadingIndicator_whenStateIsLoading() {
        composeTestRule.setContent {
            AppTheme {
                HomeContent(
                    uiState = HomeUiState(isLoading = true),
                    onRetry = {}
                )
            }
        }

        composeTestRule
            .onNodeWithContentDescription("Loading")
            .assertIsDisplayed()
    }

    @Test
    fun showsItems_whenStateIsSuccess() {
        val items = listOf(Item(id = "1", title = "First item"))

        composeTestRule.setContent {
            AppTheme {
                HomeContent(
                    uiState = HomeUiState(items = items),
                    onRetry = {}
                )
            }
        }

        composeTestRule
            .onNodeWithText("First item")
            .assertIsDisplayed()
    }

    @Test
    fun showsErrorAndRetryButton_whenStateHasError() {
        var retryClicked = false

        composeTestRule.setContent {
            AppTheme {
                HomeContent(
                    uiState = HomeUiState(errorMessage = "No internet connection"),
                    onRetry = { retryClicked = true }
                )
            }
        }

        composeTestRule
            .onNodeWithText("No internet connection")
            .assertIsDisplayed()

        composeTestRule
            .onNodeWithText("Retry")
            .performClick()

        assertTrue(retryClicked)
    }
}
```

---

## Accessibility Testing

Test that composables expose correct semantics for screen readers:

```kotlin
@RunWith(AndroidJUnit4::class)
class HomeScreenAccessibilityTest {

    @get:Rule
    val composeTestRule = createComposeRule()

    @Test
    fun favoriteButton_hasCorrectContentDescription() {
        composeTestRule.setContent {
            AppTheme {
                ItemCard(
                    item = Item(id = "1", title = "My Item", isFavorite = false),
                    onFavoriteClick = {}
                )
            }
        }

        composeTestRule
            .onNodeWithContentDescription("Add to favorites")
            .assertIsDisplayed()
            .assertHasClickAction()
    }

    @Test
    fun loadingIndicator_hasAccessibleDescription() {
        composeTestRule.setContent {
            AppTheme {
                HomeContent(uiState = HomeUiState(isLoading = true), onRetry = {})
            }
        }

        composeTestRule
            .onNodeWithContentDescription("Loading")
            .assertIsDisplayed()
    }

    @Test
    fun errorState_retryButtonIsAccessible() {
        composeTestRule.setContent {
            AppTheme {
                HomeContent(
                    uiState = HomeUiState(errorMessage = "No connection"),
                    onRetry = {}
                )
            }
        }

        composeTestRule
            .onNodeWithText("Retry")
            .assertHasClickAction()
            .assertIsEnabled()
    }

    @Test
    fun mergedSemantics_cardExposesCorrectDescription() {
        val item = Item(id = "1", title = "Alice", subtitle = "Engineer")

        composeTestRule.setContent {
            AppTheme { UserCard(user = item) }
        }

        // Merged semantics should produce one accessible node
        composeTestRule
            .onNodeWithContentDescription("Alice, Engineer", substring = true)
            .assertExists()
    }
}
```

---

## SharedFlow Event Testing

Test one-time navigation/UI events emitted via `SharedFlow`:

```kotlin
// commonTest/feature/home/presentation/viewmodel/HomeViewModelEventTest.kt
class HomeViewModelEventTest {

    private val repository = FakeUserRepository()
    private val viewModel = HomeViewModel(GetItemsUseCase(repository))

    @BeforeTest
    fun setup() {
        Dispatchers.setMain(UnconfinedTestDispatcher())
    }

    @AfterTest
    fun tearDown() {
        Dispatchers.resetMain()
    }

    @Test
    fun `onItemClicked emits NavigateToDetail event`() = runTest {
        viewModel.events.test {
            viewModel.onItemClicked("item-123")

            val event = awaitItem()
            assertIs<HomeEvent.NavigateToDetail>(event)
            assertEquals("item-123", event.id)

            cancelAndConsumeRemainingEvents()
        }
    }

    @Test
    fun `delete action emits ShowUndoSnackbar event`() = runTest {
        viewModel.events.test {
            viewModel.onDeleteItem("item-456")

            val event = awaitItem()
            assertIs<HomeEvent.ShowUndoSnackbar>(event)
        }
    }

    @Test
    fun `multiple events are emitted in order`() = runTest {
        viewModel.events.test {
            viewModel.onItemClicked("first")
            viewModel.onItemClicked("second")

            assertEquals("first", (awaitItem() as HomeEvent.NavigateToDetail).id)
            assertEquals("second", (awaitItem() as HomeEvent.NavigateToDetail).id)
        }
    }
}
```

---

## Paging 3 Tests

Test `PagingData` streams using `paging-testing` artifact:

```kotlin
// androidUnitTest/feature/home/data/repository/ItemRepositoryPagingTest.kt
@RunWith(AndroidJUnit4::class)
class ItemRepositoryPagingTest {

    private val fakeDao = FakeItemDao()
    private val repository = ItemRepositoryImpl(fakeDao)

    @Test
    fun pagingSource_loadsFirstPage() = runTest {
        // Seed fake DAO with 50 items
        repeat(50) { i -> fakeDao.insert(ItemEntity(id = "item-$i", title = "Item $i")) }

        val pager = Pager(
            config = PagingConfig(pageSize = 20, enablePlaceholders = false),
            pagingSourceFactory = { fakeDao.pagingSource() }
        )

        val snapshot = pager.flow
            .asSnapshot()  // from paging-testing — collects and waits for first page

        assertEquals(20, snapshot.size)
        assertEquals("item-0", snapshot.first().id)
    }

    @Test
    fun pagingSource_loadsAllItems_withScrolling() = runTest {
        repeat(45) { i -> fakeDao.insert(ItemEntity(id = "item-$i", title = "Item $i")) }

        val snapshot = Pager(
            config = PagingConfig(pageSize = 20),
            pagingSourceFactory = { fakeDao.pagingSource() }
        ).flow.asSnapshot {
            scrollTo(index = 44)  // scroll to trigger all pages to load
        }

        assertEquals(45, snapshot.size)
    }
}
```

---

## Screenshot / Golden Tests

Use Paparazzi (Android) or Roborazzi for composable screenshot regression tests:

```toml
# libs.versions.toml
paparazzi = "1.3.5"
[plugins]
paparazzi = { id = "app.cash.paparazzi", version.ref = "paparazzi" }
```

```kotlin
// androidUnitTest/feature/home/HomeScreenScreenshotTest.kt
@RunWith(JUnit4::class)
class HomeScreenScreenshotTest {

    @get:Rule
    val paparazzi = Paparazzi(
        deviceConfig = DeviceConfig.PIXEL_5,
        theme = "android:Theme.Material.Light.NoActionBar"
    )

    @Test
    fun homeScreen_loadingState() {
        paparazzi.snapshot {
            AppTheme {
                HomeContent(
                    uiState = HomeUiState(isLoading = true),
                    onAction = {}
                )
            }
        }
    }

    @Test
    fun homeScreen_successState() {
        paparazzi.snapshot {
            AppTheme {
                HomeContent(
                    uiState = HomeUiState(items = PreviewData.items),
                    onAction = {}
                )
            }
        }
    }

    @Test
    fun homeScreen_errorState() {
        paparazzi.snapshot {
            AppTheme {
                HomeContent(
                    uiState = HomeUiState(errorMessage = "No internet connection"),
                    onRetry = {}
                )
            }
        }
    }

    @Test
    fun homeScreen_darkTheme() {
        paparazzi.snapshot {
            AppTheme(darkTheme = true) {
                HomeContent(
                    uiState = HomeUiState(items = PreviewData.items),
                    onAction = {}
                )
            }
        }
    }
}
```

Run golden comparison:
```bash
# Record golden images (first run or after intentional UI changes)
./gradlew :shared:recordPaparazziDebug

# Verify screenshots match goldens (CI)
./gradlew :shared:verifyPaparazziDebug
```

---

## Koin Test Modules

```kotlin
// commonTest/di/TestModules.kt
val testNetworkModule = module {
    single<UserRepository> { FakeUserRepository() }
    single<ApiService> { FakeApiService() }
}

// In ViewModel tests with Koin
class HomeViewModelKoinTest : KoinTest {

    @get:Rule
    val koinRule = KoinTestRule.create {
        modules(testNetworkModule)
    }

    @Test
    fun `koin resolves ViewModel correctly`() = runTest {
        val viewModel: HomeViewModel = get()
        assertNotNull(viewModel)
    }
}
```

Always call `stopKoin()` after tests that start Koin manually:

```kotlin
@AfterTest
fun tearDown() {
    stopKoin()
}
```

---

## Test Structure

```
shared/
└── src/
    ├── commonTest/kotlin/
    │   ├── fake/
    │   │   ├── FakeUserRepository.kt
    │   │   ├── FakeApiService.kt
    │   │   └── FakeLocalDataSource.kt
    │   ├── di/
    │   │   └── TestModules.kt
    │   └── feature/
    │       └── home/
    │           ├── domain/usecase/GetItemsUseCaseTest.kt
    │           └── presentation/viewmodel/HomeViewModelTest.kt
    ├── androidUnitTest/kotlin/
    │   └── data/local/UserDaoTest.kt
    └── androidInstrumentedTest/kotlin/
        └── feature/home/HomeScreenTest.kt
```

---

## Key Rules

- **Never mock** — use fakes with controllable state
- **Always test error paths** — set errors on fakes and assert correct UI state
- **Use Turbine** for Flow/StateFlow assertions — cleaner than `toList()` with jobs
- **`UnconfinedTestDispatcher`** for ViewModel tests (immediate execution)
- **`StandardTestDispatcher`** for tests where you need explicit advancement with `advanceUntilIdle()`
- **`stopKoin()`** after any test that calls `startKoin()`
- **In-memory Room** for DAO tests — never use production database
