# Experiment Summary: Migrate Desktop Modeler to Tauri

## TL;DR

Replacing Electron with Tauri reduces the Linux distribution bundle from **137 MB → 19.2 MB** (−86%, 7× smaller). The React client was reused completely unchanged. The Rust backend (~500 lines) covers all core IPC except Zeebe API and plugins.

However, the move to a native system webview introduces a new class of distribution and support risks that don't exist with Electron's self-contained Chromium. These need to be resolved before shipping.

---

## What Was Done

1. Scaffolded a Tauri 2.x shell in a git worktree of `camunda-modeler`.
2. Wrote a JS bridge (`bridge.js`) injected as a Tauri initialization script that exposes `window.getAppPreload()` with an identical shape to the Electron preload — the React client needed **zero changes**.
3. Implemented ~30 IPC handlers in Rust covering: file I/O, dialogs, config, workspace, app lifecycle, clipboard, and external URLs.
4. Built a deb package and measured the result.

**Worktree**: `experiments-repositories/tauri-migration` (branch `experiment/tauri-migration`)

---

## Bundle Size Results (Linux)

| Metric              | Electron 42.x | Tauri 2.x | Reduction |
|---------------------|:-------------:|:---------:|:---------:|
| Compressed download | 137 MB        | **19.2 MB** | **−86%** |
| Installed size      | 218 MB        | 41 MB     | −81%      |
| Main binary         | 118 MB        | 22 MB     | −81%      |

**Root cause of saving**: Electron bundles Chromium (118 MB). Tauri delegates rendering to the OS-native WebKit2GTK, which contributes 0 bytes to the download.

The 32 MB React client bundle is embedded (compressed) inside the Tauri binary at compile time, so there are no separate asset files.

---

## What Works

- App boots and renders the React UI via Tauri (WebKit2GTK)
- File open / save / read-stats
- Open-file and save-file dialogs with filters
- Config `get` / `set` (JSON persistence)
- Workspace save / restore
- App reload / restart / quit flow
- External URL opening
- Clipboard write
- Message / error dialogs

## What Was Stubbed

- **Zeebe API** — gRPC Node.js SDK has no direct Rust equivalent (see Blockers)
- **Plugin loading** — Tauri has no equivalent to Electron's runtime JS loading
- **Dynamic native menus** — Tauri `Menu` API needs a separate implementation
- **File context / LSP watcher** — could use the `notify` Rust crate

---

## Key Learnings

1. **The IPC surface maps cleanly to Tauri commands.** The Electron request/response pattern (`ipcRenderer.send` + response callback) maps directly to `invoke()` returning a Promise. The bridge was straightforward to write.

2. **Tauri initialization scripts replace the Electron preload.** `WebviewWindowBuilder::initialization_script()` runs before page JS, bypassing the page CSP — the exact same guarantee Electron's preload provides.

3. **Frontend embedding is free.** Tauri inlines all frontend assets at compile time. No packaging step needed; the binary is self-contained.

4. **Zeebe API is the hardest migration blocker.** The `@camunda8/sdk` gRPC client has no direct Rust equivalent. The recommended path is to switch to Camunda's REST API (available since v8.6), which eliminates gRPC entirely.

5. **Plugin system needs a redesign.** Electron plugins execute arbitrary JS in the renderer. This is not possible in Tauri by design. A sandbox-iframe or CSS-only plugin model would be needed.

---

## Distribution & Runtime Risk (New Considerations)

This is the fundamental trade-off of Tauri vs. Electron. **Electron is self-contained; Tauri is not.** The OS-managed webview is what makes the bundle small, but it also means customers must have the right runtime installed.

### Per-platform situation

| Platform | Runtime needed | Availability | Risk |
|----------|---------------|-------------|------|
| **Linux** | WebKit2GTK 4.1 | In distro repos; not pre-installed on minimal/enterprise systems | 🔴 High |
| **Windows** | WebView2 (Edge/Chromium) | Win 11 built-in; Win 10 via Windows Update; may be blocked by GPO | 🟡 Medium |
| **macOS** | WKWebView | Part of macOS since 10.10; always present | 🟢 Low |

### Linux — the hardest case

WebKit2GTK is absent on RHEL 7/8 minimal installs, Ubuntu Server, and other
stripped-down systems common in enterprise environments. Air-gapped machines
cannot resolve it automatically at all.

Installing via `apt install ./camunda-modeler.deb` handles the dependency automatically
on systems with internet access and standard repos — but a raw binary or AppImage
silently exits with no user-visible error if the runtime is missing.

Today's install story (`.tar.gz` → extract → run) **breaks** without WebKit2GTK.

**Required before shipping**:
- Ship `.deb`/`.rpm` as the primary Linux format (not raw binaries) so the package
  manager resolves the dependency automatically.
- Add a startup wrapper that detects a missing WebKit2GTK and prints a clear,
  actionable error (`sudo apt install libwebkit2gtk-4.1-0`).
- Provide offline installation guidance for air-gapped environments.

### Windows — enterprise GPO risk

WebView2 is present on virtually all consumer machines. The risk is enterprise IT:
Group Policy can block Edge/WebView2 updates on managed machines, and some
organizations explicitly disable it.

**Mitigation**: Tauri supports bundling an offline WebView2 bootstrapper (~2 MB
added to the installer). This removes the GPO risk entirely but partially erodes
the size saving.

### Cross-platform rendering is no longer deterministic

With Electron, every user runs the same Chromium. With Tauri, the rendering engine
depends on OS and version:

| Platform | Engine | Comparable to |
|----------|--------|--------------|
| Linux | WebKit (GTK) | Safari |
| macOS | WebKit (WKWebView) | Safari |
| Windows | Chromium (WebView2) | Chrome/Edge |

**Practical risks for the Modeler**:
- The BPMN/DMN/Forms client renders complex SVG canvases. WebKit has historically
  had SVG rendering bugs that Blink/Chromium does not.
- CSS layout edge cases differ between WebKit and Chromium.
- JavaScript engine: macOS/Linux get JavaScriptCore (not V8). Certain patterns
  around large typed arrays or Promise scheduling may behave differently.
- The existing test suite was validated against Chromium. A dedicated QA pass
  against WebKit (Linux + macOS) is required before shipping.

---

## Blockers for Full Migration

| Blocker | Severity | Suggested Path |
|---------|----------|----------------|
| System webview distribution risk | **High** | deb/rpm primary format; startup dependency check; WebView2 bootstrapper on Windows |
| Zeebe gRPC API | High | Switch to REST API (Camunda 8.6+) |
| Cross-platform rendering (WebKit vs Chromium) | High | Dedicated QA pass; test BPMN/DMN SVG on WebKit |
| Plugin loading | Medium | CSS/HTML-only plugins in sandboxed iframe |
| App auto-update | Medium | `tauri-plugin-updater` + manifest migration |
| Air-gapped installation guidance | Medium | Document offline dependency pre-staging |
| Dynamic native menus | Low | Implement via Tauri `Menu` API |
| Code signing | Low | Tauri native signing config |
| Windows cert store | Low | `rustls-native-certs` crate |

---

## Recommendation

A full Tauri migration is feasible and the 7× bundle size reduction is a real, meaningful win. However, the distribution risk means it cannot be treated as a drop-in replacement — it requires a deliberate rollout strategy.

**Required safeguards before any public release**:
1. Ship `.deb`/`.rpm` as the primary Linux format; add a startup dependency check for raw binaries.
2. Bundle the WebView2 offline bootstrapper on Windows.
3. Run the full test suite against WebKit (Linux and macOS).
4. Publish installation guidance covering the system dependency, including offline/enterprise scenarios.

**Suggested incremental path**:
1. **Phase 1** — Ship core modeler with Tauri shell + the safeguards above. Bundle size win immediate.
2. **Phase 2** — Port Zeebe connectivity to REST API. Unblocks the majority of workflow features.
3. **Phase 3** — Redesign plugin system for Tauri's security model.
4. **Phase 4** — Replace Electron-specific CI/CD (signing, publishing, auto-update).

---

## Links

- [Baseline measurements](knowledge/01-electron-baseline.md)
- [IPC surface inventory](knowledge/02-ipc-surface.md)
- [Tauri bundle measurements](knowledge/03-tauri-bundle-measurements.md)
- [Migration architecture](knowledge/04-migration-architecture.md)
- [Migration blockers + distribution risks](knowledge/05-migration-blockers.md)
- Worktree: `~/camunda/projects/bpmn.io/.other/experiments-repositories/tauri-migration/`
- Branch: `experiment/tauri-migration` on `camunda-modeler`
