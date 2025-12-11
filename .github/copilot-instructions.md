# Copilot Instructions for ID-Scan Flutter Project

## Project Overview

A Flutter mobile application for ID document (KTP - Indonesian National ID) scanning and liveness detection via face recognition. The app uses ML Kit for computer vision tasks and custom image processing for document extraction.

## Architecture & Key Components

### Feature Modules (lib/)

- **ktp_ocr.dart**: KTP document scanning with image preprocessing and OCR extraction
- **liveness_detection.dart**: Face detection with randomized challenge-response verification (turn head, smile, blink, etc.)
- **new_page_screen.dart**: Simple completion screen
- **main.dart**: Entry point that initializes camera and routes to KTPAuthScreen

### Core Dependencies & Integration Points

- **camera (^0.11.0+2)**: Camera initialization must happen in `main()` with `WidgetsFlutterBinding.ensureInitialized()` before app launch
- **google_mlkit_face_detection (^0.12.0)**: Real-time face detection with landmarks and contour tracking
- **ktp_extractor (^0.0.2)**: Custom native plugin for KTP document analysis
- **image (^4.5.2)**: Image manipulation (resize, grayscale, sketch filters, brightness/contrast adjustment)
- **permission_handler (^11.3.1)**: Runtime permissions for camera and storage access

## Developer Workflows

### Building

```bash
flutter pub get              # Fetch dependencies
flutter build apk           # Android release build
flutter build ios           # iOS release build
flutter build web           # Web build
```

### Testing & Analysis

```bash
flutter analyze             # Lint check (uses flutter_lints)
flutter test                # Run widget tests (test/)
```

### Development

```bash
flutter run                 # Debug on connected device
flutter run -d chrome       # Run on web
```

## Code Patterns & Conventions

### Image Processing Pipeline (ktp_ocr.dart)

1. Image capture → stored in `_originalImage` (for UI display)
2. Preprocessing in `_applyFilters()` runs in isolate via `compute()` to prevent UI lag
3. Filtered image stored in `_filteredImage` → passed to `KtpExtractor.extractKtp()`
4. Results populated via `_initializeControllers()` creating TextEditingControllers for each field

**Key Pattern**: Always use `compute()` for heavy image operations to keep UI responsive.

### Face Detection Challenge System (liveness_detection.dart)

- Challenges array shuffled, then 4 random ones selected in `_generateNewChallenges()`
- Challenge text mapped via `_getCurrentChallenge()` switch statement
- Real-time face detection triggered by `_startFaceDetection()` when camera initializes
- Images captured via `_captureAndSave()` saved to gallery with timestamp

**Key Pattern**: Challenge state tracked via `currentChallengeIndex`, verified against challenge-specific face landmarks (head pose, eye closure, smile).

### State Management

- Uses StatefulWidget with simple `setState()` (no external state management library)
- Loading states via `_isLoading` boolean flags
- Form editing via `Map<String, TextEditingController> _controllers`
- Temporal state: `DateTime? _selectedDate` for date selection

### UI Patterns

- ListView for scrollable content in KTPAuthScreen
- ElevatedButton for primary actions ("Take a Picture")
- Loading indicator: `CircularProgressIndicator()`
- Conditional rendering: `if (_ktpModel != null) _buildEditableFields()`

## Permission & Native Integration

### Required Permissions

- **Camera**: Must be declared in AndroidManifest.xml and iOS Info.plist
- **Storage**: Requested at runtime in `requestStoragePermission()` for image gallery saving
- **Photos**: iOS-specific permission for image library access

### Native Dependencies

- `ktp_extractor` plugin handles native KTP field extraction (must be available in pubspec.lock)
- Ensure platform-specific build files are configured (android/build.gradle, ios/Podfile)

## Important Technical Details

### Isolate Usage

`compute(_applyFilters, file)` runs image filtering in background isolate. The `_applyFilters()` must be a **top-level or static function** to work with compute().

### Camera Controller

- Initialized with **front camera** for face detection (via `CameraLensDirection.front`)
- Uses **NV21** image format for ML Kit compatibility
- **High resolution preset** for accurate face landmark detection
- Must call `_cameraController.dispose()` in cleanup (check in full file)

### Image Format Conversions

- **readAsBytes()** → raw bytes for ML Kit
- **img.decodeImage()** → image package manipulation
- **Image.file()** → Flutter UI widget display

## Memory Management for Lower-End Devices

### Image Processing Optimization (ktp_ocr.dart)

Current implementation has **critical memory concerns**:

- `img.decodeImage()` loads full image into memory (uncompressed)
- `_applyFilters()` resizes to 800px width, but intermediate copies remain in memory
- JPEG encoding at quality 100 creates large temp files (`_filtered.jpg`)

**Recommended optimizations**:

1. **Reduce resize dimension** on low-memory devices: Check device RAM via `DeviceInfoPlugin` and dynamically set resize width (400px for <2GB RAM, 800px for ≥2GB)
2. **Compress JPEG quality**: Change `img.encodeJpg(image, quality: 100)` to `quality: 75-85` to reduce file size 30-40%
3. **Clean up intermediate images**: After `KtpExtractor.extractKtp()` succeeds, delete temp `_filtered.jpg` file immediately
4. **Use memory-efficient filters**: Avoid chained filters; combine processing in single pass where possible

### Face Detection Streaming (liveness_detection.dart)

Current implementation **streams camera frames continuously**:

- `CameraImage` planes concatenated every ~1 second in `_concatenatePlanes()`
- `WriteBuffer` allocates memory for Uint8List conversion
- Face detection runs on frames even when not processing challenges

**Recommended optimizations**:

1. **Frame throttling**: Add frame skip logic—process every 2nd or 3rd frame on low-memory devices:
   ```dart
   int frameCount = 0;
   _cameraController.startImageStream((image) async {
     if (frameCount++ % 2 != 0) return;  // Skip every other frame
     // ... detection logic
   });
   ```
2. **Reuse Uint8List**: Create buffer pool to avoid repeated allocations in `_concatenatePlanes()`
3. **Lazy initialization**: Only call `_startFaceDetection()` when challenge is active, not on screen init

### General Memory Best Practices

- **Explicit disposal**: Always call `.dispose()` on controllers, detectors, and image files (e.g., `_filteredImage.delete()` after OCR)
- **Stream cleanup**: Cancel image streams in `dispose()` to prevent background processing
- **File cleanup**: Delete temporary files (`_filtered.jpg`) immediately after extraction
- **Avoid image caching**: Set `Image.file(cacheWidth: 400, cacheHeight: 300)` if displaying preview to limit decoded size

## Common Tasks & Implementation Notes

### Adding New ML Kit Models

- Update pubspec.yaml dependency
- Import detector class (e.g., `FaceDetector`, `PoseDetector`)
- Initialize in `initState()` with options (e.g., `enableClassification: true`)
- Process frames in `_startFaceDetection()` callback

### Modifying KTP OCR Fields

- Edit form field structure in `_buildEditableFields()`
- Update `_controllers` initialization in `_initializeControllers()`
- Adjust image preprocessing filters in `_applyFilters()` if needed (sketch amount, brightness/contrast)

### Adding New Liveness Challenges

- Add challenge string to `challenges` list (e.g., `'KEDIP'`)
- Add UI text mapping in `_getCurrentChallenge()` switch statement
- Implement detection logic using `FaceDetector` landmarks (e.g., eye closure for blink, smile intensity, head rotation angles)

## Build System Notes

- **Android**: Uses Gradle wrapper (gradlew), build.gradle in android/app/
- **iOS**: Uses CocoaPods (check Runner.xcworkspace)
- **Project Dart SDK**: `>=3.4.3 <4.0.0` (verify compatibility with packages)
- **Linting**: Inherits from flutter_lints with flutter.yaml rules (add custom rules in analysis_options.yaml)

## File Organization & Naming

- All feature screens in `lib/` (flat structure, no nested folders currently)
- Dart naming: snake_case for file names and class variables, PascalCase for classes
- Platform-specific code: android/ and ios/ folders contain native configurations
- Assets: Uncommented sections in pubspec.yaml show how to add images/fonts (currently none defined)
