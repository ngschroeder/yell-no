# YellNo Onboarding Screens Outline

## Screen Flow

```
1. SplashScreen
2. SignInScreen
3. PrivacyDisclosureScreen
4. FamilyNameScreen
5. SetupModeScreen
   ├─ (if passive) PassiveModeInfoScreen
   └─ (if proactive) EnrollmentScreen → VoiceCaptureScreen (loop)
6. AlertConfigScreen
7. GamificationInfoScreen
   └─ (if enabled) GamificationSetupScreen
8. CalibrationInfoScreen
9. PermissionScreen
10. → DashboardScreen
```

---

## 1. SplashScreen

**Route:** `/splash`

**Purpose:** App initialization, auth state check

**Elements:**
- App logo (centered)
- App name "YellNo"
- Loading indicator

**Logic:**
```
on mount:
  initialize services
  check auth state
  if authenticated:
    fetch family
    if family exists and setupCompleted:
      navigate → DashboardScreen
    else if family exists and not setupCompleted:
      navigate → resume appropriate onboarding step
    else:
      navigate → PrivacyDisclosureScreen
  else:
    navigate → SignInScreen
```

**Duration:** Auto-advance when ready, minimum 1 second

---

## 2. SignInScreen

**Route:** `/sign-in`

**Purpose:** Google authentication

**Elements:**
- App logo
- App tagline (placeholder text)
- Google Sign-In button

**Actions:**
| Element | Action |
|---------|--------|
| Google Sign-In button | Trigger Google OAuth flow |

**Logic:**
```
on sign in success:
  store auth token
  fetch family
  if family exists:
    navigate → DashboardScreen (or resume onboarding)
  else:
    navigate → PrivacyDisclosureScreen
```

**Error States:**
- Sign-in cancelled: remain on screen
- Sign-in failed: show error message, retry option

---

## 3. PrivacyDisclosureScreen

**Route:** `/onboarding/privacy`

**Purpose:** Inform user about privacy practices, obtain consent

**Elements:**
- Header: "Your Privacy Matters"
- Body content (list format):
  - No audio recordings are ever saved
  - Only statistics are stored (who yelled, when, how often)
  - Temporary transcripts (passive mode only) deleted within 7 days
  - You can delete all your data at any time
- Consent button

**Actions:**
| Element | Action |
|---------|--------|
| "I Understand" button | navigate → FamilyNameScreen |

**Requirements:**
- User must scroll to bottom before button enables (optional UX choice)
- No skip option

---

## 4. FamilyNameScreen

**Route:** `/onboarding/family-name`

**Purpose:** Create family account with display name

**Elements:**
- Header: "What should we call your family?"
- Text input field
  - Placeholder: "The Smith Family"
  - Max length: 50 characters
  - Auto-focus on mount
- Continue button

**Actions:**
| Element | Action |
|---------|--------|
| Continue button | Validate input, create family via API, navigate → SetupModeScreen |

**Validation:**
- Required, non-empty
- Trim whitespace

**API Call:**
```
POST /family
{
  displayName: input.value,
  setupMode: null  // set in next screen
}
```

**Error States:**
- API failure: show error, retry option

---

## 5. SetupModeScreen

**Route:** `/onboarding/setup-mode`

**Purpose:** Choose between proactive enrollment and passive learning

**Elements:**
- Header: "How would you like to set up voice recognition?"
- Option card 1 (recommended):
  - Badge: "★ RECOMMENDED"
  - Title: "Sit together and record each family member"
  - Subtitle: "Most accurate, ready in minutes"
- Option card 2:
  - Title: "Let the app learn voices over time"
  - Subtitle: "Takes longer, requires confirmations"

**Actions:**
| Element | Action |
|---------|--------|
| Option 1 tap | Update family setupMode="proactive", navigate → EnrollmentScreen |
| Option 2 tap | navigate → PassiveModeInfoScreen |

**API Call (on selection):**
```
PATCH /family
{
  setupMode: "proactive" | "passive"
}
```

---

## 6. PassiveModeInfoScreen

**Route:** `/onboarding/passive-info`

**Purpose:** Explain passive mode implications

**Elements:**
- Header: "About Passive Learning"
- Body content:
  - The app will listen and identify voices over time
  - You'll be asked to confirm who's speaking
  - Some transcript snippets will be temporarily stored until voices are confirmed
  - This process may take several days
- Two buttons

**Actions:**
| Element | Action |
|---------|--------|
| "I Understand" button | navigate → AlertConfigScreen |
| "Go Back" button | navigate → SetupModeScreen |

---

## 7. EnrollmentScreen

**Route:** `/onboarding/enrollment`

**Purpose:** Manage family member enrollment list

**Elements:**
- Header: "Set Up Family Voices"
- Instruction text: "Add each family member and record their voice"
- Member list (if any enrolled):
  - Member name
  - Status indicator (checkmark if complete)
- "Add Family Member" button
- "Done Adding Members" button (enabled when >= 1 member enrolled)

**Actions:**
| Element | Action |
|---------|--------|
| Add Family Member button | Show name input dialog, then navigate → VoiceCaptureScreen |
| Member row tap | Navigate → VoiceCaptureScreen (re-enroll option) |
| Done Adding Members button | navigate → AlertConfigScreen |

**Validation:**
- At least 1 member must be enrolled before proceeding

---

## 8. VoiceCaptureScreen

**Route:** `/onboarding/voice-capture/{memberId}`

**Purpose:** Record voice sample for voiceprint enrollment

**Elements:**
- Header: "Recording [MemberName]"
- Instruction text: "Ask [MemberName] to speak normally for about 30 seconds"
- Sample prompt text (optional reading material)
- Progress bar (0-100%)
- Current status: "Recording... X seconds captured"
- Volume indicator (real-time feedback)
- Cancel button

**Actions:**
| Element | Action |
|---------|--------|
| Screen mount | Start audio capture, begin enrollment |
| Cancel button | Stop capture, discard, navigate → EnrollmentScreen |
| Progress reaches 100% | Auto-navigate → EnrollmentCompleteDialog |

**API Calls (during capture):**
```
POST /voiceprint/enroll
{
  memberId: memberId,
  audioData: base64,
  audioDurationMs: 5000
}

Response:
{
  enrollmentProgress: 0.35,
  remainingAudioMs: 15000,
  voiceprintStatus: "pending"
}

Repeat until voiceprintStatus = "confirmed"
```

**States:**
- Recording: progress bar animating, volume indicator active
- Processing: after capture, waiting for API confirmation
- Complete: show success, return to EnrollmentScreen

---

## 9. AlertConfigScreen

**Route:** `/onboarding/alerts`

**Purpose:** Configure yelling alert preferences

**Elements:**
- Header: "Gentle Reminders"
- Toggle: "Enable alerts when voices get raised" (default: on)
- Alert type selector (visible if toggle on):
  - Radio: Sound (gentle chime)
  - Radio: Sound + Vibration
  - Radio: Visual only
  - Radio: Vibration only
- Continue button

**Actions:**
| Element | Action |
|---------|--------|
| Toggle change | Update local state |
| Radio selection | Update local state |
| Continue button | Update family via API, navigate → GamificationInfoScreen |

**API Call:**
```
PATCH /family
{
  alertsEnabled: true,
  alertType: "sound_vibration"
}
```

---

## 10. GamificationInfoScreen

**Route:** `/onboarding/gamification-info`

**Purpose:** Introduce tracking & rewards feature (unskippable)

**Elements:**
- Header: "Make it a family challenge!"
- Icon/illustration: trophy or similar
- Body content:
  - YellNo can track progress and let family members earn rewards
  - Set daily, weekly, or monthly challenges
  - Winner picks the movie, dinner, or game
  - See who's most improved
- Footer note: "This feature is optional and off by default"
- Two buttons

**Actions:**
| Element | Action |
|---------|--------|
| "Enable Tracking & Rewards" button | Set gamificationEnabled=true, navigate → GamificationSetupScreen |
| "Skip for Now" button | Set gamificationEnabled=false, navigate → CalibrationInfoScreen |

**Requirements:**
- Screen cannot be dismissed without choosing an option
- Must display for minimum 2 seconds before buttons enable (optional UX choice)

---

## 11. GamificationSetupScreen

**Route:** `/onboarding/gamification-setup`

**Purpose:** Configure gamification details

**Elements:**
- Header: "Set Up Rewards"
- Section: Tracking interval
  - Radio: Daily
  - Radio: Weekly (default)
  - Radio: Monthly
- Section: Award type
  - Radio: Least incidents
  - Radio: Most improved
  - Radio: Both (default)
- Section: Rewards
  - Checkbox: Choose movie (default: checked)
  - Checkbox: Choose dinner (default: checked)
  - Checkbox: Choose game/activity (default: checked)
  - Checkbox: Choose book
  - "+ Add custom reward" button
- Continue button

**Actions:**
| Element | Action |
|---------|--------|
| Add custom reward button | Show text input dialog, add to list |
| Continue button | Update family via API, navigate → CalibrationInfoScreen |

**Validation:**
- At least 1 reward must be selected

**API Call:**
```
PATCH /family
{
  gamificationEnabled: true,
  trackingMode: "both",
  trackingInterval: "weekly",
  rewards: [
    { id: "uuid", label: "Choose movie", isPreset: true, presetType: "movie" },
    { id: "uuid", label: "Choose dinner", isPreset: true, presetType: "dinner" },
    { id: "uuid", label: "Extra screen time", isPreset: false, presetType: null }
  ]
}
```

---

## 12. CalibrationInfoScreen

**Route:** `/onboarding/calibration`

**Purpose:** Set expectations for initial calibration period

**Elements:**
- Header: "Almost ready!"
- Body content:
  - YellNo will learn each person's normal speaking volume over the next 48 hours
  - During this time:
    - Detection may be less accurate
    - You might see some confirmation prompts
    - Accuracy improves as the app learns
- Start button

**Actions:**
| Element | Action |
|---------|--------|
| "Start Using YellNo" button | navigate → PermissionScreen |

---

## 13. PermissionScreen

**Route:** `/onboarding/permissions`

**Purpose:** Request microphone permission

**Elements:**
- Header: "Microphone Access"
- Icon: microphone illustration
- Body: "YellNo needs microphone access to detect raised voices"
- Permission button

**Actions:**
| Element | Action |
|---------|--------|
| "Allow Microphone Access" button | Request system permission |

**Logic:**
```
on permission granted:
  mark setupCompleted = true
  start listening service
  navigate → DashboardScreen

on permission denied:
  show explanation
  show "Open Settings" button
  
on permission permanently denied:
  show explanation
  show "Open Settings" button to app settings
```

**API Call (on success):**
```
PATCH /family
{
  setupCompleted: true
}
```

---

## State Persistence

**Track onboarding progress in local storage:**

```
onboarding_state: {
  currentStep: "family-name" | "setup-mode" | "enrollment" | etc,
  familyId: "uuid" | null,
  membersEnrolled: ["uuid", ...],
  alertsConfigured: bool,
  gamificationConfigured: bool
}
```

**Resume Logic:**
```
if family exists and not setupCompleted:
  check onboarding_state
  navigate to currentStep
```

---

## Back Navigation

| Screen | Back Behavior |
|--------|---------------|
| PrivacyDisclosureScreen | Not allowed (no back button) |
| FamilyNameScreen | Not allowed |
| SetupModeScreen | Not allowed |
| PassiveModeInfoScreen | → SetupModeScreen |
| EnrollmentScreen | Not allowed (must complete or have 1+ member) |
| VoiceCaptureScreen | Cancel → EnrollmentScreen |
| AlertConfigScreen | → SetupModeScreen (proactive) or PassiveModeInfoScreen (passive) |
| GamificationInfoScreen | → AlertConfigScreen |
| GamificationSetupScreen | → GamificationInfoScreen |
| CalibrationInfoScreen | → GamificationInfoScreen or GamificationSetupScreen |
| PermissionScreen | → CalibrationInfoScreen |

---

## Error Handling

| Error | Handling |
|-------|----------|
| API failure (create family) | Show error dialog, retry button |
| API failure (update family) | Show error dialog, retry button |
| API failure (voiceprint enroll) | Show error, option to retry or skip member |
| Microphone permission denied | Show explanation, settings button |
| Network offline | Show offline overlay, pause onboarding |

---

## Analytics Events

| Event | Screen | Data |
|-------|--------|------|
| onboarding_started | PrivacyDisclosureScreen | - |
| privacy_accepted | PrivacyDisclosureScreen | - |
| family_created | FamilyNameScreen | familyId |
| setup_mode_selected | SetupModeScreen | mode: proactive/passive |
| member_enrollment_started | VoiceCaptureScreen | memberId |
| member_enrollment_completed | VoiceCaptureScreen | memberId, durationMs |
| member_enrollment_cancelled | VoiceCaptureScreen | memberId |
| alerts_configured | AlertConfigScreen | enabled, alertType |
| gamification_enabled | GamificationInfoScreen | - |
| gamification_skipped | GamificationInfoScreen | - |
| gamification_configured | GamificationSetupScreen | interval, mode, rewardCount |
| permission_granted | PermissionScreen | - |
| permission_denied | PermissionScreen | - |
| onboarding_completed | PermissionScreen | setupMode, memberCount, gamificationEnabled |      