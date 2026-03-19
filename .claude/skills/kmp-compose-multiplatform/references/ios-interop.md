# iOS / Swift Interop — KMP + Compose Multiplatform

References: [KMP iOS Integration](https://www.jetbrains.com/help/kotlin-multiplatform-dev/multiplatform-ios-integration-overview.html) | [Kotlin-Swift Interop](https://www.jetbrains.com/help/kotlin-multiplatform-dev/multiplatform-ios-integration-overview.html)

---

## Kotlin → Swift Naming Conventions

| Kotlin | Swift | Notes |
|--------|-------|-------|
| `class MyClass` | `MyClass` | Direct mapping |
| Top-level `fun myFun()` | `MyClassKt.myFun()` | File name becomes prefix |
| Top-level `val myVal` | `MyClassKt.myVal` | Same rule as functions |
| `object MySingleton` | `MySingleton.shared` | Kotlin objects → `.shared` |
| `companion object` | Direct on class | No `.companion` needed |
| `suspend fun` | `async` (with KMP adapters) | Needs `@JvmStatic` or wrapper |
| `sealed class` | Protocol + classes | Requires careful mapping |

### Top-level function access example

```kotlin
// iosMain/MainViewController.kt
fun MainViewController(): UIViewController = ComposeUIViewController { ... }
```

```swift
// ContentView.swift — note the "Kt" suffix for top-level functions
MainViewControllerKt.MainViewController()
```

---

## Nullability Bridging

Kotlin's null-safety maps directly to Swift optionals:

| Kotlin | Swift |
|--------|-------|
| `String` (non-null) | `String` |
| `String?` (nullable) | `String?` |
| `List<String>` | `[String]` |
| `List<String?>` | `[String?]` |

**Important:** Kotlin's `Unit` becomes `KotlinUnit` in Swift. Avoid exposing functions returning `Unit` to Swift — use `Void` via wrapper functions when needed.

```kotlin
// Avoid this in public iOS API
fun doSomething(): Unit { ... }

// Prefer void-like callbacks wrapped in expect/actual or explicit wrappers
```

---

## Collection Bridging

Kotlin collections are bridged to Swift, but **mutability is lost**:

```swift
// Kotlin List<String> → Swift [String] (read-only copy)
let items: [String] = kotlinObject.items as! [String]

// Kotlin Map<String, Int> → Swift [String: Int]
let map: [String: Int] = kotlinObject.config as! [String: Int]
```

Never pass Swift `Array` or `Dictionary` directly to Kotlin functions expecting `List` or `Map`. Wrap at the boundary:

```kotlin
// iosMain — expose iOS-friendly API
fun processItems(items: Array<String>) {
    internalFunction(items.toList())
}
```

---

## Coroutines & Swift Concurrency

Kotlin `suspend` functions are **not automatically available** in Swift as `async`. Use one of these patterns:

### Option 1: SKIE (recommended)

[SKIE](https://skie.touchlab.co) auto-generates Swift-friendly async wrappers:

```toml
# libs.versions.toml
skie = "0.9.5"

[plugins]
skie = { id = "co.touchlab.skie", version.ref = "skie" }
```

```kotlin
// shared/build.gradle.kts
plugins {
    alias(libs.plugins.skie)
}
```

With SKIE, Kotlin suspend functions become Swift async:

```swift
// Without SKIE — callback-based
UserRepositoryKt.getUser(id: "1") { result, error in ... }

// With SKIE — native Swift async/await
let user = try await userRepository.getUser(id: "1")
```

### Option 2: Manual callback wrapper (no dependency)

```kotlin
// iosMain — wrap suspend functions in callbacks for Swift
fun getUser(id: String, onSuccess: (User) -> Unit, onError: (String) -> Unit) {
    CoroutineScope(Dispatchers.Main).launch {
        when (val result = userRepository.getUser(id)) {
            is Resource.Success -> onSuccess(result.data)
            is Resource.Error -> onError(result.error.toUserMessage())
            is Resource.Loading -> {}
        }
    }
}
```

### Option 3: KMP-NativeCoroutines

```toml
kmp-nativecoroutines = "1.0.0-ALPHA-35"
```

Exposes Kotlin Flows and suspend functions as Swift async sequences.

---

## Flow Exposure to Swift

Kotlin `Flow<T>` is **not directly usable** in Swift. Options:

### With SKIE (recommended)
SKIE converts `Flow<T>` to `AsyncSequence` automatically.

```swift
for await item in viewModel.uiState {
    updateUI(with: item)
}
```

### Manual StateFlow → callback

```kotlin
// iosMain
class HomeViewModelIos(private val viewModel: HomeViewModel) {

    private val scope = CoroutineScope(Dispatchers.Main + SupervisorJob())

    fun observeState(onChange: (HomeUiState) -> Unit): () -> Unit {
        val job = scope.launch {
            viewModel.uiState.collect { onChange(it) }
        }
        return { job.cancel() }
    }

    fun cancel() {
        scope.cancel()
    }
}
```

```swift
class HomeViewController: UIViewController {
    private var cancelObservation: (() -> Void)?

    override func viewDidLoad() {
        super.viewDidLoad()
        cancelObservation = viewModel.observeState { [weak self] state in
            self?.updateUI(state: state)
        }
    }

    deinit {
        cancelObservation?()
        viewModel.cancel()
    }
}
```

---

## Thread Safety

Kotlin/Native enforces strict memory isolation rules. Key rules:

1. **Never share mutable state across threads** — Kotlin/Native will throw `InvalidMutabilityException` (pre-1.7) or use the new memory model (1.7+, default)
2. **Use `@ThreadLocal`** for thread-local mutable state in `iosMain`
3. **Use `@SharedImmutable`** for constants shared across threads (deprecated in new MM — just use `val`)
4. **Main dispatcher** — Always use `Dispatchers.Main` for UI updates on iOS

```kotlin
// iosMain — ensure coroutines run on main thread for UI
actual fun platformModule(): Module = module {
    single { Dispatchers.Main }
}
```

---

## Sealed Classes in Swift

Kotlin sealed classes generate a class hierarchy in Swift, but Swift `switch` won't enforce exhaustiveness. Use SKIE or document the pattern:

```kotlin
// commonMain
sealed class AuthState {
    data object Unauthenticated : AuthState()
    data class Authenticated(val userId: String) : AuthState()
    data object Loading : AuthState()
}
```

```swift
// Without SKIE — verbose and not exhaustive
if let state = authState as? AuthStateAuthenticated {
    print(state.userId)
} else if authState is AuthStateUnauthenticated {
    showLogin()
}

// With SKIE — sealed classes become Swift enums (exhaustive)
switch authState {
case .unauthenticated: showLogin()
case .authenticated(let userId): showHome(userId: userId)
case .loading: showSpinner()
}
```

---

## Xcode Build Integration (Local Development)

To use the local KMP build in Xcode without SPM remote package:

1. Add a **Run Script build phase** to the iOS target:

```bash
cd "$SRCROOT/../.."
./gradlew :shared:assembleDebugXCFramework
```

2. Add the XCFramework output as a local framework:
   - `Build Phases → Link Binary With Libraries → Add Other → Add Files`
   - Select `shared/build/XCFrameworks/debug/shared.xcframework`

3. Set **Embed & Sign** for the framework in `Frameworks, Libraries, and Embedded Content`

**Or** use the local SPM package approach (simpler):

```swift
// Package.swift for local development (no binaryTarget)
.target(
    name: "MySharedWrapper",
    dependencies: ["MySharedBinary"]
    // points to local path, no URL/checksum needed
)
```

---

## Debugging Kotlin Code from Xcode

1. **LLDB in Xcode**: Breakpoints can be set in `.kt` files when using `embedAndSignAppleFrameworkForXcode` Gradle task
2. **Enable dSYM**: In `build.gradle.kts`:
   ```kotlin
   iosArm64 {
       binaries.framework {
           debuggable = true
           isStatic = true
       }
   }
   ```
3. **Kotlin/Native memory leak debugging**: Use the `kotlin.native.internal.GC.collect()` and monitor with Instruments

---

## Logging from iOS

Use `NSLog` or `os_log` via `expect/actual`:

```kotlin
// commonMain
expect fun logDebug(tag: String, message: String)
expect fun logError(tag: String, message: String, throwable: Throwable? = null)
```

```kotlin
// iosMain
actual fun logDebug(tag: String, message: String) {
    NSLog("[$tag] DEBUG: $message")
}

actual fun logError(tag: String, message: String, throwable: Throwable?) {
    NSLog("[$tag] ERROR: $message ${throwable?.message ?: ""}")
}
```

```kotlin
// androidMain
actual fun logDebug(tag: String, message: String) {
    Log.d(tag, message)
}

actual fun logError(tag: String, message: String, throwable: Throwable?) {
    Log.e(tag, message, throwable)
}
```

---

## Public iOS API Design Guidelines

When exposing Kotlin APIs to Swift/iOS:

1. **Keep the public API small** — only expose what iOS needs, internals stay `internal`
2. **Use `@HiddenFromObjC`** to exclude Kotlin-internal types from the Swift API
3. **Avoid generics in public API** — Kotlin generics don't bridge cleanly to Swift
4. **Prefer data classes over complex hierarchies** — easier to bridge
5. **Name iOS-facing functions clearly** — `getUser(id:)` not `fetchUserById(id:)`

```kotlin
// Mark internal Kotlin utilities as hidden from Swift
@HiddenFromObjC
internal fun internalHelper(): String = "not exposed to Swift"
```
