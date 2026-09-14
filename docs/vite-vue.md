# Using litert-virtual-background with Vite + Vue

Everything below was run against a fresh `create-vite` Vue-TS project: `vite dev` (with the optional
COOP/COEP headers → multi-threaded WASM), `vite build` + `vite preview`, WebGPU inference and events.
React/Svelte/plain Vite differ only in the component wrapper.

## 1. Install

```bash
bun add https://github.com/Quickom2023/litert-virtual-background-dist/releases/download/v0.3.0/litert-virtual-background-0.3.0.tgz @litertjs/core
```

`@litertjs/core` is a peer dependency (Vite bundles it into your app). The package ships prebuilt ESM plus
type declarations - no build step runs on install.

Nothing else is copied: the inference worker is embedded in the library, and LiteRT's wasm runtime is loaded
from jsDelivr by default (see [Self-hosting the runtime](#5-self-hosting-the-litert-runtime) if you cannot use
a CDN).

## 2. Models

Put the `.tflite` files in `public/models/` (Vite serves `public/` at the site root, which matches the
default `modelsPath: '/models/'`), or host them anywhere and use URLs.

```
public/
  models/
    rvm_mobilenetv3_320x480_fp16.tflite           # RVM (GPL-3.0)
    rvm_mobilenetv3_480x320_portrait_fp16.tflite  # its portrait twin (optional, phones held upright)
    pphumanseg_192x192_fp16.tflite                # PP-HumanSeg (Apache-2.0)
```

Describe them once (`src/models.ts`). Only `file`, `scale` and `offset` are required — input size, layout and
recurrent state are read from the file:

```ts
import type {ModelSpec} from 'litert-virtual-background';

export const models: Record<string, ModelSpec> = {
  rvm_hq: {
    file: 'rvm_mobilenetv3_320x480_fp16.tflite',
    scale: 1 / 255, offset: 0,                        // input = pixel * scale + offset → [0, 1]
    lighter: 'pphumanseg',                            // auto-downgrade target when this one is too slow
    portrait: {file: 'rvm_mobilenetv3_480x320_portrait_fp16.tflite'},
  },
  pphumanseg: {
    file: 'pphumanseg_192x192_fp16.tflite',
    scale: 2 / 255, offset: -1,                       // → [-1, 1]
    output: {channels: 2, alpha: 1},                  // 2-class softmax, person = channel 1
  },
};
```

A registry with only `pphumanseg` is a valid, permissively licensed setup; the RVM weights are GPL-3.0.

## 3. The component

```vue
<script setup lang="ts">
import {onMounted, onUnmounted, ref, watch} from 'vue';
import {VirtualBackground, formatEvent, type Effect} from 'litert-virtual-background';
import {models} from './models';

const canvas = ref<HTMLCanvasElement | null>(null);
const status = ref('loading…');
const effect = ref<Effect>('blur');
const blur = ref(0.6);
let vb: VirtualBackground | null = null;

onMounted(async () => {
  vb = await VirtualBackground.create({
    canvas: canvas.value!,
    models,                                   // model: first key by default, backend: 'auto'
    render: {effect: effect.value, blurStrength: blur.value},
    onStats: s => { status.value = `${s.model} ${s.inputSize} on ${s.backend} · ${s.lastInferenceMs.toFixed(0)} ms · ${s.fps.toFixed(0)} fps`; },
    onEvent: e => console.log('[vb]', formatEvent(e)),   // downgrades, fallbacks, throttling, errors
  });
});
onUnmounted(() => vb?.dispose());

// Camera needs a user gesture on most browsers: start it from a click, not from onMounted.
async function startCamera() { await vb?.startCamera(); }
async function stop() { vb?.stop(); }
async function useFile(e: Event) { const f = (e.target as HTMLInputElement).files?.[0]; if (f) await vb?.setSource(f); }
async function useImage(e: Event) { const f = (e.target as HTMLInputElement).files?.[0]; if (f) vb?.setBackgroundImage(await createImageBitmap(f)); effect.value = 'image'; }

watch([effect, blur], () => vb?.setRender({effect: effect.value, blurStrength: blur.value}));
</script>

<template>
  <canvas ref="canvas" style="width: 100%; max-width: 640px; background: #222"></canvas>
  <p>{{ status }}</p>
  <button @click="startCamera">Start camera</button>
  <button @click="stop">Stop</button>
  <input type="file" accept="video/*" @change="useFile">
  <select v-model="effect">
    <option value="blur">Blur</option><option value="image">Image</option>
    <option value="color">Colour</option><option value="none">None</option>
  </select>
  <input v-if="effect === 'blur'" type="range" min="0" max="1" step="0.05" v-model.number="blur">
  <input v-if="effect === 'image'" type="file" accept="image/*" @change="useImage">
</template>
```

Notes on the Vue side:

- Create the instance once in `onMounted` (it needs the real `<canvas>`), dispose it in `onUnmounted`.
  Keep `vb` in a plain `let`, not a `ref`/`reactive` — proxying a class that owns WebGL and worker handles
  buys nothing and costs per-frame overhead.
- `stats` is a plain object updated once a second through `onStats`; copy what you show into refs there.
- Everything is tweakable live: `setRender`, `setModel`, `setBackend`, `setFrameSync`, `setMotionGate`,
  `setMaxInferenceFps`, `setBackgroundImage`.
- The composited canvas is also a stream: `vb.outputStream()` for `RTCPeerConnection` / `MediaRecorder`, or
  the LiveKit processor below.

## 4. `vite.config.ts`

```ts
import {defineConfig} from 'vite';
import vue from '@vitejs/plugin-vue';
// import basicSsl from '@vitejs/plugin-basic-ssl';   // camera + WebGPU over LAN need https (or localhost)

export default defineConfig({
  plugins: [vue() /*, basicSsl() */],
  server: {
    // Optional. Cross-origin isolation gives SharedArrayBuffer → multi-threaded WASM, which matters on
    // browsers without WebGPU (Firefox) — measured 55 ms/frame for rvm_hq with threads vs ~250 ms without.
    headers: {'Cross-Origin-Opener-Policy': 'same-origin', 'Cross-Origin-Embedder-Policy': 'require-corp'},
  },
});
```

Nothing is needed for the worker or the wasm files. With `require-corp` every cross-origin resource your page
loads must send `Cross-Origin-Resource-Policy` or CORS — jsDelivr does; check your own CDN if you host models
there. Set the same two headers on the production server (or drop them and accept single-threaded WASM on
non-WebGPU browsers).

## 5. Self-hosting the LiteRT runtime

Default is `https://cdn.jsdelivr.net/npm/@litertjs/core@<version>/wasm/`. For offline use or a CSP that
forbids the CDN:

```bash
mkdir -p public/wasm && cp node_modules/@litertjs/core/wasm/* public/wasm/
```
```ts
VirtualBackground.create({canvas, models, litertPath: '/wasm/'});
```

## 6. Publishing to a call (LiveKit)

```ts
import {createLocalVideoTrack} from 'livekit-client';
import {createLiveKitProcessor} from 'litert-virtual-background/livekit';

const processor = createLiveKitProcessor({models, render: {effect: 'blur'}});
const track = await createLocalVideoTrack();
await track.setProcessor(processor);           // what is published is the composited video
processor.vb?.setRender({effect: 'image'});    // live tweaks through the same API
await track.stopProcessor();
```

`processor.vb` is a normal `VirtualBackground` (pass `canvas` to also preview it). The camera track stays
LiveKit's: `restart`/`destroy` never stop a track you handed in.

## 7. Diagnostics

```ts
import {benchmark, EventLog} from 'litert-virtual-background/diagnostics';

const log = new EventLog(200, line => console.log(line));   // pass log.push as onEvent
const rows = await benchmark(vb);                            // every model: median ms, fps, input size
```

Useful `debug` options for testing: `{inferDelayMs: 30}` emulates a slow device, `{simulateWorkerWithoutWebGpu: true}`
exercises the iOS-style in-thread fallback.

## 8. If something does not work

| Symptom | Cause |
|---|---|
| `Error: unknown model 'x'; known: …` / `models['x'].scale must be a number` | registry typo — checked at `create()` |
| `model https://…/x.tflite: Failed to fetch` | wrong path, or the host does not send CORS headers |
| `spec says 256x256 but the model's image input is 192x192 NHWC` | declared size disagrees with the file — drop `width`/`height` |
| `no output of 192x192 with 1 channel(s) - set output: {channels, alpha}` | multi-channel model without `output` |
| WebGPU missing on a LAN URL (`http://192.168…`) | insecure origin: browsers hide WebGPU and the camera — use https (`@vitejs/plugin-basic-ssl`) or localhost |
| `stats.wasmThreads === false` | page not cross-origin isolated (headers above), or Safari (no relaxed SIMD) |
| Firefox uses WASM although WebGPU exists | intended: Firefox's WebGPU completion waits poll at 100 ms; WASM is faster there |
| Published video freezes when the tab is hidden | canvas capture runs on `requestAnimationFrame`, like every canvas-based processor |
