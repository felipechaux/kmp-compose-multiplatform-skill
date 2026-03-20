# Compose Multiplatform Best Practices

References: [Compose Architecture](https://developer.android.com/develop/ui/compose/architecture) | [Compose State](https://developer.android.com/develop/ui/compose/state) | [Compose Layouts](https://developer.android.com/develop/ui/compose/layouts) | [Compose Multiplatform](https://www.jetbrains.com/compose-multiplatform/)

---

## Composable Design Principles

### 1. Keep Composables Stateless

Composables should receive state and emit events — never hold state internally unless it is purely visual:

```kotlin
// GOOD — stateless, testable, reusable
@Composable
fun UserCard(
    name: String,
    avatarUrl: String,
    onCardClick: () -> Unit,
    modifier: Modifier = Modifier
) {
    Card(modifier = modifier.clickable(onClick = onCardClick)) {
        // ...
    }
}

// BAD — holds business state internally
@Composable
fun UserCard(userId: String) {
    val user by remember { mutableStateOf<User?>(null) }
    // fetching in composable is an anti-pattern
}
```

### 2. Always Accept a Modifier Parameter

Every composable must accept `modifier: Modifier = Modifier` as the last parameter before lambda parameters:

```kotlin
@Composable
fun PrimaryButton(
    text: String,
    onClick: () -> Unit,
    enabled: Boolean = true,
    modifier: Modifier = Modifier  // always last before lambdas
) {
    Button(
        onClick = onClick,
        enabled = enabled,
        modifier = modifier  // applied to the root element
    ) {
        Text(text = text)
    }
}
```

### 3. Extract Large Composables

Break large composables into smaller, focused functions:

```kotlin
// GOOD — clear structure
@Composable
fun HomeScreen(uiState: HomeUiState, onAction: (HomeAction) -> Unit) {
    Column {
        HomeHeader(uiState.title)
        HomeContent(uiState.items, onAction)
        HomeFooter(onAction)
    }
}

// BAD — one giant composable
@Composable
fun HomeScreen(uiState: HomeUiState, onAction: (HomeAction) -> Unit) {
    Column {
        // 200 lines of UI...
    }
}
```

### 4. Use Stable Parameters and @Stable

The Compose compiler marks a type as **stable** if it can guarantee that `equals()` is reliable and public properties only change when `equals()` returns `false`. Unstable parameters cause unnecessary recomposition.

**Stable by default:** primitives, `String`, lambdas, `data class` with all-stable properties.

**Unstable by default:** `List`, `Map`, `Set` (use `kotlinx.collections.immutable` for stable collections), classes with `var` properties, abstract classes/interfaces.

```kotlin
// GOOD — immutable list, stable
@Composable
fun ItemList(items: List<Item>, modifier: Modifier = Modifier) { ... }

// BAD — mutable list parameter
@Composable
fun ItemList(items: MutableList<Item>, modifier: Modifier = Modifier) { ... }
```

Mark custom classes that the compiler can't infer as stable:

```kotlin
@Stable
data class HomeUiState(
    val isLoading: Boolean = false,
    val items: List<Item> = emptyList(),
    val errorMessage: String? = null
)

// For interfaces used as composable parameters
@Stable
interface ItemActions {
    fun onItemClick(id: String)
    fun onItemDelete(id: String)
}
```

Use `derivedStateOf` for values computed from other state — prevents recomposition when the derived value hasn't changed:

```kotlin
// GOOD — only recomposes when isButtonEnabled actually changes
val isButtonEnabled by remember {
    derivedStateOf { uiState.name.isNotBlank() && uiState.email.isNotBlank() }
}

// BAD — recomputes inline on every parent recomposition
val isButtonEnabled = uiState.name.isNotBlank() && uiState.email.isNotBlank()
```

---

## State Management

### State Hoisting Pattern

Move state up to the lowest common ancestor that needs it:

```kotlin
// Screen level — owns the state
@Composable
fun SearchScreen(viewModel: SearchViewModel = koinViewModel()) {
    val uiState by viewModel.uiState.collectAsStateWithLifecycle()
    SearchContent(
        query = uiState.query,
        results = uiState.results,
        onQueryChange = viewModel::onQueryChange,
        onSearch = viewModel::onSearch
    )
}

// Content level — stateless, just renders
@Composable
fun SearchContent(
    query: String,
    results: List<SearchResult>,
    onQueryChange: (String) -> Unit,
    onSearch: () -> Unit,
    modifier: Modifier = Modifier
) { ... }
```

### When to Use remember vs StateFlow

| Scenario | Use |
|----------|-----|
| UI-only transient state (expanded, selected tab) | `remember { mutableStateOf(...) }` |
| Business state | `ViewModel + StateFlow` |
| Animation state | `remember { Animatable(...) }` |
| Scroll position | `rememberLazyListState()` |

### Avoiding rememberCoroutineScope Anti-Patterns

```kotlin
// BAD — launching coroutines from composable
@Composable
fun MyScreen() {
    val scope = rememberCoroutineScope()
    Button(onClick = { scope.launch { fetchData() } }) { ... }
}

// GOOD — delegate to ViewModel
@Composable
fun MyScreen(viewModel: MyViewModel = koinViewModel()) {
    Button(onClick = viewModel::onLoadData) { ... }
}
```

---

## Material 3 Guidelines

### Theme Setup

```kotlin
// AppTheme.kt
@Composable
fun AppTheme(
    darkTheme: Boolean = isSystemInDarkTheme(),
    content: @Composable () -> Unit
) {
    val colorScheme = if (darkTheme) DarkColorScheme else LightColorScheme

    MaterialTheme(
        colorScheme = colorScheme,
        typography = AppTypography,
        content = content
    )
}

private val DarkColorScheme = darkColorScheme(
    primary = ColorPalette.Primary,
    secondary = ColorPalette.Secondary,
    background = ColorPalette.BackgroundDark,
    surface = ColorPalette.SurfaceDark,
    onPrimary = Color.White,
    onBackground = ColorPalette.OnBackgroundDark
)
```

### Color Usage

```kotlin
// GOOD — use MaterialTheme tokens
Text(
    text = "Hello",
    color = MaterialTheme.colorScheme.onBackground
)

// BAD — hardcoded colors
Text(
    text = "Hello",
    color = Color(0xFF1A1A1A)
)
```

### Typography Usage

```kotlin
// GOOD — use typography scale
Text(text = "Title", style = MaterialTheme.typography.headlineMedium)
Text(text = "Body", style = MaterialTheme.typography.bodyLarge)

// BAD — hardcoded text style
Text(text = "Title", fontSize = 24.sp, fontWeight = FontWeight.Bold)
```

---

## Layouts

### Adaptive Layouts

Support different screen sizes:

```kotlin
@Composable
fun AdaptiveHomeScreen(
    uiState: HomeUiState,
    windowSizeClass: WindowSizeClass,
    modifier: Modifier = Modifier
) {
    when (windowSizeClass.widthSizeClass) {
        WindowWidthSizeClass.Compact -> HomeCompact(uiState, modifier)
        WindowWidthSizeClass.Medium -> HomeMedium(uiState, modifier)
        WindowWidthSizeClass.Expanded -> HomeExpanded(uiState, modifier)
    }
}
```

### Lazy Lists

```kotlin
// Prefer LazyColumn/LazyRow for long lists
LazyColumn(
    contentPadding = PaddingValues(16.dp),
    verticalArrangement = Arrangement.spacedBy(8.dp)
) {
    items(
        items = uiState.items,
        key = { item -> item.id }  // always provide stable keys
    ) { item ->
        ItemCard(item = item)
    }
}
```

---

## Resources

### String Resources (Compose Multiplatform)

Define strings in `commonMain/composeResources/values/strings.xml`:

```xml
<resources>
    <string name="app_name">My App</string>
    <string name="home_title">Welcome, %1$s</string>
</resources>
```

Use in composables:

```kotlin
import org.jetbrains.compose.resources.stringResource
import com.example.shared.generated.resources.Res
import com.example.shared.generated.resources.app_name
import com.example.shared.generated.resources.home_title

@Composable
fun HomeTitle(userName: String) {
    Text(text = stringResource(Res.string.home_title, userName))
}
```

### Image Resources

```kotlin
// In composable
Image(
    painter = painterResource(Res.drawable.logo),
    contentDescription = null
)
```

---

## Accessibility

Every composable that conveys meaning must be accessible. The Compose accessibility tree is read by TalkBack (Android) and VoiceOver (iOS).

### Content Descriptions

Always provide `contentDescription` for images and icon-only buttons. Use `null` only for purely decorative content:

```kotlin
// GOOD — meaningful image
Image(
    painter = painterResource(Res.drawable.avatar),
    contentDescription = stringResource(Res.string.user_avatar_description)
)

// GOOD — decorative image
Image(
    painter = painterResource(Res.drawable.background_pattern),
    contentDescription = null  // explicitly decorative
)

// GOOD — icon button with description
IconButton(onClick = onFavoriteClick) {
    Icon(
        imageVector = Icons.Default.Favorite,
        contentDescription = stringResource(Res.string.add_to_favorites)
    )
}
```

### Semantics Modifiers

Use `Modifier.semantics` to provide accessibility metadata beyond what Compose infers:

```kotlin
// Merge child semantics into a single accessible node
Card(
    modifier = Modifier
        .semantics(mergeDescendants = true) {}
        .clickable(onClick = onClick)
) {
    Text(text = item.title)
    Text(text = item.subtitle)
}

// Custom action for complex interactions
Box(
    modifier = Modifier.semantics {
        contentDescription = "Item: ${item.title}"
        onClick(label = "Open detail") {
            onItemClick(item.id)
            true
        }
    }
)

// State descriptions (e.g., toggle buttons)
val toggleDescription = if (isSelected) "Selected" else "Not selected"
FilterChip(
    selected = isSelected,
    onClick = onToggle,
    label = { Text(item.label) },
    modifier = Modifier.semantics {
        stateDescription = toggleDescription
    }
)
```

### `clearAndSetSemantics` — Override Inferred Semantics

Use when the default accessibility tree is noisy or misleading:

```kotlin
// Replace all child semantics with a single clean description
Row(
    modifier = Modifier.clearAndSetSemantics {
        contentDescription = "${user.name}, ${user.role}, ${if (user.isOnline) "Online" else "Offline"}"
    }
) {
    Avatar(user = user)
    Column {
        Text(user.name)
        Text(user.role)
    }
    OnlineIndicator(isOnline = user.isOnline)
}
```

### Minimum Touch Target Size

Material 3 enforces 48dp minimum touch targets. For smaller visual elements, use padding to expand the touch area:

```kotlin
Icon(
    imageVector = Icons.Default.Close,
    contentDescription = "Dismiss",
    modifier = Modifier
        .size(24.dp)
        .padding(12.dp)  // expands touch target to 48dp
        .clickable(onClick = onDismiss)
)
```

### Heading Semantics

Mark section headers so screen readers can navigate by heading:

```kotlin
Text(
    text = "Recent Activity",
    style = MaterialTheme.typography.titleMedium,
    modifier = Modifier.semantics { heading() }
)
```

---

## Focus Management

### FocusRequester — Programmatic Focus

Use `FocusRequester` to direct keyboard focus to a field automatically (e.g., on screen open or after form error):

```kotlin
@Composable
fun LoginForm(onSubmit: (String, String) -> Unit) {
    val emailFocusRequester = remember { FocusRequester() }
    val passwordFocusRequester = remember { FocusRequester() }
    val focusManager = LocalFocusManager.current

    var email by remember { mutableStateOf("") }
    var password by remember { mutableStateOf("") }

    Column {
        OutlinedTextField(
            value = email,
            onValueChange = { email = it },
            label = { Text("Email") },
            keyboardOptions = KeyboardOptions(
                keyboardType = KeyboardType.Email,
                imeAction = ImeAction.Next
            ),
            keyboardActions = KeyboardActions(
                onNext = { passwordFocusRequester.requestFocus() }
            ),
            modifier = Modifier.focusRequester(emailFocusRequester)
        )

        OutlinedTextField(
            value = password,
            onValueChange = { password = it },
            label = { Text("Password") },
            visualTransformation = PasswordVisualTransformation(),
            keyboardOptions = KeyboardOptions(
                keyboardType = KeyboardType.Password,
                imeAction = ImeAction.Done
            ),
            keyboardActions = KeyboardActions(
                onDone = {
                    focusManager.clearFocus()
                    onSubmit(email, password)
                }
            ),
            modifier = Modifier.focusRequester(passwordFocusRequester)
        )
    }

    // Auto-focus email on screen open
    LaunchedEffect(Unit) {
        emailFocusRequester.requestFocus()
    }
}
```

### Text Field Accessibility

Every `TextField`/`OutlinedTextField` must have:
- A visible `label` OR a `semantics { contentDescription }` for screen readers
- Correct `keyboardType` so the right keyboard appears (do not use `Text` keyboard for emails)
- `imeAction` matching the field's role (`Next` for multi-field forms, `Done`/`Search` for last field)
- Error state announced to accessibility services via `isError` + `supportingText`:

```kotlin
OutlinedTextField(
    value = email,
    onValueChange = { email = it },
    label = { Text("Email address") },
    isError = emailError != null,
    supportingText = emailError?.let { { Text(it, color = MaterialTheme.colorScheme.error) } },
    keyboardOptions = KeyboardOptions(keyboardType = KeyboardType.Email),
    modifier = Modifier
        .fillMaxWidth()
        .semantics {
            if (emailError != null) error(emailError)  // announces error to TalkBack
        }
)
```

### Focus Order

Override the default focus traversal order when the visual layout doesn't match the logical reading order:

```kotlin
// Explicit focus order for a custom layout
Box {
    TextField(
        modifier = Modifier.focusProperties { next = secondFieldFocusRequester }
    )
    TextField(
        modifier = Modifier.focusProperties { previous = firstFieldFocusRequester }
    )
}
```

---

## Dynamic Type / Font Scaling

Do not hardcode `sp` sizes that break at large text scale settings. Allow the system font scale to apply:

```kotlin
// GOOD — uses TextUnit.Unspecified so MaterialTheme scale applies
Text(
    text = content,
    style = MaterialTheme.typography.bodyLarge  // scales with system font size
)

// BAD — ignores user's accessibility font size preference
Text(
    text = content,
    fontSize = 16.sp,   // fixed size, still scales with SP by default
    lineHeight = 16.sp  // hardcoded line height prevents proper scaling
)
```

Test for large font scales:
```kotlin
// Set font scale in Compose preview
@Preview(fontScale = 1.5f)
@Composable
private fun HomeScreenPreview_LargeFont() {
    AppTheme { HomeContent(uiState = HomeUiState(), onAction = {}) }
}

@Preview(fontScale = 2.0f)
@Composable
private fun HomeScreenPreview_ExtraLargeFont() {
    AppTheme { HomeContent(uiState = HomeUiState(), onAction = {}) }
}
```

For containers that must not expand with text (e.g., a fixed-height chip), use `nonScaledSp`:

```kotlin
// Convert dp → sp without scaling — use sparingly, only for layout-critical sizes
val nonScaledTextSize = with(LocalDensity.current) { 12.dp.toSp() }
```

---

## Performance

### Key Performance Rules

1. **Avoid unnecessary recomposition** — use `remember`, `derivedStateOf`, and `key` parameters
2. **Use `key()` in loops** — stable keys prevent full list recomposition
3. **Avoid heavy work in composition** — use `LaunchedEffect` or ViewModel for side effects
4. **Use `derivedStateOf`** for computed state that depends on other state:

```kotlin
val isButtonEnabled by remember {
    derivedStateOf { uiState.name.isNotBlank() && uiState.email.isNotBlank() }
}
```

5. **Profile with Layout Inspector** — identify recomposition counts

---

## Previews

Always provide at minimum: light + dark previews.

```kotlin
@Preview
@Composable
private fun HomeScreenPreview_Light() {
    AppTheme(darkTheme = false) {
        HomeContent(
            uiState = HomeUiState(
                isLoading = false,
                title = "Preview Title",
                items = PreviewData.items
            ),
            onAction = {}
        )
    }
}

@Preview
@Composable
private fun HomeScreenPreview_Dark() {
    AppTheme(darkTheme = true) {
        HomeContent(
            uiState = HomeUiState.Loading,
            onAction = {}
        )
    }
}

// Preview data object (never in production code)
private object PreviewData {
    val items = List(5) { Item(id = "$it", title = "Item $it") }
}
```

---

## Official Resources

- [Compose Architecture](https://developer.android.com/develop/ui/compose/architecture)
- [Compose State Guide](https://developer.android.com/develop/ui/compose/state)
- [Compose Layouts](https://developer.android.com/develop/ui/compose/layouts)
- [Compose Performance](https://developer.android.com/develop/ui/compose/performance)
- [Compose Multiplatform Resources](https://www.jetbrains.com/help/kotlin-multiplatform-dev/compose-multiplatform-resources-usage.html)
- [Material 3 Design](https://m3.material.io/)
- [Material 3 in Compose](https://developer.android.com/develop/ui/compose/designsystems/material3)
