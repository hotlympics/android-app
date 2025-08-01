# Hotlympics Android App

An Android application for the Hotlympics platform - a photo rating system where users can upload photos, select favorites for an ELO-based competition pool, and participate in attractiveness comparisons.

## Overview

Hotlympics allows users to:
- Create accounts and authenticate via Firebase
- Upload personal photos (max 10 per user)
- Select up to 2 photos for the competitive ELO pool
- Rate random photo pairs from the global pool
- Track ELO ratings based on comparison results

The Android app provides the same core functionality as the web platform while leveraging mobile-specific features for enhanced security and user experience.

## Architecture

### Backend Integration
- **Firebase Authentication**: Shared user accounts with web platform
- **Firebase Firestore**: User data, image metadata, and ELO ratings
- **Google Cloud Storage**: Image file storage
- **REST API**: Node.js/Express server with shared endpoints

### Key API Endpoints
- `POST /auth/*` - Firebase authentication flow
- `POST /images/upload` - Upload photos (max 10 per user)
- `GET /images/user` - Fetch user's uploaded photos
- `GET /images/serve/:fileName` - Serve image files
- `DELETE /images/:imageId` - Delete user's photos
- `PUT /user/pool` - Update pool selections (max 2 photos)
- `PUT /user/profile` - Update user profile (gender, DOB, 18+ required)

## Mobile-Specific Features

### Photo Verification (Planned)
Enhanced security measures to ensure photo authenticity:

#### 1. Real Person Detection (On-Device)
- **ML Kit Face Detection**: Verify uploaded photos contain real human faces
- **CameraX Integration**: Capture photos directly within app for verification
- **Metadata Analysis**: Check EXIF data for camera source vs. downloaded images
- **Live Photo Verification**: Optional real-time face detection during capture

#### 2. User Authentication (Planned)
- **Selfie Verification**: Compare uploaded photos with live selfie using face matching
- **Biometric Authentication**: Use device biometrics for secure photo uploads
- **Device Camera Requirement**: Encourage/require camera capture vs. gallery selection
- **Liveness Detection**: Detect if selfie verification uses live person vs. photo

### Implementation Options:
```kotlin
// ML Kit Face Detection
implementation 'com.google.mlkit:face-detection:16.1.5'

// CameraX for photo capture
implementation 'androidx.camera:camera-camera2:1.3.1'
implementation 'androidx.camera:camera-lifecycle:1.3.1'
implementation 'androidx.camera:camera-view:1.3.1'

// Biometric authentication
implementation 'androidx.biometric:biometric:1.1.0'
```

## Project Structure

```
android-app/
├── app/
│   ├── src/
│   │   ├── main/
│   │   │   ├── java/com/example/hotlympics/
│   │   │   │   ├── ui/theme/          # Compose UI theming
│   │   │   │   ├── data/              # Repository pattern, API clients
│   │   │   │   ├── domain/            # Business logic, use cases
│   │   │   │   ├── presentation/      # ViewModels, UI composables
│   │   │   │   ├── utils/             # Utilities, extensions
│   │   │   │   └── MainActivity.kt
│   │   │   ├── res/                   # Resources (layouts, strings, etc.)
│   │   │   └── AndroidManifest.xml
│   │   ├── androidTest/               # Integration tests
│   │   └── test/                      # Unit tests
│   ├── build.gradle.kts
│   └── proguard-rules.pro
├── gradle/
├── build.gradle.kts
├── settings.gradle.kts
└── README.md
```

## Technology Stack

### Core
- **Language**: Kotlin
- **UI Framework**: Jetpack Compose
- **Architecture**: MVVM with Repository pattern
- **Dependency Injection**: Dagger Hilt (planned)

### Firebase Integration
- **Authentication**: Firebase Auth (shared with web)
- **Database**: Firestore (shared collections)
- **Analytics**: Firebase Analytics (planned)
- **Crashlytics**: Firebase Crashlytics (planned)

### Networking & Storage
- **HTTP Client**: Retrofit + OkHttp
- **Image Loading**: Coil for Compose
- **Local Storage**: Room database (for caching)
- **Preferences**: DataStore

### Camera & ML
- **Camera**: CameraX
- **Face Detection**: ML Kit Face Detection
- **Image Processing**: Android Camera2 API
- **Biometrics**: AndroidX Biometric

## Development Setup

### Prerequisites
- Android Studio Hedgehog | 2023.1.1 or newer
- Android SDK API 26+ (Android 8.0)
- Target SDK 36 (Android 14)
- Kotlin 1.9+

### Firebase Configuration
1. Add `google-services.json` to `app/` directory
2. Ensure Firebase project matches web platform configuration
3. Enable Authentication, Firestore, and Storage in Firebase Console

### Local Development
```bash
# Clone repository
git clone <repository-url>
cd hotlympics/android-app

# Build and run
./gradlew assembleDebug
./gradlew installDebug

# Run tests
./gradlew test
./gradlew connectedAndroidTest
```

### Environment Configuration
Create `local.properties` with:
```properties
# Android SDK path
sdk.dir=/path/to/Android/Sdk

# API configuration (if needed)
API_BASE_URL=https://your-api-domain.com
```

## Current Status

### ✅ Completed
- [x] Basic Android project structure
- [x] Jetpack Compose setup
- [x] Firebase configuration ready
- [x] Backend API analysis

### 🚧 In Progress
- [ ] Firebase Authentication integration
- [ ] Photo upload functionality
- [ ] Camera integration
- [ ] Photo verification system
- [ ] Rating/comparison interface

### 📋 Planned Features
- [ ] Face detection for upload verification
- [ ] Selfie-based user authentication
- [ ] ELO rating display and history
- [ ] Push notifications for new comparisons
- [ ] Offline support for cached photos
- [ ] Dark mode support
- [ ] Accessibility improvements

## Contributing

### Code Style
- Follow [Kotlin coding conventions](https://kotlinlang.org/docs/coding-conventions.html)
- Use [ktlint](https://ktlint.github.io/) for formatting
- Follow [Compose best practices](https://developer.android.com/jetpack/compose/guidelines)

### Git Workflow
1. Create feature branch from `main`
2. Implement changes with tests
3. Ensure all checks pass
4. Submit pull request with detailed description

### Testing Strategy
- **Unit Tests**: Domain logic, ViewModels, utilities
- **Integration Tests**: Repository implementations, API clients
- **UI Tests**: Compose screens, user flows
- **Manual Testing**: Camera functionality, ML Kit features

## Security Considerations

### Photo Verification
- Implement multiple layers of verification (face detection + metadata + user behavior)
- Consider server-side verification as backup for on-device checks
- Store verification status in Firestore for audit trails

### User Data Protection
- Minimize local data storage
- Use Android Keystore for sensitive data
- Implement proper session management
- Follow GDPR compliance for EU users

### API Security
- Implement certificate pinning
- Use Firebase Auth tokens for API authentication
- Validate all uploads server-side regardless of client verification

## Performance Optimization

### Image Handling
- Implement progressive image loading
- Use appropriate image compression
- Cache images efficiently with Coil
- Lazy loading for photo grids

### ML Kit Optimization
- Use on-device models when possible
- Implement proper model lifecycle management
- Cache face detection results appropriately
- Optimize camera preview performance

## Future Enhancements

### Advanced Features
- **AI-Powered Verification**: Server-side deepfake detection
- **Social Features**: Friend connections, private pools
- **Gamification**: Achievements, streaks, leaderboards
- **Analytics**: Detailed rating statistics and trends
- **Monetization**: Premium features, ad-free experience

### Technical Improvements
- **Modularization**: Feature-based modules
- **Performance**: Advanced caching strategies
- **Accessibility**: Full screen reader support
- **Internationalization**: Multi-language support

## Links

- **Backend Repository**: `../server/`
- **Web Application**: `../website/`
- **API Documentation**: See `../server/README.md`
- **Firebase Console**: [Your Firebase Project](https://console.firebase.google.com)

## License

[Your License Here]

---

**Note**: This Android app is designed to work seamlessly with the existing Hotlympics web platform and backend infrastructure. All user accounts, photos, and ratings are shared between platforms.