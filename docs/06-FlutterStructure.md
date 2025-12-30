# YellNo Flutter Project Structure

## Directory Layout

```
yellno/
├── android/
├── ios/                          # Future phase
├── lib/
│   ├── main.dart                 # App entry point, initialization
│   ├── app.dart                  # MaterialApp configuration, routing
│   │
│   ├── config/
│   │   ├── constants.dart        # App-wide constants
│   │   ├── environment.dart      # Environment config (dev/staging/prod)
│   │   └── theme.dart            # App theme, colors, typography
│   │
│   ├── models/
│   │   ├── family.dart           # Family entity
│   │   ├── member.dart           # Member entity
│   │   ├── incident.dart         # Incident entity
│   │   ├── device.dart           # Device entity
│   │   ├── pending_confirmation.dart
│   │   ├── stats_summary.dart
│   │   ├── reward.dart
│   │   └── analysis_result.dart  # Response from /analyze endpoint
│   │
│   ├── services/
│   │   ├── api/
│   │   │   ├── api_client.dart           # HTTP client, auth headers, error handling
│   │   │   ├── family_api.dart           # /family endpoints
│   │   │   ├── member_api.dart           # /family/members endpoints
│   │   │   ├── voiceprint_api.dart       # /voiceprint endpoints
│   │   │   ├── analyze_api.dart          # /analyze endpoint
│   │   │   ├── incident_api.dart         # /incidents endpoints
│   │   │   ├── stats_api.dart            # /stats endpoint
│   │   │   └── device_api.dart           # /devices endpoints
│   │   │
│   │   ├── audio/
│   │   │   ├── audio_capture_service.dart    # Microphone access, continuous capture
│   │   │   ├── audio_buffer.dart             # Rolling buffer management
│   │   │   ├── volume_analyzer.dart          # Real-time volume calculation
│   │   │   └── audio_encoder.dart            # WAV encoding, base64 conversion
│   │   │
│   │   ├── auth/
│   │   │   ├── auth_service.dart         # Google Sign-In orchestration
│   │   │   └── token_manager.dart        # Token storage, refresh
│   │   │
│   │   ├── alert_service.dart            # Trigger sound/vibration/visual alerts
│   │   ├── notification_service.dart     # Local notifications
│   │   ├── connectivity_service.dart     # Network status monitoring
│   │   └── device_service.dart           # Device ID generation, heartbeat
│   │
│   ├── repositories/
│   │   ├── family_repository.dart        # Family data, caching
│   │   ├── member_repository.dart        # Member data, caching
│   │   ├── incident_repository.dart      # Incidents, local cache
│   │   ├── stats_repository.dart         # Stats data
│   │   └── confirmation_repository.dart  # Pending confirmations queue
│   │
│   ├── providers/
│   │   ├── auth_provider.dart            # Auth state
│   │   ├── family_provider.dart          # Family/member state
│   │   ├── listening_provider.dart       # Audio capture state, analysis triggers
│   │   ├── incident_provider.dart        # Incidents state
│   │   ├── stats_provider.dart           # Stats/leaderboard state
│   │   ├── confirmation_provider.dart    # Pending confirmations state
│   │   ├── connectivity_provider.dart    # Online/offline state
│   │   └── onboarding_provider.dart      # Onboarding flow state
│   │
│   ├── screens/
│   │   ├── splash/
│   │   │   └── splash_screen.dart
│   │   │
│   │   ├── auth/
│   │   │   └── sign_in_screen.dart
│   │   │
│   │   ├── onboarding/
│   │   │   ├── privacy_disclosure_screen.dart
│   │   │   ├── family_name_screen.dart
│   │   │   ├── setup_mode_screen.dart
│   │   │   ├── passive_mode_info_screen.dart
│   │   │   ├── enrollment_screen.dart
│   │   │   ├── voice_capture_screen.dart
│   │   │   ├── alert_config_screen.dart
│   │   │   ├── gamification_info_screen.dart
│   │   │   ├── gamification_setup_screen.dart
│   │   │   ├── calibration_info_screen.dart
│   │   │   └── permission_screen.dart
│   │   │
│   │   ├── dashboard/
│   │   │   ├── dashboard_screen.dart
│   │   │   ├── widgets/
│   │   │   │   ├── listening_status_card.dart
│   │   │   │   ├── leaderboard_card.dart
│   │   │   │   ├── recent_incidents_card.dart
│   │   │   │   └── current_reward_card.dart
│   │   │
│   │   ├── incidents/
│   │   │   ├── incidents_list_screen.dart
│   │   │   ├── incident_detail_screen.dart
│   │   │   └── widgets/
│   │   │       ├── incident_tile.dart
│   │   │       └── removal_confirmation_dialog.dart
│   │   │
│   │   ├── family/
│   │   │   ├── family_screen.dart
│   │   │   ├── member_detail_screen.dart
│   │   │   ├── add_member_screen.dart
│   │   │   └── widgets/
│   │   │       ├── member_tile.dart
│   │   │       └── voiceprint_status_badge.dart
│   │   │
│   │   ├── settings/
│   │   │   ├── settings_screen.dart
│   │   │   ├── alert_settings_screen.dart
│   │   │   ├── gamification_settings_screen.dart
│   │   │   ├── devices_screen.dart
│   │   │   └── account_screen.dart
│   │   │
│   │   └── confirmations/
│   │       ├── speaker_confirmation_sheet.dart
│   │       └── new_speaker_sheet.dart
│   │
│   ├── widgets/
│   │   ├── common/
│   │   │   ├── loading_overlay.dart
│   │   │   ├── error_view.dart
│   │   │   ├── offline_overlay.dart
│   │   │   └── primary_button.dart
│   │   │
│   │   ├── alerts/
│   │   │   └── yelling_alert_overlay.dart
│   │   │
│   │   └── audio/
│   │       ├── volume_indicator.dart
│   │       └── enrollment_progress.dart
│   │
│   └── utils/
│       ├── audio_utils.dart          # dB conversion, RMS calculation
│       ├── date_utils.dart           # Period calculations, formatting
│       ├── hash_utils.dart           # Dedupe hash generation
│       └── validators.dart           # Input validation
│
├── test/
│   ├── unit/
│   │   ├── services/
│   │   ├── repositories/
│   │   └── utils/
│   ├── widget/
│   │   └── screens/
│   └── integration/
│
├── pubspec.yaml
└── README.md
```

---

## Key Dependencies

```yaml
# pubspec.yaml

dependencies:
  flutter:
    sdk: flutter

  # State Management
  provider: ^6.0.0

  # Networking
  http: ^1.1.0
  
  # Authentication
  google_sign_in: ^6.1.0
  flutter_secure_storage: ^9.0.0

  # Audio
  record: ^5.0.0                    # Audio recording
  
  # Local Storage
  shared_preferences: ^2.2.0       # Simple key-value
  
  # Notifications
  flutter_local_notifications: ^16.0.0
  
  # Connectivity
  connectivity_plus: ^5.0.0
  
  # Utilities
  uuid: ^4.0.0
  crypto: ^3.0.0                   # SHA256 for dedupe hash
  intl: ^0.18.0                    # Date formatting

dev_dependencies:
  flutter_test:
    sdk: flutter
  mockito: ^5.4.0
  build_runner: ^2.4.0
```

---

## Module Responsibilities

### services/audio/audio_capture_service.dart

```
Responsibilities:
- Initialize microphone with correct settings (16kHz, mono, 16-bit)
- Start/stop foreground service (Android)
- Continuous audio stream to buffer
- Expose stream for volume analyzer

Dependencies:
- audio_buffer.dart
- record package

Exposes:
- startCapture()
- stopCapture()
- Stream<Uint8List> audioStream
- bool isCapturing
```

### services/audio/volume_analyzer.dart

```
Responsibilities:
- Subscribe to audio stream
- Calculate RMS and convert to dB for each chunk
- Maintain rolling average of ambient volume
- Detect threshold exceedance
- Trigger analysis when threshold exceeded

Dependencies:
- audio_capture_service.dart
- audio_utils.dart
- family_repository.dart (for volumeThresholdMultiplier)
- member_repository.dart (for speaker baselines)

Exposes:
- Stream<VolumeReading> volumeStream
- Stream<AnalysisTrigger> triggerStream
- double currentVolumeDb
- double ambientBaselineDb
```

### services/audio/audio_encoder.dart

```
Responsibilities:
- Extract segment from rolling buffer
- Encode as WAV format
- Convert to base64 string

Dependencies:
- audio_buffer.dart

Exposes:
- Future<String> encodeSegment(int startMs, int endMs)
- int getBufferDurationMs()
```

### providers/listening_provider.dart

```
Responsibilities:
- Orchestrate audio capture lifecycle
- Listen to volume analyzer triggers
- Send audio to analyze API
- Process analysis results
- Trigger alerts
- Manage cooldown period
- Handle confirmations queue

Dependencies:
- audio_capture_service.dart
- volume_analyzer.dart
- audio_encoder.dart
- analyze_api.dart
- alert_service.dart
- confirmation_provider.dart
- connectivity_provider.dart

State:
- ListeningStatus: idle, listening, analyzing, paused, offline
- DateTime? lastAnalysisSentAt
- AnalysisResult? lastResult
```

### repositories/family_repository.dart

```
Responsibilities:
- Fetch family from API
- Cache family locally
- Provide family settings to other components
- Update family settings

Dependencies:
- family_api.dart
- shared_preferences (for cache)

Exposes:
- Future<Family?> getFamily()
- Future<Family> createFamily(...)
- Future<Family> updateFamily(...)
- Future<void> deleteFamily()
- Stream<Family?> familyStream
```

---

## State Management Pattern

Using Provider with ChangeNotifier:

```
App Startup:
1. main.dart initializes services
2. MultiProvider wraps MaterialApp
3. AuthProvider checks auth state
4. If authenticated, FamilyProvider loads family
5. If family exists and setup complete, ListeningProvider starts

Provider Hierarchy:
- AuthProvider (root)
  - ConnectivityProvider
  - FamilyProvider (depends on auth)
    - ListeningProvider (depends on family, connectivity)
    - IncidentProvider (depends on family)
    - StatsProvider (depends on family)
    - ConfirmationProvider (depends on family)
```

---

## Platform-Specific Configuration

### Android (android/)

```
android/app/src/main/AndroidManifest.xml:
- RECORD_AUDIO permission
- FOREGROUND_SERVICE permission
- INTERNET permission
- Foreground service declaration

android/app/src/main/kotlin/.../ForegroundService.kt:
- Native foreground service for background audio capture
- Persistent notification: "YellNo is listening"
```

### Android TV Considerations

```
- Same codebase, different UI layouts
- Use TV-specific widgets where needed
- D-pad navigation support
- Leanback library integration (if needed)
- Detect platform: 
  if (Platform.isAndroid) {
    // Check for TV via UiModeManager
  }
```

---

## Initialization Sequence

```
main.dart:

1. WidgetsFlutterBinding.ensureInitialized()
2. Initialize secure storage
3. Initialize shared preferences
4. Initialize connectivity service
5. Initialize notification service
6. runApp(YellNoApp())

app.dart (YellNoApp):

1. MultiProvider setup
2. MaterialApp with router
3. Initial route based on auth state:
   - Not authenticated → SignInScreen
   - Authenticated, no family → OnboardingFlow
   - Authenticated, family exists, setup incomplete → resume onboarding
   - Authenticated, family exists, setup complete → DashboardScreen
```

---

## Background Processing (Android)

```
Foreground Service Lifecycle:

App Foreground:
- Audio capture via Dart (record package)
- Full processing in Dart

App Background:
- Foreground service keeps process alive
- Persistent notification required
- Audio capture continues
- Volume analysis continues
- API calls continue

App Killed:
- Service stops
- No background processing
- Resumes on next app launch

Note: True always-on background processing would require
native implementation. Initial version requires app to be
running (foreground or background with service).
```

---

## Error Handling Strategy

```
API Errors:
- Caught in api_client.dart
- Transformed to typed exceptions
- Propagated to providers
- Displayed via error_view.dart or snackbar

Audio Errors:
- Microphone access denied → Show permission screen
- Audio capture failure → Retry 3x, then show error
- Encoding failure → Log and skip analysis

Network Errors:
- Detected by connectivity_provider.dart
- Triggers offline_overlay.dart
- Pauses listening_provider.dart
- Auto-resumes on reconnect
```

---

## Testing Strategy

```
Unit Tests (test/unit/):
- Services: Mock dependencies, test logic
- Repositories: Mock API, test caching
- Utils: Pure function tests

Widget Tests (test/widget/):
- Screens: Mock providers, test UI states
- Widgets: Test rendering, interactions

Integration Tests (test/integration/):
- Full flows: Onboarding, detection cycle
- Requires emulator with microphone mock

Key Test Files:
- volume_analyzer_test.dart (threshold logic)
- audio_encoder_test.dart (WAV encoding)
- listening_provider_test.dart (orchestration)
- removal_confirmation_dialog_test.dart (3-step flow)
```