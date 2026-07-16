# Splice

A lightweight, standalone, single-file HTML meeting recorder designed for absolute privacy and zero-friction deployment.

Splice automatically adapts to your device environment to capture high-quality audio from in-person conversations, browser-based video conferences, or desktop applications (including two-way phone calls routed via Microsoft Phone Link)—all without requiring software installation.

---

## Core Features

* **Zero-Dependency Single File:** Built entirely in a single `.html` file using vanilla JavaScript and inline CSS. No frameworks, build steps, or npm install required.
* **100% Local Recording:** Audio is captured in the browser and stays on your device until *you* choose to transcribe with your own Groq API key. Download always uses the original full-quality file.
* **Dual-Mode Environmental Detection:** On load, the app automatically determines its host architecture and configures the interface:
    * **Desktop Mode (Dual-Capture Mixer):** Uses the Web Audio API to mix your microphone (`getUserMedia`) with system/tab playback (`getDisplayMedia`) so both sides of a call are recorded.
    * **Mobile Mode (Microphone Capture):** Falls back to a device-optimized microphone layout and requests a Screen Wake Lock for long field sessions.
* **Hybrid Transcription:** Optional near-live draft updates during recording; on stop, a high-accuracy final pass always runs and becomes the source of truth for edits, summaries, and exports.
* **Long Meeting Support:** Final transcription automatically optimizes and splits long recordings so uploads stay under Groq’s file size limit (no more “Request Entity Too Large” on 30–90+ minute meetings).
* **Smart Container Targeting:** Selects the best available `MediaRecorder` type (`audio/webm;codecs=opus`, `.webm`, `.ogg`, or AAC/`.m4a`).
* **Resilient Failure Boundaries:** Share cancellations or “Stop Sharing” still preserve buffered audio and wrap up safely.

---

## How to Use

### Desktop Workflow (Zoom, Teams, & Microsoft Phone Link Calls)
Splice is highly optimized to run on restricted enterprise hardware where third-party executables are blocked by IT policies.

1. Open `index.html` in Chrome or Edge.
2. Open **Settings** and paste your Groq API key (stored only in this browser).
3. Click **Start Recording**. Grant microphone access, then share the meeting tab or entire screen with **Share tab/system audio** enabled.
4. When the meeting wraps up, click **Stop Recording**. Splice runs the final high-accuracy transcript automatically.
5. Edit speaker names or wording if needed, then download audio, download the transcript, or generate a summary / professional export.

### Mobile Workflow (In-Person Meetings)
1. Open the file or hosted link on an iOS or Android device.
2. Tap **Start Recording** and allow microphone access.
3. Place the device centrally. Splice will request a Wake Lock to try and keep the display alive.
4. Tap **Stop Recording**, wait for the final transcript, then download files as needed.

---

## Long Meetings (30–90+ minutes)

Groq’s speech API rejects direct uploads larger than **25 MB** (`request_too_large`). Dual-capture WebM files from long meetings often exceed that if sent as one blob.

**What Splice does on the final pass only** (draft near-live is separate):

1. **Optimize (default On)** — Builds a temporary mono **16 kHz** copy (Whisper’s preferred format). The original recording is **not** changed; **Download audio** is still full quality.
2. **Auto-chunk when needed** — If the optimized file is still over ~20 MB **or** the meeting is longer than ~18 minutes (or you force chunking), Splice splits audio into overlapping segments (default **10 minutes**, **8 s** overlap).
3. **Transcribe & stitch** — Each chunk is sent with the high-accuracy model; results are merged with corrected timestamps into one continuous, editable final transcript.
4. **Progress** — Status line shows messages such as:
   - `Optimizing audio…`
   - `Processing final transcript… chunk 2 of 4`
5. **Short meetings** — When size and duration are under the thresholds, a single request is used (fast path).

Typical expectations:

| Meeting length | Behavior (defaults) |
| --- | --- |
| Under ~10–15 min | Optimize → usually one API request |
| ~15–40 min | Optimize → a few chunks (e.g. 2–4) |
| 60–90+ min | Optimize → more chunks; allow several minutes for the full final pass |

If a single chunk fails (rate limit / network), Splice continues with the rest and warns which chunk was skipped so you can re-run final transcription.

### Related Settings

| Setting | Default | Purpose |
| --- | --- | --- |
| **Compress audio for final transcription** | On | Mono 16 kHz re-encode for the API only |
| **Chunk length for long recordings** | 10 min | Target segment length (5 / 8 / 10 / 12 / 15). May be shortened slightly so each upload stays under the size limit |
| **Always chunk final transcription** | Off | Testing: force multi-part final even on short meetings |
| **Draft update frequency** | Off | Optional near-live drafts during recording (does not replace the final pass) |

Preferences are stored in `localStorage` in this browser only.

---

## Local Execution

Splice relies natively on Web Media APIs. Because it does not use cross-origin modules for core capture, it can run from offline local storage.

### Offline Local Storage
Save `index.html` to a drive or thumb drive and open it via `file://`. Capture works in modern browsers. Transcription and summarization require network access to Groq with your API key.

**Note:** Icons are vendored inline (no icon CDN). The optional markdown helper script is loaded from a CDN when online for summary rendering.

---

## Versioning and Releases

This repository uses GitHub Actions to automatically generate a tagged release containing the standalone HTML file for easy downloading.

To publish a new version:
1. Commit your changes to the `main` branch.
2. Create a semantic version tag in git (e.g., `git tag v1.0.0`).
3. Push the tag to the repository (`git push origin v1.0.0`).

The automated pipeline will immediately draft a GitHub Release and attach the latest `index.html` file as a direct download.

---

## Technical Specifications

* **Audio Pipeline:** `MediaStreamAudioSourceNode` mixed via `AudioContext.createMediaStreamDestination()`.
* **Encoding Rate:** 1-second timeslice (`MediaRecorder.start(1000)`) for stable memory on long sessions.
* **Final STT prep:** Web Audio decode → mono mix → 16 kHz resample (`OfflineAudioContext`) → pure-JS 16-bit PCM WAV (no ffmpeg.wasm). Chunk windows use a configurable overlap; segments are stitched by absolute timestamps.
* **Thresholds (tunable in code via `FINAL_TX`):** safe upload size ~20 MB, single-request duration ~18 min, default chunk 10 min, overlap 8 s.
* **UI:** CSS variables with dark theme; Tabler outline icons as an inline SVG sprite.

---

## Changelog

### Long-audio final transcription
* Pre-process final-pass audio to mono 16 kHz (user-controllable, default On).
* Automatic overlapping chunking when size or duration exceeds safe thresholds.
* Progress messaging for optimize + per-chunk final transcription.
* Settings for compress, chunk length, and force-chunk; original download remains full quality.
* Prevents Groq `request_too_large` / “Request Entity Too Large” on long field meetings.
