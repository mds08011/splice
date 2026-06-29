# Prompt: Rebuild Meeting Audio Recorder

---

Please build a standalone single-file HTML meeting audio recorder and provide it as a downloadable file. Here are the full requirements:

---

## Purpose

I am a project engineer and attend a lot of Zoom and Microsoft Teams meetings on a work computer where I cannot install software. I also attend in-person meetings. I need a tool to record audio from both types of meetings so I can refer back to them when writing notes afterward.

---

## Core behavior: dual mode

The tool must auto-detect which mode to use based on the browser environment — no manual toggle needed:

**Desktop mode (Chrome/Edge on a computer)**
- Uses the browser's `getDisplayMedia` API to capture audio from another browser tab
- The intended workflow: user opens the recorder in one tab, joins Teams or Zoom in another tab, clicks Start, selects the meeting tab in Chrome's share dialog, confirms "Share tab audio" is checked, clicks Share
- The MediaRecorder should receive an audio-only MediaStream (extract audio tracks from the getDisplayMedia stream; pass the full stream — including the video track — as the "capture stream" for cleanup and track-ended detection, but only record the audio portion)
- Must detect if the user didn't check "Share tab audio" (i.e., no audio tracks were returned) and show a helpful error
- Should also work if the user picks "Window" or "Entire screen" instead of a tab (for desktop app users)

**Mobile mode (phones and tablets where getDisplayMedia is unavailable)**
- Detects the absence of `navigator.mediaDevices.getDisplayMedia` and switches to microphone recording via `getUserMedia({ audio: true })`
- Requests microphone permission and records from the device mic
- Should attempt to acquire a Wake Lock (`navigator.wakeLock.request('screen')`) to prevent the screen from sleeping during recording — fail silently if unavailable
- Release the wake lock when recording stops or is reset

---

## File format and MIME type

- Detect the best supported MIME type in this order: `audio/webm;codecs=opus`, `audio/webm`, `audio/ogg;codecs=opus`, `audio/mp4`
- Use `MediaRecorder.isTypeSupported()` for detection; fall back to browser default if none match
- Derive the file extension from the actual MIME type used: `.webm` for webm, `.ogg` for ogg, `.m4a` for mp4
- Downloaded files should be named with a timestamp: `meeting-YYYY-MM-DDTHH-MM-SS.ext`

---

## UI states and flow

**Idle state**
- Shows a microphone icon, "Ready to record" title, subtitle "Audio stays on your device — never uploaded"
- Shows the appropriate how-to instructions panel (desktop or mobile version)
- Single "Start recording" button

**Recording state**
- Pulsing/blinking animation on the status icon
- "Recording" title with a live timer (MM:SS, or H:MM:SS for recordings over an hour) that updates every second
- How-to instructions hidden
- Single "Stop recording" button styled in the danger/red color

**Complete state**
- Status shows "Recording complete" with total duration
- Audio player (`<audio controls>`) rendered inline so the user can preview before downloading
- Note below the player about the file format and how to convert if needed (e.g., VLC for MP3 conversion)
- "Download" button and a smaller "New" button (fixed width, not flex-grow)
- How-to instructions remain hidden

**Error handling**
- If the share dialog is cancelled or denied (NotAllowedError / AbortError), fail silently with no error shown
- All other errors show a dismissible error panel with a plain-English description
- On no-audio-track error: tell the user to try again and check "Share tab audio"
- On mic denied: tell the user to tap Allow when prompted

---

## Instructions panels

**Desktop panel content:**
1. Join Teams or Zoom in another browser tab
2. Click Start recording below
3. In Chrome's share dialog, select that tab
4. Confirm "Share tab audio" is checked, then click Share

Note: "Using the desktop app? Choose Window or Entire screen and enable system audio instead."

**Mobile panel content:**
1. Allow microphone access when prompted
2. Click Start recording and keep this screen open
3. Place the phone centrally if others are present
4. Tap Stop recording when done, then download the file

Note: "Keep your screen on during recording — locking the phone may pause capture on some browsers."

---

## Header / branding

- Small header above the status card showing "Meeting recorder" in muted uppercase text with a small colored dot accent
- A "mode pill" badge in the top-right of the header showing either "Tab capture" (desktop) or "Microphone" (mobile), with a small device icon
- The pill updates automatically on page load based on detected capability

---

## Design requirements

- Standalone single HTML file — no build tools, no frameworks, no npm
- All CSS and JavaScript inline in the file
- Icons via the Tabler Icons webfont CDN: `https://cdn.jsdelivr.net/npm/@tabler/icons-webfont@3/dist/tabler-icons.min.css`
- Uses CSS custom properties (variables) for all colors — never hardcode hex values in component styles
- Full dark mode support via `@media (prefers-color-scheme: dark)` on the `:root` variable definitions
- Clean flat design — no gradients, no drop shadows, no blur effects
- Responsive and usable on both desktop and mobile screen sizes
- `-webkit-tap-highlight-color: transparent` on buttons for clean mobile tap behavior
- Max content width of 440px, centered on the page

**Color palette (define as CSS variables with dark mode overrides):**

Light mode:
- `--bg`: #ffffff (page/card background)
- `--bg2`: #f5f5f4 (secondary surface, status card)
- `--tx`: #1c1917 (primary text)
- `--tx2`: #78716c (secondary text)
- `--tx3`: #a8a29e (tertiary/hint text)
- `--bd`: rgba(0,0,0,.12) (subtle border)
- `--bd2`: rgba(0,0,0,.22) (button border)
- `--red` / `--red-bg` / `--red-bd`: #dc2626 / #fef2f2 / #fecaca
- `--grn`: #16a34a
- `--blu` / `--blu-bg` / `--blu-bd`: #1d4ed8 / #eff6ff / #bfdbfe
- Border radius: `--r: 12px` (cards/panels), `--rs: 8px` (buttons/errors)

Dark mode overrides:
- `--bg`: #1c1917, `--bg2`: #292524, `--tx`: #fafaf9, `--tx2`: #a8a29e, `--tx3`: #57534e
- Borders: rgba(255,255,255,.1) and .22
- `--red`: #f87171, `--red-bg`: rgba(220,38,38,.12), `--red-bd`: rgba(220,38,38,.35)
- `--grn`: #4ade80, `--blu`: #60a5fa, `--blu-bg`: rgba(29,78,216,.12), `--blu-bd`: rgba(29,78,216,.35)

**Tabler icons used:**
- `ti-microphone` — idle state icon
- `ti-circle-dot` — recording state icon (with pulse animation)
- `ti-circle-check` — complete state icon
- `ti-circle` — on the Start recording button
- `ti-player-stop` — on the Stop recording button
- `ti-download` — on the Download button
- `ti-refresh` — on the New button
- `ti-device-desktop` — in the mode pill (desktop)
- `ti-device-mobile` — in the mode pill (mobile)

---

## JavaScript architecture

- Wrap everything in an IIFE to avoid polluting the global scope, except for the event handler functions which must be on `window` (e.g., `window._rs`, `window._rp`, `window._rd`, `window._ri`) since they're called via inline `onclick` attributes
- Use `const`/`let`, arrow functions, `async/await` — target modern Chrome/Edge/Safari only, no IE compatibility needed
- Two separate async paths: `startDesktop()` (getDisplayMedia) and `startMobile()` (getUserMedia), both calling a shared `setupRec(audioStream)` function
- Keep `captSt` (the full capture stream for cleanup) as a module-level variable separate from the audio-only stream passed to MediaRecorder
- Attach `track.onended` handlers on all tracks of `captSt` so that if the user stops sharing from the Chrome toolbar, recording stops automatically
- Use `rec.start(1000)` so data is collected in 1-second chunks
- On `rec.onstop`: create Blob, revoke any previous object URL, create new one, populate the audio player, update UI

---

## Footer

Small footer below the main content, separated by a thin border:

"Desktop: captures tab audio · Mobile: captures microphone  
All recordings stay on your device · Icons require internet to display"

---

## Hosting note (include as a comment in the file or a note in your response)

This file can be:
- Opened directly from a local drive in Chrome or Edge (file:// works fine for these APIs)
- Hosted on GitHub Pages by uploading as `index.html` to a public repository and enabling Pages in Settings

---

Please produce this as a downloadable `.html` file.
