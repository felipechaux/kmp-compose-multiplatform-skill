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
