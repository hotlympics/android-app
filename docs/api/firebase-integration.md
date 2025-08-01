# Firebase Integration Guide

Integration documentation for Firebase services used in the Hotlympics Android app.

## Firebase Services

### Authentication
- **Google Sign-In**: Primary authentication method
- **Email/Password**: Alternative authentication
- **Biometric Enhancement**: Device-level security layer

### Firestore Database
- **Users Collection**: User profiles and settings
- **Images Collection**: Photo metadata and references
- **Ratings Collection**: ELO scores and rating history

### Cloud Storage
- **Image Storage**: Uploaded photos
- **Profile Pictures**: User avatars
- **Temporary Storage**: Verification images

## Configuration

### Firebase Setup
```kotlin
// App-level build.gradle.kts
plugins {
    id("com.google.gms.google-services")
}

dependencies {
    implementation(platform("com.google.firebase:firebase-bom:32.7.0"))
    implementation("com.google.firebase:firebase-auth-ktx")
    implementation("com.google.firebase:firebase-firestore-ktx")
    implementation("com.google.firebase:firebase-storage-ktx")
    implementation("com.google.firebase:firebase-analytics-ktx")
}
```

### Authentication Implementation
```kotlin
class FirebaseAuthRepository @Inject constructor(
    private val firebaseAuth: FirebaseAuth
) : AuthRepository {
    
    suspend fun signInWithGoogle(idToken: String): AuthResult {
        val credential = GoogleAuthProvider.getCredential(idToken, null)
        return try {
            val result = firebaseAuth.signInWithCredential(credential).await()
            AuthResult.Success(result.user)
        } catch (e: Exception) {
            AuthResult.Error(e)
        }
    }
    
    suspend fun signOut() {
        firebaseAuth.signOut()
    }
    
    fun getCurrentUser(): User? = firebaseAuth.currentUser
}
```

## Data Models

### User Document
```kotlin
@Serializable
data class FirebaseUser(
    val id: String,
    val email: String,
    val displayName: String? = null,
    val photoUrl: String? = null,
    val gender: String = "unknown",
    val dateOfBirth: Timestamp? = null,
    val uploadedImageIds: List<String> = emptyList(),
    val poolImageIds: List<String> = emptyList(),
    val rateCount: Int = 0,
    val createdAt: Timestamp = Timestamp.now(),
    val updatedAt: Timestamp = Timestamp.now()
)
```

### Image Document
```kotlin
@Serializable
data class FirebaseImage(
    val id: String,
    val userId: String,
    val fileName: String,
    val inPool: Boolean = false,
    val eloScore: Int = 1000,
    val ratingCount: Int = 0,
    val uploadedAt: Timestamp = Timestamp.now(),
    val verificationStatus: String = "pending"
)
```

## Security Rules

### Firestore Rules
```javascript
rules_version = '2';
service cloud.firestore {
  match /databases/{database}/documents {
    // Users can only access their own data
    match /users/{userId} {
      allow read, write: if request.auth != null && request.auth.uid == userId;
    }
    
    // Images can be read by anyone, but only modified by owner
    match /images/{imageId} {
      allow read: if true;
      allow write: if request.auth != null && 
        (resource == null || resource.data.userId == request.auth.uid);
    }
    
    // Ratings can be read by anyone, created by authenticated users
    match /ratings/{ratingId} {
      allow read: if true;
      allow create: if request.auth != null;
      allow update, delete: if false; // Ratings are immutable
    }
  }
}
```

### Storage Rules
```javascript
rules_version = '2';
service firebase.storage {
  match /b/{bucket}/o {
    // Images can be read by anyone
    match /images/{imageId} {
      allow read: if true;
      allow write: if request.auth != null;
    }
    
    // User profile pictures
    match /profiles/{userId}/{filename} {
      allow read: if true;
      allow write: if request.auth != null && request.auth.uid == userId;
    }
  }
}
```

## API Integration

### Firestore Repository
```kotlin
class FirestoreRepository @Inject constructor(
    private val firestore: FirebaseFirestore
) {
    
    suspend fun getUserProfile(userId: String): Result<FirebaseUser> {
        return try {
            val document = firestore.collection("users")
                .document(userId)
                .get()
                .await()
            
            val user = document.toObject<FirebaseUser>()
            if (user != null) {
                Result.Success(user)
            } else {
                Result.Error(Exception("User not found"))
            }
        } catch (e: Exception) {
            Result.Error(e)
        }
    }
    
    suspend fun updateUserProfile(user: FirebaseUser): Result<Unit> {
        return try {
            firestore.collection("users")
                .document(user.id)
                .set(user.copy(updatedAt = Timestamp.now()))
                .await()
            Result.Success(Unit)
        } catch (e: Exception) {
            Result.Error(e)
        }
    }
    
    suspend fun uploadImageMetadata(image: FirebaseImage): Result<String> {
        return try {
            val docRef = firestore.collection("images")
                .document(image.id)
            docRef.set(image).await()
            Result.Success(docRef.id)
        } catch (e: Exception) {
            Result.Error(e)
        }
    }
}
```

### Storage Repository
```kotlin
class StorageRepository @Inject constructor(
    private val storage: FirebaseStorage
) {
    
    suspend fun uploadImage(
        imageUri: Uri,
        imageId: String
    ): Result<String> {
        return try {
            val ref = storage.reference.child("images/$imageId.jpg")
            val uploadTask = ref.putFile(imageUri).await()
            val downloadUrl = uploadTask.storage.downloadUrl.await()
            Result.Success(downloadUrl.toString())
        } catch (e: Exception) {
            Result.Error(e)
        }
    }
    
    suspend fun getImageUrl(imageId: String): Result<String> {
        return try {
            val url = storage.reference
                .child("images/$imageId.jpg")
                .downloadUrl
                .await()
            Result.Success(url.toString())
        } catch (e: Exception) {
            Result.Error(e)
        }
    }
}
```

## Error Handling

### Firebase Exceptions
```kotlin
sealed class FirebaseError : Exception() {
    object NetworkError : FirebaseError()
    object AuthenticationError : FirebaseError()
    object PermissionDenied : FirebaseError()
    object DocumentNotFound : FirebaseError()
    object StorageError : FirebaseError()
    data class UnknownError(val originalException: Exception) : FirebaseError()
    
    companion object {
        fun fromException(exception: Exception): FirebaseError {
            return when (exception) {
                is FirebaseNetworkException -> NetworkError
                is FirebaseAuthException -> AuthenticationError
                is FirebaseFirestoreException -> {
                    when (exception.code) {
                        FirebaseFirestoreException.Code.PERMISSION_DENIED -> PermissionDenied
                        FirebaseFirestoreException.Code.NOT_FOUND -> DocumentNotFound
                        else -> UnknownError(exception)
                    }
                }
                is StorageException -> StorageError
                else -> UnknownError(exception)
            }
        }
    }
}
```

## Testing

### Firestore Testing
```kotlin
@RunWith(AndroidJUnit4::class)
class FirestoreRepositoryTest {
    
    @get:Rule
    val instantTaskExecutorRule = InstantTaskExecutorRule()
    
    private lateinit var firestore: FirebaseFirestore
    private lateinit var repository: FirestoreRepository
    
    @Before
    fun setup() {
        // Use Firebase emulator for testing
        firestore = Firebase.firestore
        firestore.useEmulator("10.0.2.2", 8080)
        repository = FirestoreRepository(firestore)
    }
    
    @Test
    fun testUserProfileCreation() = runTest {
        val user = FirebaseUser(
            id = "test_user",
            email = "test@example.com",
            displayName = "Test User"
        )
        
        val result = repository.updateUserProfile(user)
        assertTrue(result is Result.Success)
    }
}
```

### Authentication Testing
```kotlin
class MockAuthRepository : AuthRepository {
    private var currentUser: User? = null
    
    override suspend fun signInWithGoogle(idToken: String): AuthResult {
        return AuthResult.Success(mockUser)
    }
    
    override suspend fun signOut() {
        currentUser = null
    }
    
    override fun getCurrentUser(): User? = currentUser
}
```

## Performance Optimization

### Firestore Optimization
- Use compound queries with proper indexing
- Implement pagination for large datasets
- Cache frequently accessed data locally
- Use offline persistence for better UX

### Storage Optimization
- Compress images before upload
- Use appropriate image formats (WebP)
- Implement progressive loading
- Cache downloaded images locally

### Example Optimization
```kotlin
class OptimizedImageLoader @Inject constructor(
    private val storage: StorageRepository,
    private val cache: ImageCache
) {
    
    suspend fun loadImage(imageId: String): Result<Bitmap> {
        // Check cache first
        cache.get(imageId)?.let { 
            return Result.Success(it) 
        }
        
        // Load from Firebase
        return when (val result = storage.getImageUrl(imageId)) {
            is Result.Success -> {
                val bitmap = loadBitmapFromUrl(result.data)
                cache.put(imageId, bitmap)
                Result.Success(bitmap)
            }
            is Result.Error -> result
        }
    }
}
```

## Monitoring & Analytics

### Firebase Analytics
```kotlin
class AnalyticsTracker @Inject constructor(
    private val analytics: FirebaseAnalytics
) {
    
    fun trackPhotoUpload(success: Boolean) {
        analytics.logEvent("photo_upload") {
            param("success", success)
            param("timestamp", System.currentTimeMillis())
        }
    }
    
    fun trackPhotoRating(ratingGiven: Boolean) {
        analytics.logEvent("photo_rating") {
            param("rating_given", ratingGiven)
        }
    }
}
```

---

*For backend API integration details, see the [server documentation](../../../server/README.md).*