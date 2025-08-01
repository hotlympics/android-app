# Project Architecture Overview

Technical architecture and design patterns for the Hotlympics Android application.

## Architecture Pattern

### MVVM with Repository Pattern
```
UI Layer (Compose) ↔ ViewModel ↔ Repository ↔ Data Sources
```

**Benefits:**
- Clear separation of concerns
- Testable business logic
- Reactive data flow
- State management with Compose

## Module Structure

```
app/
├── data/              # Repository implementations, API clients
├── domain/            # Business logic, use cases, entities
├── presentation/      # ViewModels, UI composables, state
├── di/               # Dependency injection modules
└── utils/            # Utilities, extensions, helpers
```

### Data Layer
- **Repositories**: Abstract data access
- **Data Sources**: Firebase, local storage, preferences
- **Models**: API responses, database entities
- **Network**: Retrofit clients, interceptors

### Domain Layer
- **Use Cases**: Business logic operations
- **Entities**: Core business models
- **Repositories**: Abstract interfaces

### Presentation Layer
- **ViewModels**: UI state management
- **Composables**: UI components
- **State**: UI state classes
- **Navigation**: Screen routing

## Technology Stack

### Core Framework
- **Language**: Kotlin
- **UI**: Jetpack Compose + Material 3
- **Architecture**: MVVM + Repository pattern
- **DI**: Dagger Hilt
- **Async**: Coroutines + Flow

### Data & Networking
- **HTTP**: Retrofit + OkHttp
- **Serialization**: Kotlinx Serialization
- **Local DB**: Room
- **Preferences**: DataStore
- **Image Loading**: Coil

### Camera & ML
- **Camera**: CameraX
- **ML**: ML Kit + AndroidX Camera ML Kit Vision
- **Image Processing**: Android Camera2 API
- **Biometrics**: AndroidX Biometric

### Testing
- **Unit**: JUnit 5 + MockK
- **UI**: Compose Testing + Roborazzi
- **Integration**: AndroidX Test

## Data Flow

### Photo Upload Flow
```
Camera Capture → ML Verification → Biometric Auth → 
Repository → API Client → Firebase Storage
```

### User Authentication
```
Firebase Auth → Repository → ViewModel → UI State Update
```

### Photo Rating
```
API Fetch → Repository Cache → ViewModel → Compose UI
```

## State Management

### UI State with Compose
```kotlin
@Stable
data class PhotoCaptureUiState(
    val isLoading: Boolean = false,
    val capturedPhoto: Uri? = null,
    val verificationResult: VerificationResult? = null,
    val error: String? = null
)

class PhotoCaptureViewModel @Inject constructor(
    private val photoRepository: PhotoRepository,
    private val verificationUseCase: VerifyPhotoUseCase
) : ViewModel() {
    
    private val _uiState = MutableStateFlow(PhotoCaptureUiState())
    val uiState: StateFlow<PhotoCaptureUiState> = _uiState.asStateFlow()
    
    fun capturePhoto(imageProxy: ImageProxy) {
        viewModelScope.launch {
            _uiState.update { it.copy(isLoading = true) }
            
            val result = verificationUseCase(imageProxy)
            _uiState.update { 
                it.copy(
                    isLoading = false,
                    verificationResult = result
                )
            }
        }
    }
}
```

## Dependency Injection

### Hilt Modules
```kotlin
@Module
@InstallIn(SingletonComponent::class)
object NetworkModule {
    
    @Provides
    @Singleton
    fun provideRetrofit(): Retrofit = Retrofit.Builder()
        .baseUrl(BuildConfig.API_BASE_URL)
        .addConverterFactory(KotlinxSerializationConverterFactory.create())
        .build()
    
    @Provides
    @Singleton
    fun providePhotoApiService(retrofit: Retrofit): PhotoApiService =
        retrofit.create(PhotoApiService::class.java)
}

@Module
@InstallIn(ViewModelComponent::class)
object ViewModelModule {
    
    @Provides
    fun providePhotoRepository(
        apiService: PhotoApiService,
        localDataSource: PhotoLocalDataSource
    ): PhotoRepository = PhotoRepositoryImpl(apiService, localDataSource)
}
```

## Navigation Architecture

### Compose Navigation
```kotlin
@Composable
fun HotlympicsNavHost(
    navController: NavHostController,
    startDestination: String = "auth"
) {
    NavHost(
        navController = navController,
        startDestination = startDestination
    ) {
        composable("auth") {
            AuthScreen(
                onAuthSuccess = { navController.navigate("main") }
            )
        }
        
        composable("main") {
            MainScreen(
                onNavigateToCamera = { navController.navigate("camera") }
            )
        }
        
        composable("camera") {
            PhotoCaptureScreen(
                onPhotoUploaded = { navController.popBackStack() }
            )
        }
        
        composable("rating") {
            PhotoRatingScreen()
        }
    }
}
```

## Security Architecture

### Authentication Flow
```kotlin
class AuthRepository @Inject constructor(
    private val firebaseAuth: FirebaseAuth,
    private val biometricManager: BiometricManager
) {
    
    suspend fun authenticateUser(): AuthResult {
        return when {
            biometricManager.canAuthenticate() == BIOMETRIC_SUCCESS -> {
                authenticateWithBiometric()
            }
            else -> authenticateWithFirebase()
        }
    }
    
    private suspend fun authenticateWithBiometric(): AuthResult {
        // Biometric authentication implementation
    }
}
```

### Photo Verification
```kotlin
class PhotoVerificationUseCase @Inject constructor(
    private val faceDetector: FaceDetector,
    private val qualityAnalyzer: PhotoQualityAnalyzer
) {
    
    suspend operator fun invoke(imageProxy: ImageProxy): VerificationResult {
        val faces = faceDetector.detect(imageProxy)
        val quality = qualityAnalyzer.analyze(imageProxy)
        
        return VerificationResult(
            hasFace = faces.isNotEmpty(),
            faceCount = faces.size,
            quality = quality,
            isValid = faces.size == 1 && quality.score > 0.7f
        )
    }
}
```

## Error Handling

### Repository Pattern
```kotlin
sealed class Result<out T> {
    data class Success<T>(val data: T) : Result<T>()
    data class Error(val exception: Throwable) : Result<Nothing>()
    object Loading : Result<Nothing>()
}

class PhotoRepositoryImpl : PhotoRepository {
    
    override suspend fun uploadPhoto(photo: Photo): Result<UploadResponse> {
        return try {
            val response = apiService.uploadPhoto(photo)
            Result.Success(response)
        } catch (exception: Exception) {
            Result.Error(exception)
        }
    }
}
```

### ViewModel Error Handling
```kotlin
class PhotoCaptureViewModel : ViewModel() {
    
    fun uploadPhoto(photo: Photo) {
        viewModelScope.launch {
            photoRepository.uploadPhoto(photo)
                .onSuccess { response ->
                    _uiState.update { it.copy(uploadSuccess = true) }
                }
                .onFailure { error ->
                    _uiState.update { 
                        it.copy(error = error.message ?: "Upload failed") 
                    }
                }
        }
    }
}
```

## Performance Considerations

### Image Processing
- Use `ImageProxy` for efficient memory management
- Implement proper lifecycle management for camera resources
- Cache ML Kit detection results when appropriate
- Use background dispatchers for heavy operations

### UI Performance
- Minimize recomposition with stable state classes
- Use `LazyColumn` for large lists
- Implement proper image loading with Coil
- Monitor performance with Compose compiler metrics

### Memory Management
```kotlin
class PhotoCaptureViewModel : ViewModel() {
    
    override fun onCleared() {
        super.onCleared()
        // Clean up camera resources
        cameraProvider?.unbindAll()
        imageAnalyzer?.clearAnalyzer()
    }
}
```

## Testing Strategy

### Unit Testing
- Repository implementations
- Use case business logic
- ViewModel state management
- Utility functions

### Integration Testing
- API client implementations
- Database operations
- End-to-end user flows

### UI Testing
- Compose screen testing
- Navigation testing
- User interaction testing
- Screenshot regression testing

---

*For implementation details, see the [development](../development/) documentation.*