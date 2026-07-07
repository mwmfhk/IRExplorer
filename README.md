# IR Explorer

Browser-based tool for auditioning and blending guitar cabinet impulse responses. Record a DI'd rhythm part, record a lead over it, and blend up to 8 IRs per side interactively while listening to the loop. Export the blended result as a `.wav` IR loadable by any modeler or IR loader.

Live: https://mwmfhk.github.io/IRExplorer/ir-explorer.html
Tech deep-dive: https://mwmfhk.github.io/IRExplorer/algorithms.html

## Repo layout

```
ir-explorer.html      Single-file app. All CSS + HTML + JS + AudioWorklet code
                      lives here. No build step, no framework, no bundler.
algorithms.html       Standalone technical write-up. 14 sections with inline
                      SVG diagrams covering the DSP, sync, iOS quirks, file
                      formats, and the noise-reduction pipeline.
index.html            <meta refresh> redirect to ir-explorer.html.
ir-explorer-b22.html  Snapshot of ir-explorer.html at commit 0bc3f10
                      (Build 22) for A/B testing before / after the b23+
                      changes (IR plot modal, auto-gain, noise filter).
ios-audio-test.html   10-button diagnostic page for iOS Safari Web Audio
                      issues. Tests each layer in isolation (plain
                      oscillator, buffer source, convolver, EQ chain,
                      HTML5 <audio>, etc). Used to isolate the audio-session
                      unlock problem (see algorithms.html §9).
ir-session-pr100.irz  Demo session bundled into the app via fetch().
                      Two DI'd guitar performances + a set of cab IRs.
LICENSE               Apache 2.0.
```

## Running locally

There is no build step. Open `ir-explorer.html` directly in a modern browser:

```
open ir-explorer.html   # macOS
xdg-open ir-explorer.html  # Linux
```

Some things (Web Audio unlock, mic access) only work over `https://` or `localhost` on modern browsers. If you need microphone recording locally, serve the directory instead:

```
python3 -m http.server 8000
# then visit http://localhost:8000/ir-explorer.html
```

The demo `.irz` needs to load via `fetch()` even if the app itself works from `file://`, so `python -m http.server` is the easiest bootstrap.

## Deployment

GitHub Pages serves the repo root. Every push to `main` triggers the `pages build and deployment` workflow; when it succeeds, the change is live at `mwmfhk.github.io/IRExplorer/…` typically within 30–90 s.

The pipeline occasionally fails with a transient "Deployment failed, try again later" error. When that happens, a manual re-run of the failed workflow or a subsequent empty commit clears it. Common recipe:

```
git commit --allow-empty -m "chore: force Pages redeploy" && git push
```

### Build number

`ir-explorer.html` contains a single `const BUILD_NO = N;` near the top of the inline `<script>`. It's rendered as `BUILD N` at the bottom of the desktop sidebar and next to the title on mobile. Convention: bump `BUILD_NO` by 1 on every commit that touches `ir-explorer.html`. Commits that only touch `algorithms.html`, `LICENSE`, or docs skip the bump.

## Architecture in one screen

`ir-explorer.html` is one file because deployment is one static drop. The code is organised roughly:

1. `<style>` — all CSS, including mobile media queries at `@media (max-width:1655px)` (mobile/portrait), `@media (max-width:1655px) and (orientation:landscape)` (landscape layout, phone + iPad), `@media (max-height:500px)` (font/padding tightening).
2. `<body>` DOM — sidebar / two `.card` panels / notes / modals.
3. Inline `<script>` — everything else. Loose grouping:
   - State variables (per-side buffers, gain values, IR slot arrays).
   - DOM refs.
   - K-weighting BS.1770 loudness helpers.
   - Pure-JS Cooley-Tukey radix-2 FFT (`fftRadix2`).
   - IR alignment (peak, cross-correlation, min-phase via cepstrum) and blending.
   - Auto-gain / LUFS matched-pair leveler.
   - AudioWorklet module string (recorder + noise filter) built as a template literal, converted to Blob, loaded via `addModule`.
   - Recording paths: MediaRecorder for rhythm, AudioWorklet for lead + calibration (see algorithms.html §7, §8).
   - Playback graph construction (`startPlay`, `startRightPlay`) and live re-wiring (`syncConvolvers`, `liveAddIrSlot`, etc).
   - Sync between rhythm and lead loops via `leftPhaseOffset`, `paddedDuration`, `rightSyncTrimSec`.
   - Session save/load (`.irz` = zip with MP3 recordings + raw f32 IRs + `session.json`).
   - Noise-profile spectrum + audition modals.
   - iOS Safari workarounds (audio session unlock, gesture-zoom block, mono worklet channel forcing, pointer-event drag paths, custom file-picker fallback for Save/Load/Export).

Anything non-obvious in the above list has a dedicated section in `algorithms.html` with math, SVG diagrams, and design-decision notes. That file is the authoritative reference for how anything DSP-adjacent works — the code just implements it.

## Notable design decisions (summaries, links to full detail)

Each of these is unpacked at length in `algorithms.html` — the section anchors match:

- **Polygon blend weighting** — inverse-distance weighting with a numerical epsilon at the vertices. [#idw]
- **Phase correction on export/live** — 4 modes: raw, peak alignment, cross-correlation with a chosen reference IR, real-cepstrum minimum phase. Default is No Correction; Min-Phase eliminates comb filtering between IRs at the cost of the original phase character. Live playback swaps `convolver.buffer` in place when the mode changes. [#phase, #hotswap]
- **Rhythm + Lead phase-locked looping** — the trickiest sync in the app. Lead is padded to the smallest integer multiple of the rhythm loop period so both loops re-align on every wrap. [#loopsync]
- **Latency calibration** — 4-count metronome + measurement beats; user plays a DI note on each click; per-beat peak detection with 1.5σ outlier rejection gives `rightSyncTrimSec`. Only way that works for DI (no acoustic loopback). [#latency]
- **AudioWorklet vs MediaRecorder** — lead + calibration use a custom worklet (`RecorderProcessor`) because they need sample-accurate PCM. Rhythm uses MediaRecorder because it's simpler and lossy is fine. [#worklet]
- **iOS Web Audio session unlock** — an HTML5 `<audio>` element must play FIRST to elevate the session from "ambient" to "media playback"; a Web Audio-only path stays silent on the speaker even when `ctx.state === 'running'`. Discovered via `ios-audio-test.html`. [#iosunlock]
- **Sample-rate mismatches** — pure-JS linear resampling (`jsResample`) at session load time bypasses iOS's strict `ConvolverNode.buffer` sample-rate check. [#resample]
- **File save/open across browsers** — the modern File System Access API for Chromium; classic `<a download>` and hidden `<input type=file>` fallback for iOS Safari and Firefox. [#fileformat, #wavexport]
- **Auto Gain** — matched-pair BS.1770 loudness leveler that measures both raw recordings and applies the same makeup gain to both so relative balance is preserved while their arithmetic-mean LUFS lands on the target. Falls back to per-side leveling when only one recording exists. Toggleable via the sidebar.
- **Background Noise Filtering** — real-time Log-MMSE (Ephraim-Malah) with Decision-Directed a-priori SNR, Signal Presence Uncertainty, 3-tap freq median, and asymmetric time smoothing. Runs in an AudioWorklet at W=2048, H=128. Fixed noise profile is the averaged spectrum over the quietest 5% of 200 ms candidate windows in each recording (robust to platform-level MP3/resample jitter). Full pipeline diagram + math in [#denoise].

## Session file format

`.irz` is a plain zip archive with a custom extension. Contents:

```
session.json                # top-level metadata + per-buffer descriptors
audio_left.mp3              # rhythm DI (320 kbps MP3)
audio_right.mp3             # lead DI (320 kbps MP3)
ir_left_{N}_ch{C}.f32       # rhythm-side IR N, channel C, raw 32-bit float PCM
ir_right_{N}_ch{C}.f32      # lead-side IR N, channel C, raw 32-bit float PCM
cal_pcm.f32                 # calibration mic capture, downsampled to 4 kHz
```

Recordings go through MP3 because guitar audio is perceptually transparent at 320 kbps and the size savings are meaningful. IRs stay as raw f32 because any lossy encoding shifts the magnitude response, which IS the IR — that's not tradeable. Full byte-level breakdown in algorithms.html §12.

## Exported IR format

`.wav` — standard uncompressed PCM, 16-bit signed little-endian, matches the current AudioContext sample rate. Any IR loader will read it. Peak-normalized to 0 dBFS before conversion so the exported file makes full use of the dynamic range. Full byte layout in algorithms.html §13.

## iOS Safari support

Supported and specifically hardened for. Notable adaptations:

- HTML5 `<audio>` element plays at the first user gesture to elevate the audio session (`unlockAndResume`).
- Pointer Events instead of `mouse*` events for all drag widgets, plus `touch-action: none` on the widgets themselves.
- `gesturestart` / `gesturechange` / `gestureend` prevented globally (WebKit-only pinch-zoom events); multi-touch `touchmove` also prevented; double-tap-within-300 ms suppressed.
- Save / Load / Export use a `<a download>` + hidden `<input type="file">` fallback since `showSaveFilePicker` / `showOpenFilePicker` don't exist on Safari.
- Noise-filter AudioWorklet is constructed with explicit `channelCount: 1`, `channelCountMode: 'explicit'`, `outputChannelCount: [1]` — otherwise if the rhythm recording decoded as stereo (some browsers) and lead as mono, the two worklet instances got different channel counts and one panel's audio ended up panned.
- `getUserMedia` constraints fall back from strict (no AGC/AEC/NS) to `audio: true` if iOS refuses the strict form.

## License

Apache License, Version 2.0 — see `LICENSE`.
