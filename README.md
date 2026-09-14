# litert-virtual-background

Real-time virtual background for the browser: person matting plus background blur or replacement, running
on [LiteRT.js](https://developers.google.com/edge/litert/web) (WebGPU, with a WASM fallback) and a WebGL2
compositor. Inference runs in a Web Worker; the compositor runs at display rate, so a slow model lowers the
matte update rate instead of dropping frames.

The library ships **no models**. The application passes the `.tflite` files it wants to use, so model choice
and model licensing stay with the application.

> **This repository holds distribution artifacts only.** Each release attaches a tarball of the prebuilt
> library (ESM + type declarations). There is no source here; open an issue for questions or bug reports.

## Requirements

| | |
|---|---|
| **Secure context** | `https://` or `localhost`. An insecure origin disables the camera, WebGPU and `SharedArrayBuffer` in every browser. |
| **WebGL2** | Hard requirement - the compositor throws without it. |
| **WebGPU** | Strongly recommended. Without it inference falls back to WASM, which needs COOP/COEP headers to be multi-threaded, and is slow without them. |
| **Peer dependency** | `@litertjs/core` |
| **Model files** | `.tflite` files reachable from the page (a `modelsPath` prefix or full URLs) |

Firefox is automatically put on WASM: its WebGPU resolves completion waits on a ~100 ms poll, which caps
readback-based inference at ~10 fps regardless of model size. See *Fallback engine* for handling this.

## Install

```bash
bun add https://github.com/Quickom2023/litert-virtual-background-dist/releases/download/v0.3.0/litert-virtual-background-0.3.0.tgz @litertjs/core
```

Or with npm / pnpm / yarn - the tarball is a plain package and no build step runs on install. Pin a version by
using that release's URL;
[the releases page](https://github.com/Quickom2023/litert-virtual-background-dist/releases) lists them all.

Framework recipe (Vite + Vue/React, config and troubleshooting): [docs/vite-vue.md](docs/vite-vue.md).

## Quick start

```ts
import {VirtualBackground, formatEvent, type ModelSpec} from 'litert-virtual-background';

// The models your application ships. Input geometry and layout are read from the .tflite at load;
// only the normalisation has to be stated (input = pixel * scale + offset).
const models: Record<string, ModelSpec> = {
  rvm_hq: {
    file: 'rvm_mobilenetv3_320x480_fp16.tflite',
    scale: 1 / 255, offset: 0,
    lighter: 'pphumanseg',                                              // next step of the auto-downgrade
    portrait: {file: 'rvm_mobilenetv3_480x320_portrait_fp16.tflite'},   // used for portrait sources
  },
  pphumanseg: {
    file: 'pphumanseg_192x192_fp16.tflite',
    scale: 2 / 255, offset: -1,
    output: {channels: 2, alpha: 1},        // 2-class softmax (background, person)
  },
};

const vb = await VirtualBackground.create({
  canvas: document.querySelector('canvas')!,   // output (WebGL2)
  modelsPath: '/models/',                      // URL prefix for the file names above
  models,
  model: 'rvm_hq',
  backend: 'auto',                             // 'webgpu' | 'wasm' | 'auto'
  render: {effect: 'blur', blurStrength: 0.6}, // or 'image' | 'color' | 'none' | 'mask'
  onStats: s => console.log(s.fps, s.lastInferenceMs),
  onEvent: e => console.log(formatEvent(e)),
});

await vb.setSource(await navigator.mediaDevices.getUserMedia({video: true}));
vb.setRender({effect: 'image'});
vb.setBackgroundImage(await createImageBitmap(file));
const stream = vb.outputStream(30);            // composited MediaStream for WebRTC / MediaRecorder
```

`ModelSpec` fields: `file`, `scale`, `offset`, optional `width`/`height`/`layout` (checked against the file),
`output: {channels, alpha}` (default `{channels: 1, alpha: 0}`), `lighter`, `gmacs`, `portrait`. Multi-input
recurrent models (state in, state out) are wired automatically by tensor shape.

## Fallback engine

Where WebGPU is unusable - Firefox, no adapter, older iOS Safari - an application that already has a lighter
processor can hand it over and keep talking to one object:

```ts
import {createBackground, type BackgroundEngine} from 'litert-virtual-background';

const bg: BackgroundEngine = await createBackground({
  canvas, models, render: {effect: 'blur'},
  fallback: (options, probe) => import('./my-other-processor').then(m => m.create(options, probe)),
  onPick: (pick, probe) => console.log(pick, probe),     // 'litert' | 'fallback'
});

bg.setRender({effect: 'image'});                          // same calls on either engine
```

`BackgroundEngine` is the shared surface (`setRender`, `setBackgroundImage`, `setSource`, `startCamera`,
`outputStream`, `stats`, `stop`, `dispose`); `VirtualBackground` implements it. The fallback is supplied by
the application - typically a lazy import, so the losing engine is never downloaded.

The choice comes from `probeWebGpu()`, which is memoised per page (~120 ms on a cold page, free afterwards):

- `defaultPick` → `fallback` when there is no adapter, or when completion latency exceeds
  `MAX_COMPLETION_MS` (20 ms); otherwise `litert`
- override with `pickEngine: probe => 'litert' | 'fallback'`

## LiveKit

`litert-virtual-background/livekit` is a `TrackProcessor` - no dependency on `livekit-client`, the interface
is matched structurally.

```ts
import {createLiveKitProcessor} from 'litert-virtual-background/livekit';

const processor = createLiveKitProcessor({models, render: {effect: 'blur'}, fallback: () => myOtherEngine()});
await track.setProcessor(processor);          // publishes the composited video
processor.engine?.setRender({effect: 'image'});
processor.pick;                                // 'litert' | 'fallback' | null before init
await track.stopProcessor();
```

A stream passed to `setSource()` stays yours: `stop()`/`dispose()` detach without stopping its tracks; only
cameras opened by `startCamera()` are stopped. As with every canvas-based processor, the published video
pauses while the tab is hidden (`requestAnimationFrame` stops).

## API

| | |
|---|---|
| `VirtualBackground.create(options)` | loads the runtime, detects WebGPU, compiles the first model |
| `setSource(MediaStream \| string \| Blob \| HTMLVideoElement)` / `startCamera(deviceId?)` | input |
| `processImage(image)` | one-shot matte for a still, returns a summary |
| `setRender(partial)` / `setBackgroundImage(img)` | effect, blur strength, colour, edge refinement, light wrap |
| `setModel(id)` / `setBackend('webgpu' \| 'wasm' \| 'auto')` | switch at runtime |
| `setFrameSync('auto' \| 'on' \| 'off')` | composite each matte with the frame it came from |
| `setInference({pad, recurrent, gpuPreprocess, ...})` | pre-processing and recurrent-state options |
| `setMotionGate(on, threshold?, maxIdleMs?)` | skip inference while the picture is static |
| `setMaxInferenceFps(n)` / `setThermalGuard(on)` / `setAutoDowngrade(on, budgetMs?)` | load management |
| `outputStream(fps?)` | composited `MediaStream` |
| `stats` | live model, backend, fps, timings, gate ratio, throttle state |
| `stop()` / `dispose()` | |

Everything the library decides at runtime is reported as data through `onEvent` - model and backend changes,
auto-downgrades, thermal throttling, fallbacks, warnings, errors. The library never writes to the console;
`formatEvent(e)` renders one log line if you want that.

### Adaptive behaviour

- **Auto model/backend** - if the first inferences exceed `budgetMs` (60 ms), the same model is measured on
  the other backend, then the spec's `lighter` model is loaded. Each step is an event.
- **Motion gate** (default on) - a luma thumbnail of the model input is compared on the GPU with the last
  segmented frame; while the picture is static the model is skipped and the previous matte reused.
- **Thermal guard** (default on) - when the running median inference time stays ≥ 1.5× the session baseline,
  inference is capped at 15 fps, then stepped down to a lighter model.
- **Portrait sources** - a spec with a `portrait` twin switches to it automatically when the source is taller
  than wide, so a phone held upright is not squashed into a landscape input.
- **720p cap** (`maxHeight`, default 720) - cameras are opened at up to 720 rows and the compositor renders at
  most that, even for a 1080p source.

## Download size

First visit, nothing cached (measured, gzip/brotli in transit):

| | |
|---|---|
| Library JS (minified + gzip) | ~30 KB |
| LiteRT wasm runtime (jsDelivr by default, self-hostable via `litertPath`) | ~2.5 MB |
| A 480×320 matting model (fp16) | ~6.6 MB |
| A 192² segmentation model (fp16) | ~2.7 MB |

Serve `.wasm` and `.tflite` compressed and cache them with a long `max-age` - the runtime is 8.9 MB raw
versus 2.5 MB compressed.

## Licence

MIT - see `LICENSE`. This covers the library code only. Model weights are the application's choice and keep
their own licences; no `.tflite` file is included in this package.
