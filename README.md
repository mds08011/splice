# Splice

A lightweight, standalone, single-file HTML meeting recorder designed for absolute privacy and zero-friction deployment. 

Splice automatically adapts to your device environment to capture high-quality audio from in-person conversations, browser-based video conferences, or desktop applications (including two-way phone calls routed via Microsoft Phone Link)—all without requiring software installation or network connectivity.

---

## Core Features

* **Zero-Dependency Single File:** Built entirely in a single `.html` file using vanilla JavaScript and inline CSS. No frameworks, build steps, or npm install required.
* **100% Local & Private:** Processing occurs strictly within the local browser runtime. Audio data is captured via memory buffers and exported directly to a local disk blob. It is never transmitted across a network.
* **Dual-Mode Environmental Detection:** On load, the app automatically determines its host architecture and configures the interface:
    * **Desktop Mode (Dual-Capture Mixer):** Uses the Web Audio API to instantiate a hardware-level mixing node, binding your system microphone input (`getUserMedia`) with your system playback audio (`getDisplayMedia`). This forces the recording of both sides of a call simultaneously.
    * **Mobile Mode (Microphone Capture):** Automatically falls back to a device-optimized microphone recording layout and silently attempts to acquire a Screen Wake Lock to prevent mobile browsers from entering sleep states during long sessions.
* **Smart Container Targeting:** Programmatically queries `MediaRecorder.isTypeSupported()` to select the optimal compression codec available on the host browser, prioritizing `audio/webm;codecs=opus`, falling back to standard `.webm`, `.ogg`, or native Apple AAC/`.m4a` containers.
* **Resilient Failure Boundaries:** Gracefully catches share cancellations or intentional stream termination (such as hitting "Stop Sharing" on the native browser banner) to preserve buffer chunks and safely wrap up the recording.

---

## How to Use

### Desktop Workflow (Zoom, Teams, & Microsoft Phone Link Calls)
Splice is highly optimized to run on restricted enterprise hardware where third-party executables are blocked by IT policies. 

1. Open `splice.html` in Chrome or Edge.
2. Click **Start Recording**. When prompted by the browser, grant permission to access your local microphone.
3. The browser sharing dialog will appear. Your selection dictates what audio is merged:
    * **For Browser Meetings (Zoom/Teams Tabs):** Select the **Chrome/Edge Tab** option, pick your meeting tab, and ensure the "Share tab audio" checkbox is selected.
    * **For Desktop Apps & Phone Link Calls:** Select the **Entire Screen** option and check the "Share system audio" toggle. This captures the incoming Bluetooth audio stream from your phone while the microphone captures your own voice.
4. Click **Share** to begin recording.
5. When the meeting wraps up, click **Stop Recording** inside Splice. Preview the file instantly using the inline player, or click **Download** to save a timestamped file (`meeting-YYYY-MM-DDTHH-MM-SS.ext`).

### Mobile Workflow (In-Person Meetings)
1. Open the file or hosted link on an iOS or Android device.
2. Tap **Start Recording** and allow microphone access.
3. Place the device centrally. Splice will request a Wake Lock to try and keep the display alive.
4. Tap **Stop Recording** and trigger the download to save the file locally to your device's file system.

---

## Local Execution & Deployment

Splice relies natively on Web Media APIs. Because it does not use cross-origin modules, it can be executed using two methods:

### 1. Offline Local Storage
Simply save the file as `splice.html` onto your local drive or an external thumb drive. Double-clicking the file to run it via the local file system (`file://splice.html`) works perfectly in standard modern browsers for all capture types.

### 2. GitHub Pages Static Hosting
To maintain instant access across both your workstation and mobile devices without manually copying files, host it securely for free using GitHub Pages:
1. Push the file to a public or private GitHub repository, naming the file `index.html`.
2. Navigate to repository **Settings** -> **Pages**.
3. Under **Build and deployment**, set the source to **Deploy from a branch** and select `main` (or `master`).
4. Click **Save**. Your instance will be live at `https://<your-username>.github.io/<repo-name>/` over a secure HTTPS connection.

---

## Versioning and Releases

This repository uses GitHub Actions to automatically generate a tagged release containing the standalone HTML file for easy downloading.

To publish a new version:
1. Commit your changes to the `main` branch.
2. Create a semantic version tag in git (e.g., `git tag v1.0.0`).
3. Push the tag to the repository (`git push origin v1.0.0`).

The automated pipeline will immediately draft a GitHub Release and attach the latest `splice.html` file as a direct download.

---

## Technical Specifications

* **Audio Pipeline:** `MediaStreamAudioSourceNode` combined via `AudioContext.createMediaStreamDestination()`.
* **Encoding Rate:** 1-second timeslice data collection (`MediaRecorder.start(1000)`) ensuring that runtime memory allocations remain low and stable even during multi-hour technical sessions.
* **Dependencies:** Styled via modern CSS variable theme architectures with integrated `@media (prefers-color-scheme: dark)` styling. Visual UI assets use the Tabler Icons CDN font toolkit.
