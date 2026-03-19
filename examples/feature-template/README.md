# Feature Template

Complete scaffold for a new KMP feature module. Copy this template for each new feature.

## Directory Structure

```
feature/
└── [feature-name]/
    ├── data/
    │   ├── local/
    │   │   ├── dao/FeatureDao.kt
    │   │   └── entity/FeatureEntity.kt
    │   ├── remote/
    │   │   ├── FeatureApiService.kt
    │   │   └── dto/FeatureDto.kt
    │   ├── repository/FeatureRepositoryImpl.kt
    │   └── mapper/FeatureMapper.kt
    ├── domain/
    │   ├── model/FeatureModel.kt
    │   ├── repository/FeatureRepository.kt
    │   └── usecase/
    │       ├── GetFeatureUseCase.kt
    │       └── UpdateFeatureUseCase.kt
    ├── presentation/
    │   ├── ui/
    │   │   ├── FeatureScreen.kt
    │   │   └── components/FeatureCard.kt
    │   ├── viewmodel/FeatureViewModel.kt
    │   └── state/FeatureUiState.kt
    └── di/FeatureModule.kt
```

## Code Templates

### Domain Model

```kotlin
// feature/[name]/domain/model/FeatureModel.kt
data class FeatureModel(
    val id: String,
    val title: String,
    val createdAt: LocalDateTime
)
```

### Repository Interface

```kotlin
// feature/[name]/domain/repository/FeatureRepository.kt
interface FeatureRepository {
    suspend fun getById(id: String): Resource<FeatureModel>
    fun observeAll(): Flow<List<FeatureModel>>
    suspend fun save(model: FeatureModel): Resource<Unit>
}
```

### Use Case

```kotlin
// feature/[name]/domain/usecase/GetFeatureUseCase.kt
class GetFeatureUseCase(private val repository: FeatureRepository) {
    suspend operator fun invoke(id: String): Resource<FeatureModel> =
        repository.getById(id)
}
```

### Repository Implementation

```kotlin
// feature/[name]/data/repository/FeatureRepositoryImpl.kt
class FeatureRepositoryImpl(
    private val apiService: FeatureApiService,
    private val dao: FeatureDao
) : FeatureRepository {

    override suspend fun getById(id: String): Resource<FeatureModel> = try {
        val entity = dao.getById(id) ?: apiService.fetch(id).also { dto ->
            dao.insert(dto.toEntity())
        }.toDomain()
        Resource.Success(entity.toDomain())
    } catch (e: Exception) {
        Resource.Error(e.message ?: "Unknown error", e)
    }

    override fun observeAll(): Flow<List<FeatureModel>> =
        dao.observeAll().map { entities -> entities.map { it.toDomain() } }

    override suspend fun save(model: FeatureModel): Resource<Unit> = try {
        dao.insert(model.toEntity())
        Resource.Success(Unit)
    } catch (e: Exception) {
        Resource.Error(e.message ?: "Save failed", e)
    }
}
```

### UiState

```kotlin
// feature/[name]/presentation/state/FeatureUiState.kt
data class FeatureUiState(
    val isLoading: Boolean = false,
    val items: List<FeatureModel> = emptyList(),
    val error: String? = null,
    val selectedItem: FeatureModel? = null
)

sealed class FeatureEvent {
    data class SelectItem(val id: String) : FeatureEvent()
    data object Refresh : FeatureEvent()
    data class ShowError(val message: String) : FeatureEvent()
}
```

### ViewModel

```kotlin
// feature/[name]/presentation/viewmodel/FeatureViewModel.kt
class FeatureViewModel(
    private val getFeatureUseCase: GetFeatureUseCase,
    private val observeFeaturesUseCase: ObserveFeaturesUseCase
) : ViewModel() {

    private val _uiState = MutableStateFlow(FeatureUiState())
    val uiState: StateFlow<FeatureUiState> = _uiState.asStateFlow()

    init {
        observeItems()
    }

    private fun observeItems() {
        viewModelScope.launch {
            _uiState.update { it.copy(isLoading = true) }
            observeFeaturesUseCase()
                .catch { e -> _uiState.update { it.copy(error = e.message, isLoading = false) } }
                .collect { items -> _uiState.update { it.copy(items = items, isLoading = false) } }
        }
    }

    fun onItemSelected(id: String) {
        viewModelScope.launch {
            when (val result = getFeatureUseCase(id)) {
                is Resource.Success -> _uiState.update { it.copy(selectedItem = result.data) }
                is Resource.Error -> _uiState.update { it.copy(error = result.message) }
                is Resource.Loading -> Unit
            }
        }
    }

    fun onErrorDismissed() {
        _uiState.update { it.copy(error = null) }
    }
}
```

### Screen Composable

```kotlin
// feature/[name]/presentation/ui/FeatureScreen.kt
@Composable
fun FeatureScreen(
    onNavigateToDetail: (String) -> Unit,
    viewModel: FeatureViewModel = koinViewModel()
) {
    val uiState by viewModel.uiState.collectAsStateWithLifecycle()

    FeatureContent(
        uiState = uiState,
        onItemClick = { id ->
            viewModel.onItemSelected(id)
            onNavigateToDetail(id)
        },
        onErrorDismiss = viewModel::onErrorDismissed
    )
}

@Composable
internal fun FeatureContent(
    uiState: FeatureUiState,
    onItemClick: (String) -> Unit,
    onErrorDismiss: () -> Unit,
    modifier: Modifier = Modifier
) {
    Box(modifier = modifier.fillMaxSize()) {
        when {
            uiState.isLoading -> CircularProgressIndicator(Modifier.align(Alignment.Center))
            else -> LazyColumn {
                items(uiState.items, key = { it.id }) { item ->
                    FeatureCard(item = item, onClick = { onItemClick(item.id) })
                }
            }
        }

        uiState.error?.let { error ->
            Snackbar(
                action = { TextButton(onClick = onErrorDismiss) { Text("Dismiss") } },
                modifier = Modifier.align(Alignment.BottomCenter)
            ) { Text(error) }
        }
    }
}
```

### Koin Module

```kotlin
// feature/[name]/di/FeatureModule.kt
val featureModule = module {
    // Data
    single { FeatureApiService(get()) }
    single<FeatureRepository> { FeatureRepositoryImpl(get(), get()) }

    // Domain
    factory { GetFeatureUseCase(get()) }
    factory { ObserveFeaturesUseCase(get()) }

    // Presentation
    viewModel { FeatureViewModel(get(), get()) }
}
```
