# Voice-to-Semantic-Search Pipeline



https://github.com/user-attachments/assets/75271c9c-5d25-418b-81fc-983eb89e8920



> **Current implementation** as of commit `b5822d7` and subsequent fixes.  
> The earlier `azure-speech-voice-input.md` describes the original Capacitor `@capacitor-community/speech-recognition` approach, which was replaced by this pipeline.

---

## Overview

Users can speak a search query into the receipt filter modal. The spoken audio is recorded in the browser, converted to WAV, uploaded to the backend, transcribed by Azure Cognitive Services Speech REST (with Azure OpenAI Whisper as the primary option when deployed), and the resulting text is used as the semantic query against the receipt vector store.

```
[Mic tap] → MediaRecorder (WebM/MP4) → WAV conversion → POST /api/speech/transcribe
         → Azure Speech REST → transcript → semantic embedding → ranked receipts
```

---

## Frontend Pipeline

### 1. AudioRecordingService (`core/services/audio-recording.service.ts`)

Low-level wrapper around the browser `MediaRecorder` API.

| Signal | Type | Description |
|---|---|---|
| `isRecording` | `Signal<boolean>` | True while mic is active |
| `isFinished` | `Signal<boolean>` | Fires true when MediaRecorder stops |
| `recordingTime` | `Signal<number>` | Elapsed seconds (0–10) |
| `amplitude` | `Signal<number>` | RMS amplitude 0–1 (drives waveform UI) |

**Key behaviours:**
- `startRecording(maxDuration = 5)` — requests `getUserMedia({ audio: true })`, starts `MediaRecorder`, starts a 1-second interval timer that auto-stops at `maxDuration`
- `VoiceTranscriptionService` passes `maxDuration: 10`
- `onstop` callback assembles `audioChunks[]` into `new Blob(chunks, { type: 'audio/webm' })`
- `cancelRecording()` nulls `onstop` before calling `.stop()` so `isFinished` never fires on cancel — preventing stale transcriptions
- The `amplitude` signal is fed by a `requestAnimationFrame` loop reading from an `AnalyserNode`

### 2. VoiceTranscriptionService (`core/services/voice-transcription.service.ts`)

Orchestrates the full record → convert → upload → display flow.

**State machine:**

```
idle  →  (startRecording)  →  recording  →  (stop/auto-stop)  →  transcribing  →  hasRecording
                                                 ↓ cancel
                                               idle
```

**Signals:**

| Signal | Type | Description |
|---|---|---|
| `isRecording` | proxied from `AudioRecordingService` | |
| `recordingTime` | proxied from `AudioRecordingService` | |
| `isTranscribing` | `Signal<boolean>` | True during WAV conversion + HTTP POST |
| `transcript` | `Signal<string>` | Latest transcribed text |
| `audioUrl` | `Signal<string \| null>` | Object URL for playback (WebM/MP4 original) |
| `error` | `Signal<string \| null>` | Transcription or mic error message |

**Session ID pattern:**  
`_sessionId` is incremented on every `startRecording()` and `cancelRecording()`. Each async step checks `this._sessionId !== sessionAtStart` before writing any signal. This prevents a cancelled recording's transcript from leaking into the query field if the HTTP response arrives late.

**WAV conversion (`_toWav`):**

Azure Cognitive Services Speech REST does not accept WebM or MP4 containers via the short-audio REST endpoint. The browser `MediaRecorder` produces `audio/webm;codecs=opus` on Android and `audio/mp4` on iOS. Conversion flow:

```
WebM/MP4 Blob
  → blob.arrayBuffer()
  → AudioContext.decodeAudioData()          // decode to floating-point PCM
  → OfflineAudioContext(1, length, 16000)   // resample: 1 channel, 16 kHz
  → offlineCtx.startRendering()             // render resampled buffer
  → _encodeWav()                            // write RIFF header + 16-bit PCM samples
  → Blob({ type: 'audio/wav' })
```

`_encodeWav` writes:
- 44-byte RIFF/WAVE/fmt/data header
- 16-bit little-endian PCM samples, mono, 16 kHz
- Clamps float samples from `[-1, 1]` to `[-32768, 32767]`

The **original blob** (WebM/MP4) is kept as `audioUrl` for the in-modal audio player — only the WAV copy is uploaded. If Web Audio decoding fails (e.g. extremely short clip), the raw WebM blob is uploaded as a fallback.

**`_uploadAndTranscribe` flow:**
1. Call `audioService.getBlobAndReset()` — stops hardware if still running, returns finalized blob
2. Create Object URL from raw blob → set `audioUrl` (player appears immediately)
3. Set `isTranscribing = true`
4. Try `_toWav(blob)` → `uploadBlob = recording.wav`; on failure → `uploadBlob = recording.webm`
5. `POST /api/speech/transcribe` with `FormData { audio: uploadBlob }`
6. On success with non-empty `result.text`: set `transcript`
7. On success with empty `result.text`: revoke `audioUrl`, set `error = 'No speech detected…'`
8. On HTTP error: set `error` from response body or message
9. `finally`: set `isTranscribing = false` (session-guarded)

**Public methods:**

| Method | Description |
|---|---|
| `startRecording()` | Resets state, increments sessionId, starts AudioRecordingService |
| `stopEarly()` | Calls `audioService.stopRecordingHardware()` — triggers `isFinished` → upload |
| `cancelRecording()` | Increments sessionId, cancels AudioRecordingService, revokes URL, clears transcript |
| `clearTranscript()` | Revokes URL, clears transcript + error |
| `clearError()` | Sets `error = null` (called from filter when user starts typing) |

### 3. Filter Components

Both `ReceiptFilterComponent` (personal) and `BusinessReceiptFilterComponent` (business) wire up the voice service identically.

**Template UI states:**

| State | Shown |
|---|---|
| `!isRecording() && !audioUrl()` | Default textarea + mic button |
| `isRecording()` | Recording header (timer, waveform, cancel/stop buttons) |
| `isTranscribing()` | "Transcribing…" spinner |
| `audioUrl()` | Player row: `<audio>` + send button + re-record button |
| `voiceService.error()` | Red error text below textarea |

**Send button guard:** `[disabled]="isTranscribing()"` — prevents submitting before the HTTP response returns.

**Error auto-clear:** `setQuery(text)` calls `voiceService.clearError()` whenever the user types, so a stale "No speech detected" message disappears immediately on input.

**Reset behaviour:** `reset()` calls `modalCtrl.dismiss(resetFilters, 'confirm')`. The parent `onDidDismiss` handler only calls `loadData(true)` when `role === 'confirm'`, so passing `'confirm'` here causes an immediate reload with cleared filters — no second tap required.

---

## Backend Pipeline

### API Endpoint: `POST /api/speech/transcribe`

**Controller:** `SpeechController` (`Api/Controllers/SpeechController.cs`)

```
POST /api/speech/transcribe
Authorization: Bearer <JWT>
Content-Type: multipart/form-data
Body: audio=<file>        (max 10 MB)
```

Response:
```json
{ "text": "Find Rose Pharmacy" }
```

- Empty `text` (`""`) = speech detected nothing (not an error)
- `503` = transcription service unavailable (both providers failed)
- `400` = no file provided

**MediatR flow:** `SpeechController` → `TranscribeAudioCommand` → `TranscribeAudioCommandHandler` → `IWhisperTranscriptionService`

### WhisperTranscriptionService

**Location:** `Infrastructure/Services/WhisperTranscriptionService.cs`

Implements `IWhisperTranscriptionService`. Two-provider waterfall:

```
Provider 1: Azure OpenAI Whisper
  → only attempted when WhisperDeploymentName is non-empty in config
  → on success: return transcript
  → on exception: log error, fall through

Provider 2: Azure Cognitive Services Speech REST
  → POST https://{region}.stt.speech.microsoft.com/speech/recognition/conversation/cognitiveservices/v1
  → Header: Ocp-Apim-Subscription-Key: {SubscriptionKey}
  → Query: ?language=en-US&format=simple
  → Returns: { "RecognitionStatus": "Success", "DisplayText": "..." }
  → status == "Success" + empty DisplayText → return "" (no speech, not an error)
  → non-success status → return null (failure)
```

**Stream buffering pattern:**  
The Azure OpenAI SDK disposes the `Stream` it receives after `TranscribeAudioAsync`. To prevent the disposed stream from corrupting the fallback provider:

```csharp
// Read once to bytes
byte[] audioBytes;
using (var staging = new MemoryStream()) {
    await audioStream.CopyToAsync(staging, cancellationToken);
    audioBytes = staging.ToArray();
}

// Independent stream per provider
using var whisperStream = new MemoryStream(audioBytes);   // Whisper (disposable by SDK)
using var speechStream  = new MemoryStream(audioBytes);   // Azure Speech (always fresh)
```

**Content-type mapping:**

| Extension | Content-Type |
|---|---|
| `.wav` | `audio/wav` |
| `.webm` | `audio/webm;codecs=opus` |
| `.mp4` / `.m4a` | `audio/mp4` |
| `.mp3` / `.mpeg` | `audio/mpeg` |
| `.ogg` | `audio/ogg;codecs=opus` |

### Token Endpoint: `GET /api/speech/token`

Returns a 10-minute Azure Speech JWT token and the configured region. Used by clients that wish to call Azure Speech SDK directly. Currently unused by the mobile app (which uses the server-side transcription path instead).

---

## Semantic Search Pipeline

After transcription, the filter is applied as a `semanticQuery` parameter on `GET /api/personal-receipts` or `GET /api/business-receipts`.

### SemanticQueryParser

Intercepts numeric range intent before embedding is called.

| Query | MinAmount | MaxAmount | RemainingQuery |
|---|---|---|---|
| `"amount less than 1000"` | null | 1000 | `""` (pure numeric) |
| `"coffee less than 500"` | null | 500 | `"coffee"` (hybrid) |
| `"Find Rose Pharmacy"` | null | null | `"Find Rose Pharmacy"` (text only) |

Three execution paths:
- **Pure numeric** — EF `WHERE` clause on `TotalAmount` only; no embedding call
- **Hybrid** — EF `WHERE` on amount + embedding on remaining keyword
- **Text only** — standard semantic path

### Noise-word stripping

Before matching, these words are removed: `find`, `get`, `show`, `me`, `items`, `item`, `receipt`, `receipts`.

Trailing punctuation is stripped: `.`, `,`, `!`, `?`, `;`, `:` — handles Azure Speech appending `.` to transcripts (e.g. `"Find Rose Pharmacy."` → `"rose pharmacy"`).

### Embedding

`IAiEmbeddingService.GetEmbeddingAsync(query)` — calls Azure OpenAI `text-embedding-3-large` (1536 dimensions) or falls back to native BM25 if `EmbeddingsApiUrl` is empty or the container app is unavailable.

### Hybrid scoring (merchant-first)

Cosine distance (`pgvector`) is combined with BM25 for the final score, but merchant name matching takes precedence via explicit tiers:

| Tier | Score | Condition |
|---|---|---|
| MerchantExact | 100% | `merchant.Contains(cleanedQuery)` |
| MerchantWords | 98% | every query word found in merchant name |
| MerchantReverse | 98% | merchant is substring of query (e.g. "Shell" in "Shell station") |
| ItemExact | 95% | any item description contains full query |
| CategoryExact | 90% | query equals (or single-word matches) category name |
| Stemmed | 85% | stemmed/plural match on merchant, item, or category |
| Hybrid | 0–80% | `(cosine × 0.7) + (BM25 × 0.3)`, capped at 80 |

**60% threshold:** Results below 60% are excluded entirely.

**Why `cleanedSearchTerm.Contains(catLower)` was removed:**  
The old check fired true whenever the search query _contained_ the category word (e.g. `"rose pharmacy".Contains("pharmacy") == true`), giving 100% match score to every receipt in the PHARMACY category regardless of merchant name. Category is now only a signal when the query **is** the category, not when the category is a substring of the query.

---

## Configuration Reference

### `appsettings.json` / `appsettings.Development.json`

```json
{
  "AzureOpenAI": {
    "Endpoint": "https://<resource>.openai.azure.com/",
    "ApiKey": "<key>",
    "DeploymentName": "text-embedding-3-large",
    "EmbeddingDimensions": 1536,
    "WhisperDeploymentName": ""
  },
  "AzureSpeech": {
    "SubscriptionKey": "<key>",
    "Region": "southeastasia"
  },
  "AiSettings": {
    "EmbeddingsApiUrl": ""
  }
}
```

| Key | Effect when empty |
|---|---|
| `WhisperDeploymentName` | Whisper disabled; Azure Speech REST used directly |
| `AzureSpeech.SubscriptionKey` | Both providers fail; `POST /api/speech/transcribe` returns 503 |
| `AiSettings.EmbeddingsApiUrl` | Skips HTTP embedding container; uses native BM25 directly |

---

## Known Behaviours & Edge Cases

| Scenario | Behaviour |
|---|---|
| Recording cancelled mid-flight | Session ID mismatch prevents transcript from writing to UI |
| Web Audio decoding fails (very short clip) | Falls back to uploading raw WebM blob |
| Azure Speech returns `RecognitionStatus: InitialSilenceTimeout` | `null` returned → 503 to client → "Transcription failed" error |
| Azure Speech returns `RecognitionStatus: Success` with empty text | `""` returned → client shows "No speech detected" |
| User types after "No speech detected" error | `clearError()` called on first keystroke — error vanishes immediately |
| Send button tapped while transcribing | Button is `[disabled]` while `isTranscribing()` is true |
| Reset tapped in filter modal | Dismisses with `role='confirm'` + reset filters → parent immediately reloads receipts |
