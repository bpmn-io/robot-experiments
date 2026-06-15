# Electron Bundle Baseline

Measured on: 2026-06-15  
Version: Camunda Modeler 5.48.0  
Electron: 42.x

## Bundle Sizes (Linux)

| Artifact          | Size    |
|-------------------|---------|
| Compressed tar.gz | 137 MB  |
| Unpacked total    | 218 MB  |
| – Electron binary | 118 MB  |
| – Resources dir   | 51 MB   |
| – app.asar        | 50 MB   |
| Client bundle     | 32 MB   |

## What the Electron Binary Contains

The 118 MB `camunda-modeler` binary is mostly Chromium:
- `libGLESv2.so` — 6 MB
- `libvk_swiftshader.so` — 4 MB
- `libvulkan.so.1` — 2 MB
- `libffmpeg.so` — 2 MB
- `icudtl.dat` — 10 MB
- Chromium renderer + sandbox
- V8 snapshot

## What app.asar Contains

The 50 MB asar holds all Node.js app code and npm dependencies:
- `@camunda8/sdk` — gRPC-based Zeebe client (heavy: protobuf generated code)
- `@sentry/node` + `@sentry/integrations`
- `chokidar`, `fast-glob`, `saxen`, etc.

## Key Takeaway

Eliminating Electron's bundled Chromium (118 MB) is the main bundle-size win.
Even with a comparable app.asar, replacing the Electron binary alone would cut
the total size by ~54%.
