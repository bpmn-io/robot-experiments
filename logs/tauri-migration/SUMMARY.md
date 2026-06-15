# Experiment Summary: Migrate Desktop Modeler to Tauri

## TL;DR

Replacing Electron with Tauri reduces the Linux distribution bundle from **137 MB → 19.2 MB** (−86%, 7× smaller). The React client was reused completely unchanged. The Rust backend (~500 lines) covers all core IPC except Zeebe API and plugins.

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

## Blockers for Full Migration

| Blocker | Severity | Suggested Path |
|---------|----------|----------------|
| Zeebe gRPC API | High | Switch to REST API (Camunda 8.6+) |
| Plugin loading | Medium | CSS/HTML-only plugins in sandboxed iframe |
| App auto-update | Medium | `tauri-plugin-updater` + manifest migration |
| Dynamic native menus | Low | Implement via Tauri `Menu` API |
| Code signing | Low | Tauri native signing config |
| Windows cert store | Low | `rustls-native-certs` crate |

---

## Recommendation

A full Tauri migration is feasible and worth pursuing, especially given the 7× bundle size reduction. The suggested incremental path:

1. **Phase 1** — Ship core modeler (file open/save, local deployment) with Tauri shell. Bundle size win immediate.
2. **Phase 2** — Port Zeebe connectivity to REST API. Unblocks the majority of the workflow features.
3. **Phase 3** — Redesign plugin system for Tauri's security model.
4. **Phase 4** — Replace Electron-specific CI/CD (signing, publishing, auto-update).

---

## Links

- [Baseline measurements](knowledge/01-electron-baseline.md)
- [IPC surface inventory](knowledge/02-ipc-surface.md)
- [Tauri bundle measurements](knowledge/03-tauri-bundle-measurements.md)
- [Migration architecture](knowledge/04-migration-architecture.md)
- [Migration blockers](knowledge/05-migration-blockers.md)
- Worktree: `~/camunda/projects/bpmn.io/.other/experiments-repositories/tauri-migration/`
- Branch: `experiment/tauri-migration` on `camunda-modeler`
