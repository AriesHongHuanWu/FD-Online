# FD-Online

Browser-based fall detection: open a URL, allow the camera, and pose estimation runs locally in the tab. When a fall is detected it saves a clip that starts five seconds *before* the event, and can optionally send it to Discord with a Gemini description of the scene.

Live: **https://arieshonghuanwu.github.io/FD-Online/**

The desktop versions of this project needed Python, a model download and a machine to run on. That is a lot of setup to show someone a demo. This one is a single HTML file: the pose model streams from a CDN, inference runs on the viewer's own device, and no video leaves the browser unless the user configures a Discord webhook. The interface is in Traditional Chinese.

This is a student computer-vision project, not a medical device and not a certified safety system. The fall rule is a geometric heuristic on body landmarks, it has not been clinically or statistically validated, and it will both miss real falls and raise false alarms. Do not rely on it for anyone's safety.

## Project family

This repository is one of five related to the same line of work.

| Repo | Role |
| --- | --- |
| [falldetection](https://github.com/AriesHongHuanWu/falldetection) | Desktop Python detector: YOLOv8 person boxes plus a user-drawn bed region. The original implementation. |
| [FDdemo](https://github.com/AriesHongHuanWu/FDdemo) | The same desktop app repackaged with the model weights committed so it runs offline. |
| **FD-Online** (this repo) | Browser version. The deployed one, and the only one with pre-event video capture and vision-model analysis. |
| [FD-PD](https://github.com/AriesHongHuanWu/FD-PD) | The same browser idea rebuilt as ES modules, trading this repo's recording and alerting for Kalman filtering, obstacle context and biomechanics readouts. |
| [Disaster-OS](https://github.com/AriesHongHuanWu/Disaster-OS) | Separate application in the same emergency-response theme. Shares no code with the detectors, but reuses the pattern of asking a vision model for structured JSON, first tried here. |

The desktop detectors classify a person-shaped rectangle. This one reads 33 body landmarks and reasons about torso orientation, which is a better signal but depends on the whole torso being visible.

## How it works

Everything is in `index.html`: markup, Tailwind classes from the CDN, and about 660 lines of inline JavaScript. There is no build step and no bundler. `.github/workflows/static.yml` publishes the repository root to GitHub Pages on every push to `main`.

### Detection loop

`startCamera()` requests `getUserMedia`, then a `requestAnimationFrame` loop (`poseLoop`) pushes each frame into MediaPipe Pose (`modelComplexity: 1`, smoothing on, detection and tracking confidence 0.5). `onResults()` receives the landmarks and does three things: draws the 2D skeleton over the video on `#output-canvas` with `drawConnectors` and `drawLandmarks`, drives the Three.js scene from `poseWorldLandmarks`, and calls `checkFall()`.

`checkFall()` takes the shoulder midpoint and the hip midpoint and measures the torso angle:

```js
const angleDeg = Math.atan2(verticalDist, horizontalDist) * (180 / Math.PI);
const isHorizontal = angleDeg < 45;   // torso closer to flat than upright
const isLow        = hipMidY > 0.5;   // hips in the lower half of the frame
```

Both conditions must hold for `CONFIG.fallTriggerFrames` (10) consecutive frames before `triggerAlarm()` fires. The live angle is shown in the UI, so the threshold can be inspected while testing.

### Pre-event video buffer

This is the part worth reading. An alarm that starts recording when it fires misses the fall itself, so `initRecordingStream()` calls `canvas.captureStream()` and keeps a `MediaRecorder` running continuously with `recorder.start(1000)`, emitting one chunk per second. While no alarm is active, `ondataavailable` discards chunks older than `CONFIG.preFallDuration` (5000 ms), so memory holds a sliding five-second window.

On a fall, `triggerFallEvent()` calls `requestData()` to flush the current chunk, captures a JPEG snapshot with `canvas.toBlob()`, and schedules `finishRecording()` after `CONFIG.postFallDuration` (10000 ms). The result is a clip spanning both sides of the event. `startBuffering()` negotiates a container with `MediaRecorder.isTypeSupported()`, preferring MP4 on desktop and falling back through QuickTime to WebM on mobile, because browsers disagree about what they will record. The list keeps `CONFIG.maxRecordings` (5) clips and calls `URL.revokeObjectURL()` on the ones it drops.

### Optional integrations

Both stay off until the user enters their own credentials in the settings panel. Those are kept in `localStorage` and used only from the browser.

- `analyzeWithAI()` posts the snapshot to `gemini-1.5-flash-latest` and asks for a short description of posture, likely cause and severity. It is a description of an image, not a diagnosis, and the model can be wrong about all of it.
- `sendDiscordWebhook()` and `uploadVideoToDiscord()` post the snapshot and the clip to a Discord webhook.

### 3D view

`initThreeJS()` builds 33 spheres and 16 connecting lines, updated every frame from `poseWorldLandmarks`, with `OrbitControls`, a grid and exponential fog. Points and lines turn red while an alarm is active.

## Tech stack

No package manager. Every dependency is a CDN `<script>` tag in `index.html`.

| Library | Use |
| --- | --- |
| `@mediapipe/pose` | 33-landmark body pose estimation in the browser |
| `@mediapipe/drawing_utils`, `camera_utils` | Skeleton overlay helpers |
| `three.js` r128 with `OrbitControls` | 3D skeleton view |
| Tailwind CSS (CDN) | Styling |
| Font Awesome | Icons |
| `MediaRecorder`, `canvas.captureStream` | Rolling video buffer, browser APIs |
| Gemini 1.5 Flash, Discord webhooks | Optional, user-supplied keys |

## Getting started

Use the [live version](https://arieshonghuanwu.github.io/FD-Online/), or serve the file locally. `getUserMedia` needs a secure context, so `https://` or `localhost` works and opening `index.html` from the filesystem does not.

```bash
git clone https://github.com/AriesHongHuanWu/FD-Online.git
cd FD-Online
python -m http.server 8000
# open http://localhost:8000
```

Allow camera access. Detection starts on its own. To enable Gemini analysis or Discord upload, open the settings panel and paste your own API key or webhook URL.

## Status and limitations

A working demo built over six commits on 2 December 2025, and the one project in this family that anyone can try without installing anything. Known boundaries:

- **No accuracy figures.** No labelled dataset, no evaluation, no measured precision or recall. This README quotes no numbers because there are none to quote.
- **Torso angle only.** No velocity, no acceleration, no ground plane. Lying down deliberately reads as a fall; a slow slide down a wall may not.
- **Single person.** MediaPipe Pose tracks one subject, and the whole torso needs to be in frame.
- **Ten frames is not a fixed duration.** The trigger counts frames, so on a slow device the confirmation delay stretches.
- **Clearing the alarm is manual.** `resetAlarm()` runs only when the user dismisses it.
- **Nothing is persisted.** Clips are in-memory blob URLs and are gone on reload. Only the API key and the webhook survive, in `localStorage`.
- **Keys live in the browser.** There is no backend, so requests go straight from the tab to Google or Discord with the user's own key. Treat any key entered here as exposed to that browser profile.
- **Camera and codec support vary.** Recording depends on `MediaRecorder` behaviour that differs across iOS Safari, Android Chrome and desktop browsers.
- **Not implemented:** any server, any account system, any stored history of events.

## License

MIT. See [LICENSE](LICENSE).
