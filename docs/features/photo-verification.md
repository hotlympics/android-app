# Photo Verification System

Comprehensive specification for the photo verification and authentication system.

## Overview

The photo verification system ensures that uploaded photos meet quality standards and authenticity requirements through multi-layered validation.

## Verification Pipeline

### 1. Real-Time Camera Verification
```
Camera Preview → ML Kit Analysis → Quality Check → User Feedback
```

**Components:**
- Live face detection during capture
- Photo quality assessment
- Real-time user guidance
- Automatic capture when criteria met

### 2. Authentication Verification
```
Photo Capture → Biometric Auth → Selfie Match → Upload Authorization
```

**Components:**
- Device biometric authentication
- Face matching with selfie
- Liveness detection
- Anti-spoofing measures

### 3. Server-Side Validation
```
Upload → Backend Analysis → Quality Score → Pool Eligibility
```

**Components:**
- Additional AI verification
- Content moderation
- Duplicate detection
- Final quality scoring

## Technical Implementation

### ML Kit Integration
```kotlin
class PhotoVerificationAnalyzer(
    private val faceDetector: FaceDetector,
    private val poseDetector: PoseDetector,
    private val onResult: (VerificationResult) -> Unit
) : ImageAnalysis.Analyzer {
    
    override fun analyze(imageProxy: ImageProxy) {
        val inputImage = InputImage.fromImageProxy(imageProxy, imageProxy.imageInfo.rotationDegrees)
        
        // Face detection
        faceDetector.process(inputImage)
            .addOnSuccessListener { faces ->
                // Pose detection
                poseDetector.process(inputImage)
                    .addOnSuccessListener { poses ->
                        val result = analyzeResults(faces, poses)
                        onResult(result)
                    }
            }
            .addOnCompleteListener {
                imageProxy.close()
            }
    }
    
    private fun analyzeResults(faces: List<Face>, poses: List<Pose>): VerificationResult {
        return VerificationResult(
            faceCount = faces.size,
            faceConfidence = faces.firstOrNull()?.getBoundingBox()?.let { calculateConfidence(it) } ?: 0f,
            poseQuality = poses.firstOrNull()?.let { analyzePose(it) } ?: PoseQuality.POOR,
            lightingQuality = analyzeLighting(faces.firstOrNull()),
            isValid = isPhotoValid(faces, poses)
        )
    }
}
```

### Verification Result Model
```kotlin
data class VerificationResult(
    val faceCount: Int,
    val faceConfidence: Float,
    val poseQuality: PoseQuality,
    val lightingQuality: LightingQuality,
    val blurLevel: BlurLevel,
    val isValid: Boolean,
    val failureReasons: List<FailureReason> = emptyList()
)

enum class PoseQuality { EXCELLENT, GOOD, FAIR, POOR }
enum class LightingQuality { EXCELLENT, GOOD, FAIR, POOR }
enum class BlurLevel { NONE, SLIGHT, MODERATE, SEVERE }

sealed class FailureReason(val message: String) {
    object NoFaceDetected : FailureReason("No face detected in photo")
    object MultipleFaces : FailureReason("Multiple faces detected")
    object PoorLighting : FailureReason("Lighting is too poor")
    object BlurryImage : FailureReason("Image is too blurry")
    object InvalidPose : FailureReason("Face pose is not suitable")
    object TooFarAway : FailureReason("Face is too far from camera")
    object TooClose : FailureReason("Face is too close to camera")
}
```

## Verification Criteria

### Face Detection Requirements
- **Single Face**: Exactly one face must be detected
- **Face Size**: Face must occupy 15-60% of image area
- **Face Orientation**: Face angle within ±30 degrees
- **Eye Visibility**: Both eyes must be clearly visible
- **Face Completeness**: Full face from forehead to chin

### Image Quality Standards
- **Resolution**: Minimum 480x640 pixels
- **Lighting**: Even lighting across face
- **Blur**: Sharp focus on facial features
- **Contrast**: Sufficient contrast to distinguish features
- **Saturation**: Natural color reproduction

### Technical Metrics
```kotlin
data class QualityMetrics(
    val faceArea: Float,        // 0.15 - 0.60 (15% - 60% of image)
    val sharpness: Float,       // 0.0 - 1.0 (higher is sharper)
    val brightness: Float,      // 0.3 - 0.8 (optimal range)
    val contrast: Float,        // 0.4 - 1.0 (sufficient contrast)
    val symmetry: Float,        // 0.0 - 1.0 (face symmetry)
    val eyeOpenness: Float      // 0.7 - 1.0 (eyes open probability)
)

fun isQualityAcceptable(metrics: QualityMetrics): Boolean {
    return metrics.faceArea in 0.15f..0.60f &&
           metrics.sharpness > 0.6f &&
           metrics.brightness in 0.3f..0.8f &&
           metrics.contrast > 0.4f &&
           metrics.symmetry > 0.5f &&
           metrics.eyeOpenness > 0.7f
}
```

## User Experience Flow

### 1. Camera Preview with Guidance
```kotlin
@Composable
fun CameraPreviewWithGuidance(
    analyzer: PhotoVerificationAnalyzer,
    onPhotoAccepted: (Uri) -> Unit
) {
    var verificationState by remember { mutableStateOf<VerificationResult?>(null) }
    
    Box(modifier = Modifier.fillMaxSize()) {
        CameraPreview(
            analyzer = analyzer.apply {
                setResultCallback { result ->
                    verificationState = result
                    if (result.isValid) {
                        // Auto-capture when criteria met
                        capturePhoto()
                    }
                }
            }
        )
        
        // Overlay guidance
        verificationState?.let { state ->
            VerificationOverlay(
                state = state,
                modifier = Modifier.align(Alignment.Center)
            )
        }
        
        // Instruction text
        InstructionCard(
            instructions = getInstructions(verificationState),
            modifier = Modifier.align(Alignment.BottomCenter)
        )
    }
}

@Composable
fun VerificationOverlay(state: VerificationResult) {
    Canvas(modifier = Modifier.fillMaxSize()) {
        // Draw face detection box
        if (state.faceCount > 0) {
            drawFaceGuidance(state.faceConfidence)
        }
        
        // Draw quality indicators
        drawQualityIndicators(state)
    }
}
```

### 2. Real-Time Feedback Messages
```kotlin
fun getInstructions(state: VerificationResult?): String {
    return when {
        state == null -> "Position your face in the frame"
        state.faceCount == 0 -> "Move closer - no face detected"
        state.faceCount > 1 -> "Only one person should be in frame"
        state.lightingQuality == LightingQuality.POOR -> "Find better lighting"
        state.blurLevel == BlurLevel.SEVERE -> "Hold still - image is blurry"
        state.poseQuality == PoseQuality.POOR -> "Look straight at the camera"
        state.isValid -> "Perfect! Hold still..."
        else -> "Adjust position and try again"
    }
}
```

### 3. Biometric Authentication Flow
```kotlin
class BiometricAuthenticator @Inject constructor(
    private val biometricManager: BiometricManager,
    private val faceMatchingService: FaceMatchingService
) {
    
    suspend fun authenticateForPhotoUpload(
        capturedPhoto: Bitmap,
        activity: FragmentActivity
    ): AuthenticationResult {
        
        // Step 1: Device biometric authentication
        val biometricResult = authenticateWithBiometric(activity)
        if (!biometricResult.isSuccess) {
            return AuthenticationResult.BiometricFailed(biometricResult.error)
        }
        
        // Step 2: Capture selfie for face matching
        val selfieResult = captureSelfie(activity)
        if (!selfieResult.isSuccess) {
            return AuthenticationResult.SelfieFailed(selfieResult.error)
        }
        
        // Step 3: Match faces
        val matchResult = faceMatchingService.matchFaces(
            originalPhoto = capturedPhoto,
            selfiePhoto = selfieResult.bitmap
        )
        
        return when {
            matchResult.confidence > 0.8f -> AuthenticationResult.Success
            matchResult.confidence > 0.6f -> AuthenticationResult.LowConfidence(matchResult.confidence)
            else -> AuthenticationResult.NoMatch
        }
    }
}
```

## Security Measures

### Anti-Spoofing Protection
- **Liveness Detection**: Random movements during selfie
- **Depth Analysis**: Using device depth sensors if available
- **Texture Analysis**: Detecting printed photos vs. real faces
- **Temporal Analysis**: Video-based verification for movement

### Privacy Protection
- **On-Device Processing**: Face detection happens locally
- **Encrypted Storage**: Temporary face embeddings encrypted
- **No Face Storage**: Only verification results stored
- **Automatic Cleanup**: Face data deleted after verification

### Implementation
```kotlin
class LivenessDetector {
    
    suspend fun detectLiveness(
        videoFrames: List<Bitmap>
    ): LivenessResult {
        val movements = analyzeMovements(videoFrames)
        val depthVariation = analyzeDepth(videoFrames)
        val textureConsistency = analyzeTexture(videoFrames)
        
        val livenessScore = calculateLivenessScore(
            movements, depthVariation, textureConsistency
        )
        
        return LivenessResult(
            isLive = livenessScore > 0.7f,
            confidence = livenessScore,
            evidence = LivenessEvidence(movements, depthVariation, textureConsistency)
        )
    }
}
```

## Verification States

### UI State Management
```kotlin
sealed class VerificationState {
    object Initializing : VerificationState()
    object Analyzing : VerificationState()
    data class Guidance(val instructions: String) : VerificationState()
    data class ReadyToCapture(val countdown: Int) : VerificationState()
    object Captured : VerificationState()
    object BiometricAuth : VerificationState()
    object SelfieCapture : VerificationState()
    object Processing : VerificationState()
    data class Success(val photoUri: Uri) : VerificationState()
    data class Failed(val reason: String) : VerificationState()
}

class PhotoVerificationViewModel : ViewModel() {
    
    private val _state = MutableStateFlow<VerificationState>(VerificationState.Initializing)
    val state: StateFlow<VerificationState> = _state.asStateFlow()
    
    fun startVerification() {
        _state.value = VerificationState.Analyzing
        // Begin camera analysis
    }
    
    fun onVerificationResult(result: VerificationResult) {
        _state.value = when {
            result.isValid -> VerificationState.ReadyToCapture(3)
            else -> VerificationState.Guidance(result.failureReasons.firstOrNull()?.message ?: "Please adjust")
        }
    }
}
```

## Performance Optimization

### ML Kit Optimization
- **Model Caching**: Cache face detection models
- **Frame Skipping**: Analyze every 3rd frame for better performance
- **Resolution Scaling**: Use appropriate resolution for detection
- **Threading**: Run detection on background threads

### Memory Management
```kotlin
class OptimizedVerificationAnalyzer : ImageAnalysis.Analyzer {
    
    private var lastAnalysisTime = 0L
    private val analysisInterval = 100L // Analyze every 100ms
    
    override fun analyze(imageProxy: ImageProxy) {
        val currentTime = System.currentTimeMillis()
        
        if (currentTime - lastAnalysisTime > analysisInterval) {
            performAnalysis(imageProxy)
            lastAnalysisTime = currentTime
        }
        
        imageProxy.close()
    }
    
    private fun performAnalysis(imageProxy: ImageProxy) {
        // Efficient analysis implementation
    }
}
```

## Testing Strategy

### Unit Tests
- Verification logic testing
- Quality metrics calculation
- State management testing

### Integration Tests  
- ML Kit integration testing
- Camera functionality testing
- Biometric authentication flow

### UI Tests
- Verification flow testing
- Error state handling
- User guidance testing

---

*See [api/verification-api.md](../api/verification-api.md) for backend integration details.*