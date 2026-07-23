# Splice

A lightweight, standalone, single-file HTML meeting recorder designed for absolute privacy and zero-friction deployment.

Splice automatically adapts to your device environment to capture high-quality audio from in-person conversations, browser-based video conferences, or desktop applications (including two-way phone calls routed via Microsoft Phone Link)—all without requiring software installation.

---

## Core Features

* **Zero-Dependency Single File:** Built entirely in a single `.html` file using vanilla JavaScript and inline CSS. No frameworks, build steps, or npm install required.
* **100% Local Recording:** Audio is captured in the browser and stays on your device until *you* choose to transcribe with your own Groq API key. Download always uses the original full-quality file.
* **Dual-Mode Environmental Detection:** On load, the app automatically determines its host architecture and configures the interface:
    * **Desktop Mode (Dual-Capture, Stereo Split):** Uses the Web Audio API to record your microphone (`getUserMedia`) on the **left** channel and system/tab playback (`getDisplayMedia`) on the **right**, so both sides of a call are captured *and* stay distinguishable. Both sides are audible in the download.
    * **Mobile Mode (Microphone Capture):** Falls back to a device-optimized microphone layout and requests a Screen Wake Lock for long field sessions.
* **Channel-Based Speaker Labels (desktop only):** The final pass compares per-channel energy to label each turn **Me** or **Them**. See [Speaker labeling](#speaker-labeling) — this is source separation, not voice identification.
* **Assign-Forward Labeling:** Transcript rows are grouped into turns with a single speaker chip. Click to cycle; an assignment applies forward until the next one, so naming a meeting takes a handful of clicks rather than one per line.
* **Hallucination Filtering:** Whisper reproduces its own biasing prompt as transcript text during silence. Splice filters those lines out using segment confidence metadata plus fuzzy prompt matching, and skips near-silent chunks entirely.
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
5. Lines are pre-labeled **Me** / **Them** from the audio channels. Correct any that are wrong (click a chip), rename speakers, fix wording, then download audio, download the transcript, or generate a summary / professional export.

> **Use headphones where you can.** Microphone echo cancellation is what keeps the far end out of your mic channel. On laptop speakers, meeting audio bleeds into the microphone and Me/Them accuracy drops.

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

## Speaker labeling

### What it is — and what it is not

Splice does **not** identify voices. There is no diarization and no speaker recognition. What it has on desktop is a *capture channel*, which is a different and much cheaper signal.

In desktop dual-capture, your microphone is recorded on the left channel and the meeting/system audio on the right. The final pass compares the energy of the two channels over each segment and labels it accordingly:

| Label | Means |
| --- | --- |
| **Me** | Came through this device's microphone |
| **Them** | Came through the meeting / system audio |

The practical consequence: **everyone sharing your microphone is "Me", and every remote participant is "Them."** On a two-party call — including Phone Link — that is exactly right. On an eight-person Teams call you get two labels for eight people, all of them true but coarse; use assign-forward labeling to split "Them" into real names.

Every export states which of these applies, so a transcript never implies more attribution than the audio supports.

### Where auto-labeling applies

| Capture | Auto labels |
| --- | --- |
| Desktop dual-capture (this version onward) | **Yes** — Me / Them |
| Mobile microphone mode | No — unlabeled, assign by hand |
| Recordings made before this version | No — the channels were summed, so the signal isn't there |
| Near-live drafts | No — drafts are never labeled |

Without auto labels, lines render as plain timestamped paragraphs. Unlabeled is deliberate: it is more truthful than a confident guess.

### Assign-forward labeling

Consecutive lines from the same speaker, less than 2.5 s apart, group into one **turn** sharing a timestamp and a speaker chip.

| Action | Effect |
| --- | --- |
| **Click a chip** | Cycles to the next speaker, then back to auto. Applies **from this turn forward** until the next assignment |
| **Shift-click** (desktop) | Assigns that turn only |
| **Long-press** (touch) | Same as shift-click |
| **Pencil in the speaker bar** | Renames a speaker everywhere |
| **+ Add speaker** | Creates a speaker to assign — use this on mobile and mono captures |

Chips shown in a lighter weight are automatic; solid ones you set yourself.

Assignments are stored as sparse anchors **keyed by timestamp, not by row number**, so re-running final transcription — which rebuilds segments with different boundaries — does not silently shift your labels onto other sentences. Anchors live in memory for the session and are not written to storage, matching how transcripts already behave (a reload starts fresh).

All exports — transcript download, Word, PDF, and the summary input — use the same turn grouping, naming each speaker once per turn.

---

## Transcript accuracy safeguards

Whisper is given a jargon-biasing prompt (SCADA, lift station, force main, RFI, submittal…) to improve recognition of field terminology. A known Whisper failure mode is reproducing that prompt *verbatim as transcript text* whenever a chunk contains little or no speech — in a real 26-minute meeting it appeared roughly once per silent stretch, and flowed straight into summaries and exports.

Three independent defenses, each of which catches the leak on its own:

1. **Segment confidence metadata** — `no_speech_prob`, `avg_logprob`, and `compression_ratio` from `verbose_json`, using Whisper's own decoding thresholds.
2. **Fuzzy prompt matching** — bigram containment plus normalized Levenshtein against the prompt. Catches partial leaks like `force main, manhole, RFI, submittal` that exact matching misses, while leaving real speech that merely *uses* the jargon untouched.
3. **Silence gate** — chunks whose loudest 1-second frame never clears an RMS floor are never uploaded at all, which saves the API call and removes the trigger rather than filtering the symptom.

Dropped lines are logged to the browser console with the reason, and the status line reports counts (e.g. *"2 silent chunks skipped, 3 hallucinated lines filtered"*).

### Tuning thresholds

Defaults are conservative. All are adjustable live from the browser console via `__splice`:

| Constant | Default | Raise it if… |
| --- | --- | --- |
| `TX_FILTER.PROMPT_SIMILARITY_MAX` | `0.80` | Real speech is being dropped (measured separation is wide: leaks score 1.00, genuine speech 0.00–0.32) |
| `TX_FILTER.SILENT_FRAME_RMS` | `0.005` | Silent chunks still slip through. **Lower** it if a quiet talker gets skipped — this is the most likely one to need tuning |
| `TX_FILTER.NO_SPEECH_PROB_HARD` | `0.90` | Real speech in noisy rooms is dropped |
| `DIARIZE.MARGIN_RATIO` | `1.3` | Labels flip back and forth mid-turn. **Lower** it if they stick to one speaker too long |

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

* **Audio Pipeline:** Desktop routes mic and system audio through a `ChannelMergerNode` (mic → channel 0, system → channel 1) into `AudioContext.createMediaStreamDestination()`. Mobile stays mono. Merger inputs are single-channel per spec, so a stereo tab source is downmixed into its channel.
* **Encoding Rate:** 1-second timeslice (`MediaRecorder.start(1000)`) for stable memory on long sessions.
* **Final STT prep:** Web Audio decode → mono mix → 16 kHz resample (`OfflineAudioContext`) → pure-JS 16-bit PCM WAV (no ffmpeg.wasm). Chunk windows use a configurable overlap; segments are stitched by absolute timestamps.
* **Channel envelopes:** Per-channel RMS at a 50 ms hop is captured while the decoded buffer still has both channels, then the samples are released. Holding per-channel data for a 2-hour meeting would be ~2.8 GB; both envelopes together are ~1.2 MB.
* **Thresholds (tunable in code):** `FINAL_TX` — safe upload ~20 MB, single-request duration ~18 min, default chunk 10 min, overlap 8 s. `TX_FILTER` — hallucination and silence thresholds. `DIARIZE` — envelope hop and channel margin.
* **Speaker model:** Sparse timestamp-keyed anchors layered over channel labels; effective speaker resolves as single anchor → forward anchor → auto label → unassigned. Turns break on speaker change, on a gap ≥ 2.5 s, or on a change of channel label.
* **UI:** CSS variables with dark theme; Tabler outline icons as an inline SVG sprite. Transcript lines are `contenteditable` with plain-text paste enforcement.
* **Debug surface:** `window.__splice` exposes thresholds, pure helpers, and a `loadSample()` for exercising the transcript UI without recording. It never touches audio, settings, or the API key.

---

## Changelog

### 1.1.0 — Honest speaker labels and hallucination filtering

**Fixed — prompt leakage into transcripts.** The jargon-biasing prompt was appearing verbatim as transcript lines during silent stretches and flowing into summaries and exports. Now filtered by segment confidence metadata, fuzzy prompt matching, and a pre-upload silence gate. See [Transcript accuracy safeguards](#transcript-accuracy-safeguards).

**Fixed — fabricated speaker labels.** Labels were produced by alternating "Speaker 1"/"Speaker 2" on any silence gap over 1.8 s. That is pause timing, not voices, so an eight-person meeting rendered as two speakers and lines were attributed to people who never said them. The heuristic is removed.

**New — channel-based labeling (desktop).** Desktop dual-capture now records mic and system audio on separate channels instead of summing them, and the final pass labels turns **Me** / **Them** from per-channel energy. Mobile, pre-1.1.0 recordings, and drafts get no automatic labels and render unlabeled rather than guessing.

**New — assign-forward labeling.** Per-segment cards with a redundant label and a dropdown are replaced by turn-grouped rows with a single chip. An assignment applies forward until the next one; shift-click or long-press assigns one turn. Anchors are timestamp-keyed so re-running final transcription cannot shift labels onto other sentences.

**Changed — export headers** now carry the app version, the ASR model, and a statement of where speaker labels came from, instead of `Version: final`.

**Compatibility:** no storage format changes; existing settings and API keys are untouched. Recordings made before 1.1.0 still transcribe normally, just without automatic labels.

### Long-audio final transcription
* Pre-process final-pass audio to mono 16 kHz (user-controllable, default On).
* Automatic overlapping chunking when size or duration exceeds safe thresholds.
* Progress messaging for optimize + per-chunk final transcription.
* Settings for compress, chunk length, and force-chunk; original download remains full quality.
* Prevents Groq `request_too_large` / “Request Entity Too Large” on long field meetings.
