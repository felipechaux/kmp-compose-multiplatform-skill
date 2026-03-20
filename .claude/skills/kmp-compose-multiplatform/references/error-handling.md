# Error Handling — KMP + Compose Multiplatform

---

## Domain Error Types

Never use raw `String` for errors. Define a structured domain error hierarchy:

```kotlin
// core/util/AppError.kt — commonMain
sealed class AppError : Throwable() {

    // Network errors
    sealed class Network : AppError() {
        data object NoConnection : Network()
        data object Timeout : Network()
        data class ServerError(val code: Int, val body: String?) : Network()
        data class Unauthorized(val message: String = "Session expired") : Network()
        data object Unknown : Network()
    }

    // Local persistence errors
    sealed class Database : AppError() {
        data class ReadFailure(override val cause: Throwable?) : Database()
        data class WriteFailure(override val cause: Throwable?) : Database()
        data object NotFound : Database()
    }

    // Business/domain validation errors
    sealed class Validation : AppError() {
        data class InvalidInput(val field: String, val reason: String) : Validation()
        data object RequiredFieldMissing : Validation()
    }

    // Unknown/unexpected
    data class Unexpected(override val cause: Throwable?) : AppError()
}
```

---

## Resource Sealed Class (with typed errors)

```kotlin
// core/util/Resource.kt — commonMain
sealed class Resource<out T> {
    data class Success<out T>(val data: T) : Resource<T>()
    data class Error(val error: AppError) : Resource<Nothing>()
    data object Loading : Resource<Nothing>()
}

// Extension functions for ergonomic handling
inline fun <T> Resource<T>.onSuccess(action: (T) -> Unit): Resource<T> {
    if (this is Resource.Success) action(data)
    return this
}

inline fun <T> Resource<T>.onError(action: (AppError) -> Unit): Resource<T> {
    if (this is Resource.Error) action(error)
    return this
}

inline fun <T> Resource<T>.onLoading(action: () -> Unit): Resource<T> {
    if (this is Resource.Loading) action()
    return this
}

inline fun <T, R> Resource<T>.map(transform: (T) -> R): Resource<R> = when (this) {
    is Resource.Success -> Resource.Success(transform(data))
    is Resource.Error -> this
    is Resource.Loading -> this
}
```

---

## Ktor Error Mapping

Map HTTP and network exceptions to domain errors at the data layer boundary — never let `ClientRequestException` or `IOException` escape into domain/presentation:

```kotlin
// core/network/NetworkErrorMapper.kt — commonMain
import io.ktor.client.plugins.*
import io.ktor.http.*

suspend fun <T> safeApiCall(call: suspend () -> T): Resource<T> {
    return try {
        Resource.Success(call())
    } catch (e: ClientRequestException) {
        val error = when (e.response.status) {
            HttpStatusCode.Unauthorized -> AppError.Network.Unauthorized()
            HttpStatusCode.NotFound -> AppError.Database.NotFound
            else -> AppError.Network.ServerError(
                code = e.response.status.value,
                body = e.response.toString()
            )
        }
        Resource.Error(error)
    } catch (e: ServerResponseException) {
        Resource.Error(AppError.Network.ServerError(e.response.status.value, null))
    } catch (e: HttpRequestTimeoutException) {
        Resource.Error(AppError.Network.Timeout)
    } catch (e: Exception) {
        if (e.message?.contains("Unable to resolve host") == true ||
            e.message?.contains("Network is unreachable") == true) {
            Resource.Error(AppError.Network.NoConnection)
        } else {
            Resource.Error(AppError.Unexpected(e))
        }
    }
}
```

Usage in repository:

```kotlin
class UserRepositoryImpl(private val api: UserApiService) : UserRepository {

    override suspend fun getUser(id: String): Resource<User> =
        safeApiCall { api.getUser(id) }.map { it.toDomain() }
}
```

---

## Room Error Mapping

```kotlin
// Wrap all database calls similarly
suspend fun <T> safeDbCall(call: suspend () -> T): Resource<T> {
    return try {
        Resource.Success(call())
    } catch (e: Exception) {
        Resource.Error(AppError.Database.ReadFailure(e))
    }
}
```

---

## Error Handling in ViewModel

Map domain errors to user-facing messages at the presentation layer:

```kotlin
// Presentation layer — never expose AppError directly to the UI
data class HomeUiState(
    val isLoading: Boolean = false,
    val items: List<Item> = emptyList(),
    val errorMessage: String? = null   // human-readable, never AppError
)

class HomeViewModel(private val getItemsUseCase: GetItemsUseCase) : ViewModel() {

    private val _uiState = MutableStateFlow(HomeUiState())
    val uiState: StateFlow<HomeUiState> = _uiState.asStateFlow()

    fun loadItems() {
        viewModelScope.launch {
            _uiState.update { it.copy(isLoading = true, errorMessage = null) }
            getItemsUseCase().onSuccess { items ->
                _uiState.update { it.copy(isLoading = false, items = items) }
            }.onError { error ->
                _uiState.update { it.copy(isLoading = false, errorMessage = error.toUserMessage()) }
            }
        }
    }
}

// Extension — localize error strings here, not in the domain
fun AppError.toUserMessage(): String = when (this) {
    is AppError.Network.NoConnection -> "No internet connection. Please check your network."
    is AppError.Network.Timeout -> "Request timed out. Please try again."
    is AppError.Network.Unauthorized -> "Your session has expired. Please log in again."
    is AppError.Network.ServerError -> "Server error ($code). Please try again later."
    is AppError.Database.NotFound -> "Item not found."
    is AppError.Validation.InvalidInput -> "Invalid $field: $reason"
    else -> "Something went wrong. Please try again."
}
```

---

## Error Handling in Compose UI

```kotlin
@Composable
fun HomeScreen(viewModel: HomeViewModel = koinViewModel()) {
    val uiState by viewModel.uiState.collectAsStateWithLifecycle()

    HomeContent(
        uiState = uiState,
        onRetry = viewModel::loadItems
    )
}

@Composable
fun HomeContent(
    uiState: HomeUiState,
    onRetry: () -> Unit,
    modifier: Modifier = Modifier
) {
    Box(modifier = modifier.fillMaxSize()) {
        // Content
        if (uiState.items.isNotEmpty()) {
            ItemList(items = uiState.items)
        }

        // Loading overlay
        if (uiState.isLoading) {
            CircularProgressIndicator(modifier = Modifier.align(Alignment.Center))
        }

        // Error state
        uiState.errorMessage?.let { message ->
            ErrorBanner(
                message = message,
                onRetry = onRetry,
                modifier = Modifier.align(Alignment.BottomCenter)
            )
        }
    }
}
```

---

## Recoverable vs Fatal Error Classification

Not all errors are equal — classify them to drive the right UI response:

```kotlin
// Extend AppError with recoverability metadata
val AppError.isRecoverable: Boolean get() = when (this) {
    is AppError.Network.NoConnection -> true    // user can re-enable wifi
    is AppError.Network.Timeout -> true         // user can retry
    is AppError.Network.ServerError -> code in 500..599  // server-side, worth retrying
    is AppError.Network.Unauthorized -> false   // must re-authenticate
    is AppError.Database.ReadFailure -> false   // data corruption — escalate
    is AppError.Database.WriteFailure -> true   // may succeed on retry
    is AppError.Database.NotFound -> false      // no point retrying
    is AppError.Validation -> false             // user input issue — don't retry automatically
    is AppError.Unexpected -> false             // unknown — treat as fatal
    else -> false
}

val AppError.isFatal: Boolean get() = !isRecoverable
```

Use `isFatal` to decide whether to show a retry button or navigate to an error screen:

```kotlin
fun AppError.toUiAction(): ErrorAction = when {
    isRecoverable -> ErrorAction.ShowRetry
    this is AppError.Network.Unauthorized -> ErrorAction.NavigateToLogin
    isFatal -> ErrorAction.ShowFatalDialog
    else -> ErrorAction.ShowRetry
}

sealed class ErrorAction {
    data object ShowRetry : ErrorAction()
    data object NavigateToLogin : ErrorAction()
    data object ShowFatalDialog : ErrorAction()
}
```

---

## 429 Rate Limiting

Handle `429 Too Many Requests` separately from other server errors — back off and retry after the `Retry-After` header:

```kotlin
suspend fun <T> safeApiCall(call: suspend () -> T): Resource<T> {
    return try {
        Resource.Success(call())
    } catch (e: ClientRequestException) {
        val error = when (e.response.status) {
            HttpStatusCode.Unauthorized -> AppError.Network.Unauthorized()
            HttpStatusCode.TooManyRequests -> {
                // Respect Retry-After header if present
                val retryAfter = e.response.headers["Retry-After"]?.toLongOrNull() ?: 60L
                delay(retryAfter * 1000L)
                return safeApiCall(call)  // single retry after back-off
            }
            HttpStatusCode.NotFound -> AppError.Database.NotFound
            else -> AppError.Network.ServerError(e.response.status.value, e.response.toString())
        }
        Resource.Error(error)
    }
    // ... other catch blocks
}
```

In Ktor `HttpRequestRetry`, exclude 429 from default server-error retry (handle it manually above):

```kotlin
install(HttpRequestRetry) {
    retryIf(maxRetries = 3) { _, response ->
        response.status.value in 500..599 && response.status != HttpStatusCode.TooManyRequests
    }
    exponentialDelay(base = 2.0, maxDelayMs = 10_000, randomizationMs = 500)
}
```

---

## Error Analytics and Breadcrumbs

Record non-fatal errors and add contextual breadcrumbs before sending to crash services:

```kotlin
interface ErrorReporter {
    fun recordError(error: AppError, context: Map<String, String> = emptyMap())
    fun addBreadcrumb(message: String, category: String = "app")
}

// androidMain
class FirebaseErrorReporter : ErrorReporter {
    private val crashlytics = FirebaseCrashlytics.getInstance()

    override fun recordError(error: AppError, context: Map<String, String>) {
        // Attach context as custom keys — visible in Crashlytics dashboard
        context.forEach { (key, value) -> crashlytics.setCustomKey(key, value) }
        crashlytics.setCustomKey("error_type", error::class.simpleName ?: "Unknown")
        if (error is AppError.Network.ServerError) {
            crashlytics.setCustomKey("http_code", error.code)
        }
        if (error.isFatal) {
            crashlytics.recordException(error)
        } else {
            crashlytics.log("Non-fatal error: ${error::class.simpleName}")
        }
    }

    override fun addBreadcrumb(message: String, category: String) {
        crashlytics.log("[$category] $message")
    }
}
```

Use in the ViewModel — record before mapping to user message:

```kotlin
fun loadItems() {
    viewModelScope.launch {
        errorReporter.addBreadcrumb("Loading items", "home")
        getItemsUseCase().onError { error ->
            errorReporter.recordError(error, mapOf(
                "screen" to "home",
                "action" to "loadItems"
            ))
            _uiState.update { it.copy(errorMessage = error.toUserMessage()) }
        }
    }
}
```

---

## Retry Logic

For use cases that should support retry (e.g., network-dependent operations):

```kotlin
// domain/util/RetryPolicy.kt — commonMain
suspend fun <T> withRetry(
    times: Int = 3,
    initialDelay: Long = 1000L,
    factor: Double = 2.0,
    block: suspend () -> Resource<T>
): Resource<T> {
    var currentDelay = initialDelay
    repeat(times - 1) {
        val result = block()
        if (result is Resource.Success || result is Resource.Error &&
            result.error !is AppError.Network.NoConnection &&
            result.error !is AppError.Network.Timeout) {
            return result
        }
        delay(currentDelay)
        currentDelay = (currentDelay * factor).toLong()
    }
    return block()
}
```
