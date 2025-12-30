# YellNo API Contract

## Base URL

`https://yellno-api.azurewebsites.net/api`

## Authentication

All endpoints require Bearer token authentication via Google OAuth 2.0.

**Header:** `Authorization: Bearer {google_id_token}`

Server validates token with Google and extracts OwnerId (Google account ID).

---

## Endpoints

---

### POST /family

Create a new family account.

**Request:**
```json
{
  "displayName": "string (required)",
  "setupMode": "string (required, enum: proactive, passive)",
  "language": "string (optional, default: en-US)"
}
```

**Response 201:**
```json
{
  "familyId": "string (GUID)",
  "displayName": "string",
  "setupMode": "string",
  "language": "string",
  "alertsEnabled": true,
  "alertType": "sound",
  "gamificationEnabled": false,
  "setupCompleted": false,
  "createdAt": "datetime ISO 8601"
}
```

**Errors:**
- 400: Invalid request body
- 401: Invalid or missing token
- 409: Family already exists for this owner

---

### GET /family

Get current user's family.

**Response 200:**
```json
{
  "familyId": "string",
  "displayName": "string",
  "setupMode": "string",
  "language": "string",
  "alertsEnabled": "bool",
  "alertType": "string",
  "gamificationEnabled": "bool",
  "trackingMode": "string | null",
  "trackingInterval": "string | null",
  "rewards": "array | null",
  "volumeThresholdMultiplier": "float",
  "setupCompleted": "bool",
  "createdAt": "datetime",
  "updatedAt": "datetime"
}
```

**Errors:**
- 401: Invalid or missing token
- 404: No family exists for this owner

---

### PATCH /family

Update family settings.

**Request:**
```json
{
  "displayName": "string (optional)",
  "alertsEnabled": "bool (optional)",
  "alertType": "string (optional, enum: sound, visual, vibration, sound_vibration)",
  "gamificationEnabled": "bool (optional)",
  "trackingMode": "string (optional, enum: least_incidents, most_improved, both)",
  "trackingInterval": "string (optional, enum: daily, weekly, monthly)",
  "rewards": "array (optional)",
  "volumeThresholdMultiplier": "float (optional, range: 1.1-3.0)",
  "setupCompleted": "bool (optional)"
}
```

**Response 200:** Updated family object (same schema as GET /family)

**Errors:**
- 400: Invalid request body
- 401: Invalid or missing token
- 404: No family exists for this owner

---

### DELETE /family

Delete family and all associated data.

**Response 204:** No content

**Side Effects:**
- Deletes all members, incidents, confirmations, devices, stats
- Deletes voiceprint profiles from Azure Speaker Recognition

**Errors:**
- 401: Invalid or missing token
- 404: No family exists for this owner

---

### POST /family/members

Add a new family member.

**Request:**
```json
{
  "displayName": "string (required)"
}
```

**Response 201:**
```json
{
  "memberId": "string (GUID)",
  "displayName": "string",
  "voiceprintStatus": "none",
  "baselineVolume": null,
  "createdAt": "datetime"
}
```

**Errors:**
- 400: Invalid request body
- 401: Invalid or missing token
- 404: No family exists

---

### GET /family/members

Get all family members.

**Response 200:**
```json
{
  "members": [
    {
      "memberId": "string",
      "displayName": "string",
      "voiceprintStatus": "string (enum: none, pending, confirmed)",
      "baselineVolume": "float | null",
      "baselineSampleCount": "int",
      "isActive": "bool",
      "createdAt": "datetime"
    }
  ]
}
```

**Errors:**
- 401: Invalid or missing token
- 404: No family exists

---

### PATCH /family/members/{memberId}

Update a family member.

**Request:**
```json
{
  "displayName": "string (optional)",
  "isActive": "bool (optional)"
}
```

**Response 200:** Updated member object

**Errors:**
- 400: Invalid request body
- 401: Invalid or missing token
- 404: Member not found

---

### DELETE /family/members/{memberId}

Delete a family member.

**Response 204:** No content

**Side Effects:**
- Deletes voiceprint profile from Azure Speaker Recognition
- Incidents remain but MemberId set to null

**Errors:**
- 401: Invalid or missing token
- 404: Member not found

---

### POST /voiceprint/enroll

Start voiceprint enrollment for a member.

**Request:**
```json
{
  "memberId": "string (required)",
  "audioData": "string (required, base64 encoded WAV)",
  "audioDurationMs": "int (required)"
}
```

**Audio Requirements:**
- Format: WAV, 16kHz, 16-bit, mono
- Minimum duration: 5000ms
- Recommended total: 20000ms across multiple calls

**Response 200:**
```json
{
  "memberId": "string",
  "voiceprintStatus": "string (pending | confirmed)",
  "enrollmentProgress": "float (0.0-1.0)",
  "remainingAudioMs": "int",
  "baselineVolume": "float | null",
  "message": "string"
}
```

**Notes:**
- Multiple calls may be needed to reach confirmed status
- Baseline volume is calculated from enrollment audio
- Returns confirmed status when enrollmentProgress >= 1.0

**Errors:**
- 400: Invalid audio format or duration
- 401: Invalid or missing token
- 404: Member not found

---

### POST /voiceprint/identify

Identify speaker from audio sample.

**Request:**
```json
{
  "audioData": "string (required, base64 encoded WAV)",
  "audioDurationMs": "int (required)"
}
```

**Response 200:**
```json
{
  "identified": "bool",
  "memberId": "string | null",
  "memberDisplayName": "string | null",
  "confidence": "float (0.0-1.0)",
  "needsConfirmation": "bool",
  "confirmationId": "string | null"
}
```

**Notes:**
- If needsConfirmation=true, a PendingConfirmation record was created
- Client should prompt user to confirm identification

**Errors:**
- 400: Invalid audio format
- 401: Invalid or missing token
- 404: No family exists

---

### GET /voiceprint/confirmations

Get pending voiceprint confirmations.

**Response 200:**
```json
{
  "confirmations": [
    {
      "confirmationId": "string",
      "confirmationType": "string (enum: new_speaker, verify_speaker)",
      "transcriptSnippet": "string",
      "proposedMemberId": "string | null",
      "proposedMemberName": "string | null",
      "createdAt": "datetime",
      "expiresAt": "datetime"
    }
  ]
}
```

**Errors:**
- 401: Invalid or missing token
- 404: No family exists

---

### POST /voiceprint/confirmations/{confirmationId}

Respond to a voiceprint confirmation.

**Request:**
```json
{
  "confirmed": "bool (required)",
  "memberId": "string (required if confirmationType=new_speaker and confirmed=true)",
  "newMemberName": "string (required if confirmationType=new_speaker and confirmed=true and memberId not provided)"
}
```

**Response 200:**
```json
{
  "success": true,
  "memberId": "string | null",
  "voiceprintStatus": "string | null"
}
```

**Notes:**
- For new_speaker: provide existing memberId OR newMemberName to create member
- For verify_speaker: just confirmed true/false

**Errors:**
- 400: Invalid request
- 401: Invalid or missing token
- 404: Confirmation not found or expired

---

### POST /analyze

Analyze audio for yelling detection.

**Request:**
```json
{
  "audioData": "string (required, base64 encoded WAV)",
  "audioDurationMs": "int (required)",
  "deviceId": "string (required)",
  "capturedAt": "datetime (required, ISO 8601)",
  "measuredVolumeDb": "float (required)"
}
```

**Response 200:**
```json
{
  "isYelling": "bool",
  "confidence": "float (0.0-1.0)",
  "memberId": "string | null",
  "memberDisplayName": "string | null",
  "speakerIdentified": "bool",
  "transcriptSnippet": "string | null",
  "detectedMood": "string | null (enum: angry, frustrated, excited, unknown)",
  "incidentId": "string | null",
  "isDuplicate": "bool",
  "needsSpeakerConfirmation": "bool",
  "confirmationId": "string | null"
}
```

**Notes:**
- If isYelling=true and isDuplicate=false, incident was recorded
- incidentId is null if duplicate or not yelling
- Audio is discarded after processing

**Errors:**
- 400: Invalid audio format
- 401: Invalid or missing token
- 404: No family exists

---

### GET /incidents

Get yelling incidents.

**Query Parameters:**
- `limit`: int (optional, default: 50, max: 200)
- `before`: datetime ISO 8601 (optional, for pagination)
- `memberId`: string (optional, filter by member)
- `includeRemoved`: bool (optional, default: false)

**Response 200:**
```json
{
  "incidents": [
    {
      "incidentId": "string",
      "memberId": "string | null",
      "memberDisplayName": "string | null",
      "timestamp": "datetime",
      "confidence": "float",
      "volumeLevel": "float",
      "transcriptSnippet": "string | null",
      "detectedMood": "string | null",
      "removed": "bool"
    }
  ],
  "hasMore": "bool",
  "nextBefore": "datetime | null"
}
```

**Errors:**
- 401: Invalid or missing token
- 404: No family exists

---

### DELETE /incidents/{incidentId}

Remove a yelling incident.

**Request:**
```json
{
  "confirmationStep": "int (required, 1-3)",
  "gamificationEnabled": "bool (required)"
}
```

**Response 200:**
```json
{
  "removed": "bool",
  "nextStep": "int | null",
  "message": "string"
}
```

**Flow:**
- Step 1: Returns nextStep=2, message="Are you sure this wasn't yelling?"
- Step 2: Returns nextStep=3, message="If you aren't sure, it's better to let it remain..."
- Step 3 (if gamificationEnabled): Returns removed=true after final warning
- Step 2 (if !gamificationEnabled): Returns removed=true

**Errors:**
- 400: Invalid step
- 401: Invalid or missing token
- 404: Incident not found

---

### GET /stats

Get statistics for dashboard/leaderboard.

**Query Parameters:**
- `interval`: string (required, enum: daily, weekly, monthly)
- `periodStart`: datetime ISO 8601 (optional, defaults to current period)
- `periodsBack`: int (optional, default: 1, max: 12)

**Response 200:**
```json
{
  "interval": "string",
  "periods": [
    {
      "periodStart": "datetime",
      "periodEnd": "datetime",
      "familyTotal": "int",
      "members": [
        {
          "memberId": "string",
          "displayName": "string",
          "incidentCount": "int",
          "previousCount": "int | null",
          "improvement": "float | null"
        }
      ]
    }
  ]
}
```

**Errors:**
- 400: Invalid interval
- 401: Invalid or missing token
- 404: No family exists

---

### POST /devices

Register a device.

**Request:**
```json
{
  "deviceId": "string (required)",
  "deviceType": "string (required, enum: android_phone, android_tv)",
  "deviceName": "string (optional)"
}
```

**Response 201:**
```json
{
  "deviceId": "string",
  "deviceType": "string",
  "deviceName": "string | null",
  "createdAt": "datetime"
}
```

**Errors:**
- 400: Invalid request
- 401: Invalid or missing token
- 404: No family exists

---

### POST /devices/{deviceId}/heartbeat

Update device last seen timestamp.

**Response 204:** No content

**Errors:**
- 401: Invalid or missing token
- 404: Device not found

---

### DELETE /devices/{deviceId}

Remove a device.

**Response 204:** No content

**Errors:**
- 401: Invalid or missing token
- 404: Device not found

---

## Error Response Format

All errors return:

```json
{
  "error": {
    "code": "string",
    "message": "string"
  }
}
```

**Common Error Codes:**
- `invalid_request`: Malformed request body
- `unauthorized`: Invalid or missing auth token
- `not_found`: Resource not found
- `conflict`: Resource already exists
- `internal_error`: Server error