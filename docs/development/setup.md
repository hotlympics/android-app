# Development Setup Guide

Complete setup instructions for developing the Hotlympics Android application.

## Prerequisites

### Required Software
- **Android Studio**: Hedgehog | 2023.1.1 or newer
- **JDK**: 17 or newer (bundled with Android Studio)
- **Android SDK**: API 26+ (Android 8.0) to API 36 (Android 14)
- **Git**: For version control
- **Node.js**: For backend development (if needed)

### Hardware Requirements
- **RAM**: 8GB minimum, 16GB recommended
- **Storage**: 10GB free space for Android SDK and project
- **USB Debugging Device**: Android device for testing (optional but recommended)

## Project Setup

### 1. Clone Repository
```bash
git clone <repository-url>
cd hotlympics/android-app
```

### 2. Android Studio Configuration
1. Open Android Studio
2. Import the `android-app` directory as an existing project
3. Wait for Gradle sync to complete
4. Ensure proper SDK configuration in File > Project Structure

### 3. Firebase Configuration
1. Download `google-services.json` from Firebase Console
2. Place file in `app/` directory
3. Verify Firebase project matches backend configuration
4. Enable required services:
   - Authentication
   - Firestore Database
   - Cloud Storage

### 4. Environment Configuration

#### Create `local.properties`
```properties
# Android SDK path (usually auto-generated)
sdk.dir=/path/to/Android/Sdk

# API Configuration
API_BASE_URL=https://your-api-domain.com
FIREBASE_PROJECT_ID=your-firebase-project-id

# Debug configurations
DEBUG_LOGGING=true
ENABLE_STRICT_MODE=false
```

#### Build Configuration
Verify `app/build.gradle.kts` contains:
```kotlin
android {
    compileSdk = 36
    
    defaultConfig {
        applicationId = "com.example.hotlympics"
        minSdk = 26
        targetSdk = 36
        versionCode = 1
        versionName = "1.0"
    }
    
    buildTypes {
        debug {
            isDebuggable = true
            buildConfigField("String", "API_BASE_URL", "\"${project.findProperty("API_BASE_URL")}\"")
        }
    }
}
```

## Development Workflow

### 1. Build Commands
```bash
# Build debug APK
./gradlew assembleDebug

# Build release APK
./gradlew assembleRelease

# Install debug APK to connected device
./gradlew installDebug

# Clean build
./gradlew clean
```

### 2. Testing Commands
```bash
# Run unit tests
./gradlew test

# Run instrumented tests (requires connected device)
./gradlew connectedAndroidTest

# Run specific test class
./gradlew test --tests="com.example.hotlympics.PhotoVerificationTest"
```

### 3. Code Quality
```bash
# Run lint checks
./gradlew lint

# Format code with ktlint
./gradlew ktlintFormat

# Check code formatting
./gradlew ktlintCheck
```

## IDE Configuration

### Android Studio Plugins
Install recommended plugins:
- **Kotlin**: Language support (usually pre-installed)
- **Firebase**: Firebase integration tools
- **ML Kit**: Machine learning development tools
- **Database Inspector**: SQLite/Room debugging

### Code Style Settings
Import the project's code style configuration:
1. File > Settings > Editor > Code Style
2. Import from `config/android-studio-codestyle.xml` (if available)
3. Apply Kotlin official style guide

### Live Templates
Set up useful code templates:
- Compose function templates
- ViewModel boilerplate
- Repository implementations
- Test function templates

## Debugging Setup

### Device Configuration
1. Enable Developer Options on your Android device
2. Enable USB Debugging
3. Connect device and accept debugging permission
4. Verify device appears in Android Studio device list

### Debugging Tools
- **Layout Inspector**: Debug Compose UI layouts
- **Database Inspector**: Inspect Room database contents
- **Network Inspector**: Monitor API calls
- **Memory Profiler**: Track memory usage

### Logging Configuration
```kotlin
// Enable comprehensive logging in debug builds
if (BuildConfig.DEBUG) {
    Timber.plant(Timber.DebugTree())
    StrictMode.enableDefaults()
}
```

## Testing Setup

### Unit Testing Framework
```kotlin
// In app/build.gradle.kts
dependencies {
    testImplementation("junit:junit:4.13.2")
    testImplementation("io.mockk:mockk:1.13.8")
    testImplementation("org.jetbrains.kotlinx:kotlinx-coroutines-test:1.8.1")
    testImplementation("androidx.arch.core:core-testing:2.2.0")
}
```

### UI Testing Framework
```kotlin
// Compose testing dependencies
androidTestImplementation("androidx.compose.ui:ui-test-junit4:$compose_version")
androidTestImplementation("androidx.test.espresso:espresso-core:3.5.1")
androidTestImplementation("androidx.test.ext:junit:1.1.5")
debugImplementation("androidx.compose.ui:ui-test-manifest:$compose_version")
```

### Test Configuration
Create test configuration files:
- `src/test/resources/robolectric.properties`
- Test application class for mock dependencies
- Shared test utilities and factories

## Common Issues & Solutions

### Gradle Sync Issues
```bash
# Clear Gradle cache
./gradlew clean
rm -rf ~/.gradle/caches/

# Refresh dependencies
./gradlew build --refresh-dependencies
```

### Firebase Configuration
- Ensure `google-services.json` is in `app/` directory
- Verify package name matches applicationId
- Check Firebase project has required services enabled

### ML Kit Issues
- Verify Google Play Services is up to date on test device
- Check internet connection for model downloads
- Clear app data if models fail to load

### Camera Permissions
Add to `AndroidManifest.xml`:
```xml
<uses-permission android:name="android.permission.CAMERA" />
<uses-feature android:name="android.hardware.camera" android:required="true" />
```

## Performance Optimization

### Build Performance
```kotlin
// In gradle.properties
org.gradle.jvmargs=-Xmx4g -XX:MaxMetaspaceSize=1g
org.gradle.parallel=true
org.gradle.caching=true
android.useAndroidX=true
android.enableJetifier=true
```

### Runtime Performance
- Enable R8 code shrinking in release builds
- Use ProGuard rules for optimization
- Monitor memory usage with profiler
- Optimize image loading and caching

## Continuous Integration

### GitHub Actions Setup
Create `.github/workflows/android.yml`:
```yaml
name: Android CI

on:
  push:
    branches: [ main, develop ]
  pull_request:
    branches: [ main ]

jobs:
  test:
    runs-on: ubuntu-latest
    
    steps:
    - uses: actions/checkout@v3
    
    - name: Set up JDK 17
      uses: actions/setup-java@v3
      with:
        java-version: '17'
        distribution: 'temurin'
        
    - name: Cache Gradle packages
      uses: actions/cache@v3
      with:
        path: |
          ~/.gradle/caches
          ~/.gradle/wrapper
        key: gradle-${{ runner.os }}-${{ hashFiles('**/*.gradle*', '**/gradle-wrapper.properties') }}
        
    - name: Run tests
      run: ./gradlew test
      
    - name: Run lint
      run: ./gradlew lint
```

## Development Tips

### Hot Reload for Compose
- Use Compose preview for rapid UI development
- Enable Live Edit in Android Studio settings
- Use `@Preview` annotations extensively

### Database Inspection
- Use Database Inspector to view Room database contents
- Set up database exports for debugging
- Create database migration tests

### API Development
- Use Android Studio's Network Inspector
- Set up mock API responses for testing
- Implement proper error handling and retry logic

### Version Control
```bash
# Recommended .gitignore additions
local.properties
*.keystore
*.jks
/captures
.externalNativeBuild
.cxx
```

## Resources

### Documentation
- [Android Developer Guides](https://developer.android.com/guide)
- [Jetpack Compose Documentation](https://developer.android.com/jetpack/compose)
- [ML Kit Documentation](https://developers.google.com/ml-kit)
- [Firebase Android Setup](https://firebase.google.com/docs/android/setup)

### Tools
- [Scrcpy](https://github.com/Genymobile/scrcpy) - Screen mirroring for device testing
- [Flipper](https://fbflipper.com/) - Advanced debugging platform
- [LeakCanary](https://square.github.io/leakcanary/) - Memory leak detection

---

*For advanced development patterns, see [architecture/overview.md](../architecture/overview.md).*