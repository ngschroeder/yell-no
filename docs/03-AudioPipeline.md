# YellNo Audio Processing Pipeline

## Overview

Audio flows from device microphone through local filtering to cloud analysis. No audio is persisted. Processing occurs only when volume thresholds suggest potential yelling.

---

## Client-Side Processing

### Audio Capture

**Configuration:**
- Sample rate: 16kHz
- Bit depth: 16-bit
- Channels: Mono
- Buffer size: 100ms chunks
- Rolling window: 5 seconds (50 chunks)

**Android Implementation:**
- Foreground service with persistent notification
- `AudioRecord` class with `VOICE_RECOGNITION` audio source
- Continuous capture while app is active/backgrounded

### Volume Monitoring

**Continuous Calculation:**
```
For each 100ms chunk:
  1. Calculate RMS amplitude
  2. Convert to dB: dB = 20 * log10(RMS / reference)
  3. Update rolling average (last 3 seconds of non-elevated audio)
  4. Compare current dB to threshold
```

**Threshold Logic:**
```
ambientBaseline = rolling average of recent non-elevated chunks
speakerBaseline = stored baseline for identified speaker (if available)
globalThreshold = configurable minimum dB to consider as potential yelling

triggerThreshold = max(
  ambientBaseline + 15dB,
  speakerBaseline * volumeThresholdMultiplier (default 1.5),
  globalThreshold
)

if currentVolumeDb >= triggerThreshold:
  trigger cloud analysis
```

### Trigger Decision

**When volume exceeds threshold:**

1. Continue capturing for 3 seconds total (or until volume drops for 1+ second)
2. Extract audio segment from rolling buffer
3. Check local cooldown (no duplicate sends within 5 seconds)
4. If not in cooldown, send to cloud

**Local State:**
```
lastAnalysisSentAt: datetime
cooldownPeriodMs: 5000
rollingBuffer: circular buffer of last 5 seconds
isCapturing: bool
captureStartedAt: datetime
```

---

## Cloud Transmission

### Request Preparation

**Audio Encoding:**
1. Extract relevant audio segment (3-5 seconds)
2. Encode as WAV format
3. Base64 encode for JSON transport

**Request Payload:**
```json
{
  "audioData": "base64 WAV data",
  "audioDurationMs": 3500,
  "deviceId": "device-uuid",
  "capturedAt": "2024-01-15T14:32:05.123Z",
  "measuredVolumeDb": 78.5
}
```

**Transmission:**
- HTTPS POST to `/analyze` endpoint
- Timeout: 10 seconds
- Retry: 1 retry with exponential backoff on network failure
- On persistent failure: discard and log locally

---

## Server-Side Processing

### Azure Function: Analyze

**Trigger:** HTTP POST

**Processing Steps:**

```
1. AUTHENTICATE
   - Validate Bearer token with Google
   - Extract OwnerId
   - Fetch FamilyId from families table

2. DECODE AUDIO
   - Base64 decode audioData
   - Validate WAV format and duration

3. PARALLEL PROCESSING (await all)
   
   3a. TRANSCRIPTION (Azure Speech-to-Text)
       - Send audio to Azure Speech Services
       - Receive: transcript text, word timings, confidence
       - Extract first 4-5 words for snippet
   
   3b. SPEAKER IDENTIFICATION (Azure Speaker Recognition)
       - Send audio to Speaker Recognition API
       - Compare against enrolled profiles for this family
       - Receive: profileId match (if any), confidence score
       - Map profileId to MemberId

4. MOOD ANALYSIS
   Input: transcript, word timings, measuredVolumeDb, speakerBaseline
   
   Calculate mood signals:
   - volumeDeviation = measuredVolumeDb / speakerBaseline (if known)
   - speechRate = words per second from timings
   - pitchVariation = (future: extract from audio)
   - aggressiveKeywords = count of flagged words in transcript
   
   Mood classification:
   - "angry": high volume + high speech rate + aggressive keywords
   - "frustrated": high volume + moderate indicators
   - "excited": high volume + positive/neutral keywords
   - "unknown": insufficient signals

5. YELLING DETERMINATION
   
   isYelling = (
     measuredVolumeDb >= triggerThreshold AND
     (volumeDeviation >= 1.3 OR speakerBaseline unknown) AND
     (detectedMood in ["angry", "frustrated"] OR confidence >= 0.7)
   )
   
   confidence = weighted average of:
     - volume confidence: 0.4
     - mood confidence: 0.3
     - speaker identification confidence: 0.2
     - speech rate deviation: 0.1

6. DEDUPLICATION CHECK
   If isYelling:
     - Generate hash: SHA256(transcript_first_4_words_lower + timestamp_10sec + memberId_or_unknown)
     - Query dedupewindow table: PartitionKey=FamilyId, RowKey=hash
     - If exists: isDuplicate=true, skip recording
     - If not exists: insert dedupe record with 30sec TTL

7. RECORD INCIDENT
   If isYelling AND NOT isDuplicate:
     - Generate IncidentId
     - Insert into incidents table
     - Update statssummaries (increment counts)

8. SPEAKER CONFIRMATION CHECK
   If speaker not confidently identified (confidence < 0.7):
     - Create PendingConfirmation record
     - Set needsSpeakerConfirmation=true in response

9. DISCARD AUDIO
   - Audio buffer released from memory
   - No persistence of audio data

10. RETURN RESPONSE
```

### Azure Services Integration

**Speech-to-Text:**
```
Endpoint: {region}.stt.speech.microsoft.com
API: Speech SDK or REST
Model: Default conversation model
Language: from family.language setting

Request:
  - Audio: WAV stream
  - Config: word-level timestamps enabled

Response:
  - transcript: string
  - words: [{word, startMs, endMs, confidence}]
  - overallConfidence: float
```

**Speaker Recognition:**
```
Endpoint: {region}.api.cognitive.microsoft.com/speaker
API: Speaker Recognition REST API
Mode: Text-independent identification

Request:
  - Audio: WAV stream
  - ProfileIds: all confirmed voiceprint IDs for family

Response:
  - identifiedProfile: {profileId, confidence} | null
  - candidates: [{profileId, confidence}]
```

---

## Response Handling (Client)

### On Analysis Response

```
If response.isYelling AND NOT response.isDuplicate:
  If family.alertsEnabled:
    Trigger alert based on family.alertType:
      - "sound": play gentle chime
      - "visual": show overlay notification
      - "vibration": vibrate device
      - "sound_vibration": both
    
    Display brief message: "Take a breath 💙"
  
  Update local incident cache
  Refresh stats if dashboard visible

If response.needsSpeakerConfirmation:
  Queue confirmation prompt for user:
    "Did [proposedName] just say '[transcriptSnippet]...'?"
    OR (if new speaker)
    "Who just said '[transcriptSnippet]...'?"

If response.isDuplicate:
  No action (another device already recorded)
```

### Offline Handling

```
If network unavailable:
  Display "Offline" overlay
  Pause audio capture
  Stop all processing
  
  On network restore:
    Remove overlay
    Resume audio capture
    Resume normal operation
  
  No offline queuing or storage
```

---

## Voiceprint Enrollment Flow

### Proactive Enrollment

```
1. User initiates enrollment for member
2. App prompts: "Ask [Name] to speak normally for about 30 seconds"
3. App captures audio in 5-second segments
4. Each segment sent to POST /voiceprint/enroll
5. Response indicates progress (0.0 to 1.0)
6. Continue until status="confirmed" or user cancels
7. Baseline volume calculated from enrollment audio average
```

### Passive Learning

```
1. During normal /analyze calls, speaker not identified
2. After ~20 seconds cumulative unidentified audio from same voice:
   - Server creates tentative voiceprint profile
   - Server creates PendingConfirmation (type: new_speaker)
3. Client prompts user to identify speaker
4. User selects existing member or creates new
5. Voiceprint linked to member, status="pending"
6. Next few identifications trigger verify_speaker confirmations
7. After 3-5 successful verifications, status="confirmed"
```

---

## Baseline Calibration

### During Enrollment

```
For each enrollment audio segment:
  Calculate average volume (dB)
  
On enrollment complete:
  baselineVolume = average of all segment volumes
  Store in member record
```

### Ongoing Refinement

```
For each /analyze call where:
  - Speaker identified with confidence >= 0.8
  - isYelling = false
  
Update baseline:
  newSample = measuredVolumeDb
  currentBaseline = member.baselineVolume
  sampleCount = member.baselineSampleCount
  
  updatedBaseline = (currentBaseline * sampleCount + newSample) / (sampleCount + 1)
  updatedCount = min(sampleCount + 1, 100)  // cap at 100 samples
  
  Store updatedBaseline and updatedCount
```

---

## Mood Keyword Lists

### Aggressive Indicators (weight: +0.2 each, max +0.6)
```
damn, dammit, hell, crap, cuss words (configurable list),
shut up, stop it, hate, stupid, idiot
```

### Escalation Phrases (weight: +0.3 each, max +0.6)
```
"i said", "i told you", "how many times",
"listen to me", "right now", "immediately"
```

### De-escalation Indicators (weight: -0.2 each)
```
"i'm sorry", "my bad", "okay okay",
"you're right", "let me", "can we"
```

### Excitement Indicators (mood=excited, not angry)
```
"yes!", "we won", "amazing", "awesome",
"oh my god" (positive context), "i can't believe"
```

**Note:** Keyword detection is supplementary. Volume and speaker baseline deviation are primary signals.

---

## Performance Targets

| Metric | Target |
|--------|--------|
| Local volume check latency | < 10ms per chunk |
| Cloud round-trip (analyze) | < 2 seconds |
| Alert display after yelling | < 3 seconds total |
| Voiceprint identification | < 1.5 seconds |
| Enrollment audio required | 20-30 seconds total |
| Baseline calibration accuracy | ±3 dB after 10 samples |

---

## Error Handling

| Scenario | Handling |
|----------|----------|
| Network timeout | Retry once, then discard |
| Azure Speech failure | Log error, return isYelling=false |
| Speaker Recognition failure | Continue without identification |
| Transcription failure | Use volume-only detection |
| Invalid audio format | Return 400, client logs error |
| Duplicate detected | Return isDuplicate=true, no alert |