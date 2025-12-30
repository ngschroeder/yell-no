# YellNo Data Model Specification

## Storage: Azure Table Storage

---

## Entity: Family

**Table Name:** `families`

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| PartitionKey | string | yes | OwnerId (Google account ID) |
| RowKey | string | yes | FamilyId (GUID) |
| OwnerId | string | yes | Google account ID of app installer |
| OwnerEmail | string | yes | Google account email |
| DisplayName | string | yes | User-defined family name |
| CreatedAt | datetime | yes | ISO 8601 |
| UpdatedAt | datetime | yes | ISO 8601 |
| AlertsEnabled | bool | yes | Default: true |
| AlertType | string | yes | Enum: "sound", "visual", "vibration", "sound_vibration" |
| GamificationEnabled | bool | yes | Default: false |
| TrackingMode | string | no | Enum: "least_incidents", "most_improved", "both". Null if GamificationEnabled=false |
| TrackingInterval | string | no | Enum: "daily", "weekly", "monthly". Null if GamificationEnabled=false |
| Rewards | JSON string | no | Array of reward objects. Null if GamificationEnabled=false |
| Language | string | yes | Default: "en-US". Future: "es", "hi", "pa" |
| VolumeThresholdMultiplier | float | yes | Default: 1.5. Yelling threshold = baseline * this value |
| SetupMode | string | yes | Enum: "proactive", "passive" |
| SetupCompleted | bool | yes | Default: false |

**Reward Object Schema:**
```json
{
  "id": "string (GUID)",
  "label": "string",
  "isPreset": "bool",
  "presetType": "string | null (enum: movie, dinner, game, book)"
}
```

---

## Entity: Member

**Table Name:** `members`

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| PartitionKey | string | yes | FamilyId |
| RowKey | string | yes | MemberId (GUID) |
| DisplayName | string | yes | User-defined name for this family member |
| VoiceprintProfileId | string | no | Azure Speaker Recognition profile ID. Null if not enrolled |
| VoiceprintStatus | string | yes | Enum: "none", "pending", "confirmed" |
| BaselineVolume | float | no | Calibrated normal speaking volume (dB). Null until calibrated |
| BaselineSampleCount | int | yes | Number of non-yelling samples used for baseline. Default: 0 |
| CreatedAt | datetime | yes | ISO 8601 |
| UpdatedAt | datetime | yes | ISO 8601 |
| IsActive | bool | yes | Default: true. Soft delete flag |

---

## Entity: DedupeWindow

**Table Name:** `dedupewindow`

Short-lived entries for detecting duplicate yelling incidents across devices.

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| PartitionKey | string | yes | FamilyId |
| RowKey | string | yes | DedupeHash |
| IncidentId | string | yes | The incident this hash belongs to |
| CreatedAt | datetime | yes | ISO 8601 |
| ExpiresAt | datetime | yes | CreatedAt + 30 seconds |

**Cleanup:** Azure Function timer trigger runs every 60 seconds, deletes rows where ExpiresAt < now.

**Usage:** Before recording incident, attempt to fetch by PartitionKey + RowKey. If exists, discard as duplicate. If not exists, insert dedupe row and proceed with incident recording.

---

## Entity: Incident

**Table Name:** `incidents`

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| PartitionKey | string | yes | FamilyId |
| RowKey | string | yes | InvertedTimestamp_IncidentId for recent-first ordering |
| IncidentId | string | yes | GUID |
| MemberId | string | no | Null if speaker unidentified |
| MemberDisplayName | string | no | Denormalized for query efficiency. Null if unidentified |
| Timestamp | datetime | yes | ISO 8601. When yelling occurred |
| DedupeHash | string | yes | SHA256(first_4_words_lowercase + timestamp_10sec_window + memberId_or_unknown) |
| Confidence | float | yes | 0.0-1.0. Detection confidence score |
| VolumeLevel | float | yes | Measured volume (dB) |
| BaselineAtTime | float | no | Speaker's baseline at detection time. Null if unidentified |
| TranscriptSnippet | string | no | First 4-5 words. Null if transcription failed |
| DetectedMood | string | no | Enum: "angry", "frustrated", "excited", "unknown" |
| SourceDeviceId | string | yes | Device that reported the incident |
| Removed | bool | yes | Default: false |
| RemovedAt | datetime | no | ISO 8601. Null unless removed |

**RowKey Format:**
- InvertedTimestamp = String(DateTime.MaxValue.Ticks - Timestamp.Ticks)
- Full RowKey = `{InvertedTimestamp}_{IncidentId}`
- This ensures newest incidents sort first in Azure Table queries

---

## Entity: PendingConfirmation

**Table Name:** `pendingconfirmations`

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| PartitionKey | string | yes | FamilyId |
| RowKey | string | yes | ConfirmationId (GUID) |
| ConfirmationType | string | yes | Enum: "new_speaker", "verify_speaker" |
| TranscriptSnippet | string | yes | First 4-5 words spoken |
| ProposedMemberId | string | no | Null for "new_speaker", MemberId for "verify_speaker" |
| TentativeVoiceprintId | string | no | Temporary Azure profile ID for new speakers |
| CreatedAt | datetime | yes | ISO 8601 |
| ExpiresAt | datetime | yes | CreatedAt + 7 days |

---

## Entity: Device

**Table Name:** `devices`

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| PartitionKey | string | yes | FamilyId |
| RowKey | string | yes | DeviceId (generated on app install) |
| DeviceType | string | yes | Enum: "android_phone", "android_tv" |
| DeviceName | string | no | User-friendly name (e.g., "Living Room TV") |
| LastSeenAt | datetime | yes | ISO 8601. Last heartbeat |
| IsActive | bool | yes | Default: true |
| CreatedAt | datetime | yes | ISO 8601 |

---

## Entity: StatsSummary

**Table Name:** `statssummaries`

Precomputed statistics for efficient dashboard queries.

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| PartitionKey | string | yes | Format: `{FamilyId}_{Interval}` |
| RowKey | string | yes | Format: `{PeriodStart}_{MemberId_or_FAMILY}` |
| MemberId | string | no | Null for family-wide stats |
| Interval | string | yes | Enum: "daily", "weekly", "monthly" |
| PeriodStart | datetime | yes | ISO 8601. Start of period |
| PeriodEnd | datetime | yes | ISO 8601. End of period |
| IncidentCount | int | yes | Total incidents in period |
| RemovedCount | int | yes | Incidents removed by user |
| AverageConfidence | float | yes | Mean confidence of detections |

---

## Indexes and Query Patterns

**Primary Query Patterns:**

1. Get family by OwnerId: Point query `families` where PartitionKey = OwnerId (O(1) lookup)
2. Get all members for family: Query `members` where PartitionKey = FamilyId
3. Get recent incidents: Query `incidents` where PartitionKey = FamilyId (RowKey sorts newest first)
4. Get incidents for member: Query `incidents` where PartitionKey = FamilyId, filter by MemberId
5. Get pending confirmations: Query `pendingconfirmations` where PartitionKey = FamilyId
6. Check dedupe: Point query `dedupewindow` where PartitionKey = FamilyId, RowKey = DedupeHash (O(1) lookup)
7. Get stats for leaderboard: Query `statssummaries` where PartitionKey = `{FamilyId}_{Interval}`, filter by PeriodStart

**Scalability Notes:**

- Family lookup is O(1) via OwnerId partition
- Dedupe check is O(1) via direct hash lookup in dedicated table
- Each family's data is isolated in its own partition(s), preventing cross-family interference
- StatsSummary partitioned by FamilyId + Interval keeps leaderboard queries within single partition
- No table scans required for any primary query pattern

---

## Blob Storage: Voiceprints

**Container:** `voiceprints`

Voiceprint data is managed by Azure Speaker Recognition service. The `VoiceprintProfileId` in the Member entity references the Azure-managed profile.

No direct blob storage access required—all operations go through Azure Speaker Recognition API.