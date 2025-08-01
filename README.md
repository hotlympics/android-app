# Hotlympics Android App

Android application for the Hotlympics photo rating platform.

## Tech Stack
- **Language**: Kotlin
- **UI**: Jetpack Compose
- **Architecture**: MVVM
- **Backend**: Firebase Auth, Firestore, Cloud Storage
- **Camera**: CameraX + ML Kit Face Detection
- **Target SDK**: API 36 (Android 14)
- **Min SDK**: API 26 (Android 8.0)

## Setup
```bash
# Build
./gradlew assembleDebug

# Tests
./gradlew test
./gradlew connectedAndroidTest

# Install
./gradlew installDebug
```

## Key Features
- Firebase authentication (shared with web)
- Photo upload with face detection
- ELO-based photo rating system
- Real-time photo verification
- Biometric security

## Documentation
See [docs/](docs/) for detailed architecture and development guides.