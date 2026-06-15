# Tauri Bundle Measurements

Measured on: 2026-06-15  
Tauri CLI: 2.11.2  
Rust: 1.95.0

## Linux Bundle Comparison

| Artifact                | Electron 42.x | Tauri 2.x | Reduction |
|-------------------------|--------------|-----------|-----------|
| Compressed distribution | **137 MB**   | **19.2 MB** | −86% (7.1×) |
| Unpacked / installed    | **218 MB**   | **41 MB**   | −81% (5.3×) |
| Main binary             | 118 MB       | 22 MB     | −81% (5.4×) |
| Frontend assets         | 50 MB (asar) | 0 MB *    | −100%     |

\* Frontend (32 MB) is embedded inside the Tauri binary at compile time via Tauri's
  asset inlining; it does not appear as separate files in the installed bundle.

## Why So Much Smaller

| Component            | Electron              | Tauri               |
|----------------------|-----------------------|---------------------|
| Browser engine       | Bundled Chromium 118 MB | OS WebKit2GTK (0 MB) |
| Runtime              | Node.js embedded      | Rust, statically linked |
| Frontend assets      | Packaged in .asar     | Compressed into binary |
| Node.js modules      | 50 MB in asar (incl. @camunda8/sdk) | Not needed |

## Binary Size Breakdown (Tauri)

- Tauri runtime + core + plugins: ~5 MB (estimated)
- Embedded frontend (32 MB → ~17 MB compressed): ~17 MB
- Total binary stripped: **22 MB**
- Total deb package: **19.2 MB** (further compressed)

## System Dependency Trade-off

The Tauri bundle requires WebKit2GTK as a system runtime dependency:

```
Depends: libwebkit2gtk-4.1-0 (>= 2.40.0)
```

- **Pro**: No bundled browser engine = smaller download
- **Con**: Users without WebKit2GTK must install it first (~30 MB)
- **Linux**: WebKit2GTK is available on all major distros (Ubuntu, Fedora, Arch, etc.)
- **macOS/Windows**: WKWebView / WebView2 are part of the OS

Net download for a fresh user (Linux):
- Electron: 137 MB (fully self-contained)
- Tauri: 19.2 MB + 30 MB WebKit2GTK (if not present) ≈ first-time ~49 MB, subsequent updates ~19 MB
