# Navigation — KMP + Compose Multiplatform

References: [Navigation Compose](https://developer.android.com/guide/navigation/design/kotlin-dsl) | [Deep Links](https://developer.android.com/guide/navigation/design/deep-link) | [Predictive Back](https://developer.android.com/guide/navigation/custom-back/predictive-back-gesture)

---

## Type-Safe Routes

Define all routes as a sealed class. Argument types are declared explicitly — never use raw strings in navigation calls:

```kotlin
// commonMain/presentation/ui/navigation/Screen.kt
sealed class Screen(val route: String) {
    data object Home : Screen("home")
    data object Settings : Screen("settings")

    data object Detail : Screen("detail/{id}") {
        const val ARG_ID = "id"
        fun createRoute(id: String) = "detail/$id"
    }

    data object Profile : Screen("profile/{userId}?tab={tab}") {
        const val ARG_USER_ID = "userId"
        const val ARG_TAB = "tab"
        fun createRoute(userId: String, tab: String = "posts") = "profile/$userId?tab=$tab"
    }
}
```

---

## NavHost Setup with Transitions

```kotlin
@Composable
fun AppNavigation(
    navController: NavHostController = rememberNavController(),
    startDestination: String = Screen.Home.route
) {
    NavHost(
        navController = navController,
        startDestination = startDestination,
        enterTransition = {
            slideIntoContainer(AnimatedContentTransitionScope.SlideDirection.Start, tween(300))
        },
        exitTransition = {
            slideOutOfContainer(AnimatedContentTransitionScope.SlideDirection.Start, tween(300))
        },
        popEnterTransition = {
            slideIntoContainer(AnimatedContentTransitionScope.SlideDirection.End, tween(300))
        },
        popExitTransition = {
            slideOutOfContainer(AnimatedContentTransitionScope.SlideDirection.End, tween(300))
        }
    ) {
        composable(Screen.Home.route) {
            HomeScreen(navController = navController)
        }

        composable(
            route = Screen.Detail.route,
            arguments = listOf(
                navArgument(Screen.Detail.ARG_ID) { type = NavType.StringType }
            ),
            deepLinks = listOf(
                navDeepLink { uriPattern = "myapp://detail/{id}" },
                navDeepLink { uriPattern = "https://myapp.com/detail/{id}" }
            )
        ) { backStackEntry ->
            val id = backStackEntry.arguments?.getString(Screen.Detail.ARG_ID)
                ?: return@composable
            DetailScreen(id = id, onBack = navController::popBackStack)
        }

        composable(
            route = Screen.Profile.route,
            arguments = listOf(
                navArgument(Screen.Profile.ARG_USER_ID) { type = NavType.StringType },
                navArgument(Screen.Profile.ARG_TAB) {
                    type = NavType.StringType
                    defaultValue = "posts"
                }
            )
        ) { backStackEntry ->
            val userId = backStackEntry.arguments?.getString(Screen.Profile.ARG_USER_ID) ?: return@composable
            val tab = backStackEntry.arguments?.getString(Screen.Profile.ARG_TAB) ?: "posts"
            ProfileScreen(userId = userId, initialTab = tab)
        }
    }
}
```

---

## Deep Links

### Android Setup

Declare deep link intent filters in `AndroidManifest.xml`:

```xml
<activity android:name=".MainActivity">
    <intent-filter android:autoVerify="true">
        <action android:name="android.intent.action.VIEW" />
        <category android:name="android.intent.category.DEFAULT" />
        <category android:name="android.intent.category.BROWSABLE" />
        <data android:scheme="https" android:host="myapp.com" />
        <data android:scheme="myapp" />
    </intent-filter>
</activity>
```

Pass the intent to `NavController` in `MainActivity`:

```kotlin
class MainActivity : ComponentActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContent {
            val navController = rememberNavController()
            AppTheme {
                AppNavigation(navController = navController)
            }
            // Handle deep link from intent
            LaunchedEffect(intent) {
                navController.handleDeepLink(intent)
            }
        }
    }
}
```

### iOS Deep Link Handling

Handle universal links and custom URL schemes in Swift:

```swift
// iOSApp.swift
@main
struct iOSApp: App {
    var body: some Scene {
        WindowGroup {
            ContentView()
                .onOpenURL { url in
                    DeepLinkHandlerKt.handleDeepLink(url: url.absoluteString)
                }
        }
    }
}
```

```kotlin
// iosMain — expose deep link handler
object DeepLinkHandler {
    var onDeepLink: ((String) -> Unit)? = null

    fun handleDeepLink(url: String) {
        onDeepLink?.invoke(url)
    }
}
```

---

## Nested Navigation (Bottom Navigation)

Use a nested `NavHost` per tab — each tab maintains its own back stack:

```kotlin
@Composable
fun MainScreen() {
    val navController = rememberNavController()
    val tabs = listOf(Tab.Home, Tab.Search, Tab.Profile)

    Scaffold(
        bottomBar = {
            NavigationBar {
                val currentDestination by navController.currentBackStackEntryAsState()
                tabs.forEach { tab ->
                    NavigationBarItem(
                        selected = currentDestination?.destination?.hierarchy
                            ?.any { it.route == tab.route } == true,
                        onClick = {
                            navController.navigate(tab.route) {
                                // Avoid building up a large back stack
                                popUpTo(navController.graph.findStartDestination().id) {
                                    saveState = true
                                }
                                launchSingleTop = true
                                restoreState = true  // Restore tab state on reselect
                            }
                        },
                        icon = { Icon(tab.icon, contentDescription = tab.label) },
                        label = { Text(tab.label) }
                    )
                }
            }
        }
    ) { innerPadding ->
        NavHost(
            navController = navController,
            startDestination = Tab.Home.route,
            modifier = Modifier.padding(innerPadding)
        ) {
            homeGraph(navController)
            searchGraph(navController)
            profileGraph(navController)
        }
    }
}

// Nested graph per tab
fun NavGraphBuilder.homeGraph(navController: NavHostController) {
    navigation(startDestination = Screen.Home.route, route = Tab.Home.route) {
        composable(Screen.Home.route) { HomeScreen(navController) }
        composable(Screen.Detail.route) { /* ... */ }
    }
}
```

---

## Back Navigation

### Predictive Back Gesture (Android 14+)

Use `BackHandler` to intercept system back:

```kotlin
@Composable
fun HomeScreen(viewModel: HomeViewModel = koinViewModel()) {
    val uiState by viewModel.uiState.collectAsStateWithLifecycle()

    // Intercept back only when there's unsaved state
    BackHandler(enabled = uiState.hasUnsavedChanges) {
        viewModel.onBackPressed()  // Show confirmation dialog
    }

    // ...
}
```

### Custom Back Navigation

```kotlin
@Composable
fun DetailScreen(
    id: String,
    onBack: () -> Unit,
    modifier: Modifier = Modifier
) {
    Scaffold(
        topBar = {
            TopAppBar(
                title = { Text("Detail") },
                navigationIcon = {
                    IconButton(onClick = onBack) {
                        Icon(Icons.AutoMirrored.Filled.ArrowBack, contentDescription = "Back")
                    }
                }
            )
        }
    ) { ... }
}
```

---

## Navigation State in ViewModel (via SharedFlow)

Never navigate directly from a ViewModel. Emit events via `SharedFlow` and handle them in the composable:

```kotlin
// ViewModel
class HomeViewModel : ViewModel() {
    private val _events = MutableSharedFlow<HomeEvent>()
    val events: SharedFlow<HomeEvent> = _events.asSharedFlow()

    fun onItemClicked(id: String) {
        viewModelScope.launch {
            _events.emit(HomeEvent.NavigateToDetail(id))
        }
    }
}

sealed class HomeEvent {
    data class NavigateToDetail(val id: String) : HomeEvent()
    data object NavigateToSettings : HomeEvent()
}

// Screen composable
@Composable
fun HomeScreen(
    navController: NavHostController,
    viewModel: HomeViewModel = koinViewModel()
) {
    LaunchedEffect(Unit) {
        viewModel.events.collect { event ->
            when (event) {
                is HomeEvent.NavigateToDetail ->
                    navController.navigate(Screen.Detail.createRoute(event.id))
                HomeEvent.NavigateToSettings ->
                    navController.navigate(Screen.Settings.route)
            }
        }
    }
}
```

---

## Navigation State Persistence (Process Death)

Use `rememberSaveable` for UI state that should survive process death. For navigation stack, the `NavController` saves and restores automatically via `SavedStateHandle`:

```kotlin
// Access navigation arguments in ViewModel via SavedStateHandle
class DetailViewModel(
    savedStateHandle: SavedStateHandle,
    private val getItemUseCase: GetItemUseCase
) : ViewModel() {

    private val itemId: String = checkNotNull(savedStateHandle[Screen.Detail.ARG_ID])

    val uiState: StateFlow<DetailUiState> = getItemUseCase(itemId)
        .map { resource -> resource.toUiState() }
        .stateIn(viewModelScope, SharingStarted.WhileSubscribed(5_000), DetailUiState())
}
```

Register in Koin with `SavedStateHandle`:

```kotlin
val detailModule = module {
    viewModel { params -> DetailViewModel(savedStateHandle = params.get(), getItemUseCase = get()) }
}
```

---

## Dialog and Bottom Sheet Navigation

```kotlin
NavHost(...) {
    composable(Screen.Home.route) { HomeScreen() }

    // Modal dialog
    dialog(Screen.ConfirmDelete.route) {
        ConfirmDeleteDialog(
            onConfirm = { navController.popBackStack() },
            onDismiss = { navController.popBackStack() }
        )
    }

    // Bottom sheet
    bottomSheet(Screen.Filter.route) {
        FilterBottomSheet(
            onApply = { navController.popBackStack() }
        )
    }
}
```

---

## Cross-Module Navigation Contracts

In multi-module apps, features must not import each other's `Screen` classes. Use a navigation contract interface in the `:api` module:

```kotlin
// feature/auth/api/AuthNavigation.kt  (feature:auth:api module)
interface AuthNavigation {
    val loginRoute: String
    val registerRoute: String
    fun navigateToLogin(navController: NavController)
    fun navigateAfterLogin(navController: NavController)
}
```

```kotlin
// feature/auth/impl  (implements the contract)
class AuthNavigationImpl : AuthNavigation {
    override val loginRoute = "auth/login"
    override val registerRoute = "auth/register"

    override fun navigateToLogin(navController: NavController) {
        navController.navigate(loginRoute)
    }

    override fun navigateAfterLogin(navController: NavController) {
        navController.navigate("home") {
            popUpTo(loginRoute) { inclusive = true }
        }
    }
}
```

Register in the root `NavHost` — only the app-level module knows all feature implementations:

```kotlin
// app/AppNavigation.kt — wires all feature graphs together
@Composable
fun AppNavigation(
    authNav: AuthNavigation = get(),
    homeNav: HomeNavigation = get()
) {
    val navController = rememberNavController()

    NavHost(navController, startDestination = authNav.loginRoute) {
        authGraph(navController, authNav)
        homeGraph(navController, homeNav)
    }
}
```

---

## Predictive Back (Android 14+)

Enable the predictive back gesture animation by adding the flag to the Android manifest and using `PredictiveBackHandler` for custom back animations:

```xml
<!-- AndroidManifest.xml -->
<application android:enableOnBackInvokedCallback="true" ...>
```

```kotlin
// For a custom screen-level back animation with Predictive Back progress
@Composable
fun DetailScreen(onBack: () -> Unit) {
    var scale by remember { mutableFloatStateOf(1f) }

    PredictiveBackHandler { progress ->
        // progress is a Flow<BackEventCompat> emitting 0.0 → 1.0 as user swipes
        try {
            progress.collect { backEvent ->
                scale = 1f - (backEvent.progress * 0.1f)  // shrink slightly during gesture
            }
            // User committed the back gesture
            onBack()
        } catch (e: CancellationException) {
            // User cancelled the back gesture — restore state
            scale = 1f
        }
    }

    Box(modifier = Modifier.scale(scale)) {
        DetailContent()
    }
}
```

For the default system animation (no custom handling needed), simply set the manifest flag — the system provides the animation automatically.

---

## Deep Link Validation

Validate deep link parameters before processing — never trust incoming URL data:

```kotlin
// ViewModel — validate deep link arguments from SavedStateHandle
class DetailViewModel(savedStateHandle: SavedStateHandle) : ViewModel() {

    // Will throw if argument is missing — handle at the NavHost level
    private val rawId: String = checkNotNull(savedStateHandle[Screen.Detail.ARG_ID]) {
        "DetailScreen requires a non-null item ID"
    }

    // Validate format before use
    val itemId: String = rawId.takeIf { it.isNotBlank() && it.length <= 64 }
        ?: throw IllegalArgumentException("Invalid item ID: $rawId")
}
```

Test deep links in Android with ADB:

```bash
# Test HTTPS deep link
adb shell am start -a android.intent.action.VIEW \
  -d "https://myapp.com/detail/123" \
  com.example.myapp

# Test custom scheme
adb shell am start -a android.intent.action.VIEW \
  -d "myapp://detail/123" \
  com.example.myapp
```

Register a `NavDeepLinkRequest` builder in tests to verify routing:

```kotlin
@Test
fun deepLink_navigatesToDetailScreen() {
    val request = NavDeepLinkRequest.Builder
        .fromUri("https://myapp.com/detail/abc123".toUri())
        .build()
    navController.handleDeepLink(request)

    assertEquals(Screen.Detail.route, navController.currentDestination?.route)
    assertEquals("abc123", navController.currentBackStackEntry
        ?.arguments?.getString(Screen.Detail.ARG_ID))
}
```
