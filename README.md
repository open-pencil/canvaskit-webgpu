# canvaskit-webgpu

Custom [CanvasKit](https://skia.org/docs/user/modules/canvaskit/) WASM build with **Skia Graphite + Dawn (WebGPU)** backend, replacing the default Ganesh + WebGL backend.

Built for [OpenPencil](https://github.com/open-pencil/open-pencil).

## Why

Google [removed the Dawn/WebGPU backend](https://issues.chromium.org/issues/376862750) from CanvasKit's standard builds. The Graphite rendering engine offers better performance on modern GPUs, but CanvasKit only ships with Ganesh/WebGL. This repo patches Skia to produce a working CanvasKit WASM with Graphite/WebGPU.

## Usage

Download `canvaskit.js` + `canvaskit.wasm` from [Releases](https://github.com/open-pencil/canvaskit-webgpu/releases), then:

```js
const ck = await CanvasKitInit({
  locateFile: (file) => `/canvaskit-webgpu/${file}`
})

const adapter = await navigator.gpu.requestAdapter()
const device = await adapter.requestDevice()

const devCtx = ck.MakeGPUDeviceContext(device)
const canvasCtx = ck.MakeGPUCanvasContext(devCtx, canvas)

// Each frame: create surface → draw → flush
const surface = ck.MakeGPUCanvasSurface(canvasCtx, ck.ColorSpace.SRGB)
const skCanvas = surface.getCanvas()
// ... draw ...
surface.flush()
```

> **Note:** Graphite requires a fresh surface per frame (wrapping the current swapchain texture). Don't reuse surfaces across frames. Don't dispose the surface until the next frame — the browser needs the texture alive to present it.

## Build

Requires: git, Python 3, CMake, Ninja, ~20 min on M-series Mac.

```bash
./build.sh
```

Output: `out/canvaskit.js` + `out/canvaskit.wasm`

Set `SKIA_DIR` to reuse an existing Skia checkout:

```bash
SKIA_DIR=~/skia ./build.sh
```

## What's patched

6 files in Skia, 1 in Dawn (see `patches/`):

| File | Change |
|------|--------|
| `modules/canvaskit/BUILD.gn` | Add `CK_ENABLE_WEBGPU` define, merge `EXPORTED_RUNTIME_METHODS` |
| `modules/canvaskit/canvaskit_bindings.cpp` | Graphite context/recorder, surface creation via `WrapBackendTexture`, flush via snap+submit, embind registration |
| `modules/canvaskit/webgpu.js` | Per-frame surface from swapchain texture, Path→PathBuilder compat shim |
| `src/gpu/graphite/dawn/DawnBuffer.cpp` | Fix `SKGPU_LOG` constexpr issue, missing paren, wrong variable name |
| `src/gpu/graphite/dawn/DawnCaps.cpp` | Guard Dawn extensions not available in Emscripten WebGPU headers |
| `third_party/dawn/BUILD.gn` | Skip native Dawn library for WASM builds |
| Dawn `include/tint/tint.h` | Add missing include for `bindings.h` |

## Pinned versions

| Component | Commit |
|-----------|--------|
| Skia | `f886711f180d6987f0ec7dd6bf04f85e4c86c50d` |
| Dawn | `bc602f0c897dde1d9bae1449477b0877c17150f3` |

## GN args

```
skia_enable_graphite = true
skia_enable_ganesh = false
skia_use_dawn = true
skia_use_webgpu = true
```

---

Maintained as part of [OpenPencil](https://github.com/open-pencil) for the [OpenPencil editor](https://github.com/open-pencil/open-pencil).

## License

Skia and Dawn are licensed under BSD-3-Clause. Patches in this repo are MIT.
