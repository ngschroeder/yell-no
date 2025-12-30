# YellNo Architecture Document
## Family Yelling Reduction App

**Version:** 0.1 (Draft)
**Last Updated:** December 2024

---

## 1. Overview

YellNo is a privacy-first application that helps families reduce yelling by detecting raised voices, providing gentle real-time reminders, and optionally gamifying progress through tracking and rewards. The app runs on multiple household devices to maximize coverage while ensuring no audio recordings are retained.

### Core Principles

- **Privacy-first**: Audio is processed in real-time and immediately discarded. Only metadata and statistics are stored.
- **Family-as-a-system**: The app treats yelling as a household pattern, not an individual failing.
- **Configurable**: Families choose their level of engagement (alerts only, tracking, gamification).
- **Non-judgmental**: Messaging is gentle and supportive regardless of household yelling frequency.

---

## 2. Target Platforms

### Phase 1
- Android phones
- Android TV

### Future Phases
- iOS
- Web (for dashboard/stats viewing)
- Additional TV platforms (Fire TV, Roku, etc.)

### Technology Choice

**Flutter** for cross-platform development. Rationale:
- Single codebase for Android phone and Android TV
- Strong Azure integration via REST APIs
- Path to iOS and web with minimal rework
- Mature ecosystem for audio handling

---

## 3. High-Level Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                        CLIENT DEVICES                           │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐             │
│  │ Phone App   │  │ Phone App   │  │ Android TV  │             │
│  │ (Parent)    │  │ (Teen)      │  │ App         │             │
│  └──────┬──────┘  └──────┬──────┘  └──────┬──────┘             │
│         │                │                │                     │
│         └────────────────┼────────────────┘                     │
│                          │                                      │
└──────────────────────────┼──────────────────────────────────────┘
                           │ HTTPS
                           ▼
┌─────────────────────────────────────────────────────────────────┐
│                      AZURE BACKEND                              │
│                                                                 │
│  ┌─────────────────┐    ┌─────────────────┐                    │
│  │ Azure Functions │    │ Azure Speech    │                    │
│  │ (API Layer)     │◄──►│ Services        │                    │
│  └────────┬────────┘    │ - Speech-to-Text│                    │
│           │             │ - Speaker ID    │                    │
│           │             └─────────────────┘                    │
│           ▼                                                     │
│  ┌─────────────────┐    ┌─────────────────┐                    │
│  │ Azure Table     │    │ Azure Blob      │                    │
│  │ Storage         │    │ Storage         │                    │
│  │ (Statistics)    │    │ (Voiceprints)   │                    │
│  └─────────────────┘    └─────────────────┘                    │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 4. Component Details

### 4.1 Client Application (Flutter)

**Responsibilities:**
- Continuous audio monitoring (when app is active/backgrounded)
- Local audio processing for initial volume detection
- Streaming audio to Azure for analysis (only when volume threshold suggests potential yelling)
- Displaying alerts, statistics, and gamification UI
- Managing local state and syncing with cloud

**Key Modules:**

| Module | Purpose |
|--------|---------|
| Audio Capture | Continuous microphone access, background audio processing |
| Volume Analyzer | Local volume level detection to trigger cloud analysis |
| Alert Manager | Gentle notification/sound when yelling detected |
| Sync Manager | Bi-directional sync with Azure backend |
| UI Layer | Setup wizard, dashboard, family stats, gamification views |

**Background Processing:**
- Android: Foreground service with persistent notification ("YellNo is listening")
- Audio buffer: Rolling 5-10 second window, never persisted to storage

### 4.2 Azure Functions (API Layer)

**Endpoints:**

| Endpoint | Purpose |
|----------|---------|
| POST /analyze | Receive audio clip, return analysis (speaker, mood, yelling determination) |
| POST /incident | Record a confirmed yelling incident |
| DELETE /incident/{id} | Remove incident (with confirmation flags) |
| GET /stats | Retrieve family/individual statistics |
| POST /voiceprint | Create or update speaker voiceprint |
| GET /voiceprint/confirm | Get pending voiceprint confirmations |
| POST /voiceprint/confirm | Confirm or reject a voiceprint match |
| POST /family | Create family account |
| PUT /family/members | Add/update family members |
| GET /family/rewards | Get reward configuration |
| PUT /family/rewards | Update reward settings |

### 4.3 Azure Speech Services

**Services Used:**

1. **Speech-to-Text**: Transcription for mood analysis and de-duplication
2. **Speaker Recognition**: Voiceprint creation and matching
   - Text-independent verification (doesn't require specific phrases)
   - Enrollment requires ~20 seconds of speech for reliable identification

**Processing Flow:**
1. Client detects elevated volume locally
2. Audio clip (3-5 seconds) sent to Azure Function
3. Function calls Speech Services for transcription + speaker identification
4. Function applies mood/tone analysis (see section 4.4)
5. Results returned to client; audio immediately discarded

### 4.4 Mood/Tone Analysis

**Approach:** Hybrid analysis combining multiple signals

| Signal | Source | Weight |
|--------|--------|--------|
| Volume | Client-side measurement | High |
| Speech rate | Derived from transcription timing | Medium |
| Pitch variation | Audio analysis | Medium |
| Keyword detection | Transcription (aggressive words, expletives) | Low |
| Speaker baseline deviation | Comparison to stored baseline | High |

**Initial Implementation:**
- Azure Speech provides base transcription and timing
- Custom logic in Azure Function for volume/baseline comparison
- Future: Azure AI Language for sentiment, or custom ML model

**Per-Speaker Calibration:**
- Baseline volume captured during voiceprint enrollment
- Rolling average updated as speaker is detected in non-yelling contexts
- Yelling threshold = baseline + configurable multiplier (default: 1.5x)

### 4.5 Data Storage

**Azure Table Storage Schema:**

*Family Table*
| Field | Type | Notes |
|-------|------|-------|
| PartitionKey | string | "family" |
| RowKey | string | FamilyId (GUID) |
| Name | string | Family display name |
| CreatedAt | datetime | |
| Settings | JSON | Alert preferences, gamification config |

*Member Table*
| Field | Type | Notes |
|-------|------|-------|
| PartitionKey | string | FamilyId |
| RowKey | string | MemberId (GUID) |
| Name | string | Display name |
| VoiceprintId | string | Reference to Azure Speaker Recognition profile |
| BaselineVolume | float | Calibrated normal speaking volume |
| VoiceprintConfirmed | bool | Whether voiceprint is reliably established |
| CreatedAt | datetime | |

*Incident Table*
| Field | Type | Notes |
|-------|------|-------|
| PartitionKey | string | FamilyId |
| RowKey | string | Timestamp (inverted for recent-first) + IncidentId |
| MemberId | string | Nullable (if speaker unidentified) |
| Timestamp | datetime | When incident occurred |
| DedupeHash | string | Hash of first N words + 10-sec window |
| Confidence | float | How confident the detection was |
| Removed | bool | If user disputed and removed |
| RemovedReason | string | Optional note |

*Pending Confirmation Table*
| Field | Type | Notes |
|-------|------|-------|
| PartitionKey | string | FamilyId |
| RowKey | string | ConfirmationId |
| TranscriptSnippet | string | First 4-5 words for user confirmation |
| ProposedMemberId | string | Who the system thinks spoke |
| CreatedAt | datetime | |
| ExpiresAt | datetime | Auto-cleanup after 7 days |

**Azure Blob Storage:**
- Voiceprint enrollment data (managed by Azure Speaker Recognition)
- No raw audio stored

### 4.6 De-duplication Strategy

**Problem:** Multiple devices may detect the same yelling incident.

**Solution:**
1. Generate hash from: `SHA256(first_4_words_lowercase + timestamp_rounded_to_10sec)`
2. If no voiceprint available, add "unknown" to hash
3. If voiceprint available, add MemberId to hash
4. Before recording incident, check if hash exists in last 30 seconds
5. If duplicate found, discard; otherwise record

**Edge Cases:**
- Different speakers yelling same words within 10 seconds: Differentiated by voiceprint or recorded as separate incidents
- Same speaker, different rooms, slight timestamp drift: 10-second window should absorb typical drift

---

## 5. User Flows

### 5.1 Onboarding

1. **Welcome & Privacy Disclosure**
   - Clear explanation: no audio saved, only statistics
   - Explicit consent required to proceed

2. **Family Setup**
   - Create family name
   - Choose setup mode:
     - **Recommended:** Sit together, enroll each family member's voice
     - **Passive:** App learns over time (warning about slower accuracy, confirmation prompts, temporary transcript retention)

3. **Voiceprint Enrollment (if recommended path)**
   - Each family member speaks for ~30 seconds (reading prompts or free speech)
   - System captures voiceprint AND baseline volume simultaneously
   - Confirmation: "Got it! [Name] is set up."

4. **Alert Configuration**
   - Toggle: Gentle yelling reminders (on/off)
   - Choose alert type: sound, visual, vibration

5. **Gamification Introduction**
   - Unskippable informational screen explaining the feature
   - Toggle: Enable tracking & rewards (off by default)
   - If enabled: Configure reward presets or custom rewards, tracking interval (daily/weekly/monthly)

6. **Calibration Notice**
   - Brief explanation that accuracy improves over 48 hours
   - Set expectations for occasional confirmation prompts

### 5.2 Passive Voiceprint Learning

1. App detects speech from unrecognized speaker
2. After enough samples (~20 seconds cumulative), system creates tentative voiceprint
3. App prompts primary account holder: "Someone just said '[first four words]...' — who was this?"
4. User assigns name → voiceprint linked to new family member
5. For next few detections, confirmation prompts: "Was that [Name]?"
6. After 3-5 confirmations, voiceprint marked as confirmed

### 5.3 Yelling Detection

1. Client detects volume spike above ambient threshold
2. 3-5 second audio clip sent to backend
3. Backend processes:
   - Transcription
   - Speaker identification
   - Mood/tone analysis
   - Baseline comparison
4. If yelling confirmed:
   - Generate de-dupe hash
   - Check for duplicates
   - If unique: record incident, notify client
5. Client displays gentle alert (if enabled)
6. Audio discarded server-side

### 5.4 Incident Removal

1. User views incident in app
2. Taps "This wasn't yelling"
3. **Confirmation 1:** "Are you sure this wasn't yelling?"
4. **Confirmation 2:** "If you aren't sure, it's better to let it remain as a yelling instance to preserve accuracy."
5. **Confirmation 3 (if gamification enabled):** "Removing a yelling instance just to change the tally could disrupt app performance and reduce accuracy over time. Please only confirm if this was 100% a mistaken identification."
6. If confirmed through all steps: incident marked as Removed (soft delete for analytics)

---

## 6. Privacy & Security

### 6.1 Data Handling

| Data Type | Retention | Storage Location |
|-----------|-----------|------------------|
| Raw audio | Never stored | Memory only, discarded after processing |
| Transcripts (passive mode) | Until voiceprint confirmed, max 7 days | Azure Table (encrypted) |
| Transcript snippets (for confirmation) | Until confirmed or 7 days | Azure Table (encrypted) |
| Voiceprints | Until account deletion | Azure Speaker Recognition (encrypted) |
| Incident statistics | Until account deletion | Azure Table (encrypted) |

### 6.2 Security Measures

- All client-server communication over HTTPS
- Google Sign-In for authentication (Apple ID in future phases)
- Data encrypted at rest (Azure default)
- No PII in logs
- Family data isolated by FamilyId partition

### 6.3 User Controls

- Delete individual incidents
- Delete individual family member (and their voiceprint)
- Delete entire account and all associated data
- Export personal data (GDPR compliance)

---

## 7. Gamification Details

### 7.1 Tracking Modes

- **Least incidents:** Person with fewest yelling incidents in period wins
- **Most improved:** Person with largest reduction vs. previous period wins
- **Both:** Dual awards

### 7.2 Reward Presets

- Choose tonight's movie
- Choose tonight's dinner
- Choose family game/activity
- Choose book for reading time
- Custom (user-defined text)

### 7.3 Intervals

- Daily
- Weekly
- Monthly

### 7.4 Display

- In-app leaderboard showing current standings
- End-of-period notification announcing winner
- Historical stats viewable per family member

---

## 8. Cost Considerations

### Azure Services (Estimated)

| Service | Free Tier | Pay-As-You-Go |
|---------|-----------|---------------|
| Azure Functions | 1M executions/month | ~$0.20 per million after |
| Azure Speech (STT) | 5 hours/month | $1.00 per hour after |
| Azure Speaker Recognition | 10K transactions/month | $1.00 per 1K after |
| Azure Table Storage | 5GB, 20K transactions | Negligible |
| Azure Blob Storage | 5GB | Negligible |

**Volume Gate Threshold:**
- Client continuously monitors ambient audio levels
- Audio processing and cloud transmission only triggered when human speech exceeds configurable volume threshold
- Threshold should be above normal conversational volume to avoid unnecessary API calls
- This is the primary cost control mechanism

---

## 9. Design Decisions

1. **Offline mode:** No offline functionality. App displays "Offline" overlay and pauses all processing until connectivity is restored.

2. **Multi-language support:** English at launch. Architecture supports future expansion to Spanish, Hindi, Punjabi, and other languages via Azure Speech's multi-language capabilities. Mood analysis keyword lists will need localization per language.

3. **Wearables:** Out of scope.

4. **Account model:** Single account owner (app installer) who manages all family member profiles. Family members do not have their own accounts—they are simply identified voices labeled by the owner. All voiceprint confirmations and incident removals are handled by the account owner.

5. **Intervention copy:** Deferred to UX writing phase.

---

## 10. Next Steps

1. Review and finalize architecture
2. Design detailed UI/UX wireframes
3. Set up Azure infrastructure (Functions, Storage, Speech Services)
4. Build Flutter app scaffold with audio capture
5. Implement voiceprint enrollment flow
6. Implement yelling detection pipeline
7. Build gamification features
8. Beta testing with select families