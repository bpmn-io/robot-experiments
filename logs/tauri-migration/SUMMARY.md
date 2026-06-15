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

## JS Codebase & Tooling Impact

This is the consideration most underrepresented in the bundle-size narrative: **the entire main-process JavaScript codebase — ~7,150 lines of code and ~10,000 lines of tests — does not migrate. It must be rewritten in Rust.**

### What gets replaced

The React client (`client/`) is untouched. Everything else — `app/lib/` — is discarded:

| Subsystem | Lines | Test lines | Notes |
|-----------|-------|------------|-------|
| `zeebe-api/` | 1,659 | 4,918 | Dual gRPC/REST, cert extraction, error mapping |
| `file-context/` | 1,394 | 752 | chokidar watcher, BPMN/DMN/Form XML indexing |
| `menu/` | 1,197 | 546 | ~30 dynamic menu items, platform-specific |
| `index.js` (main) | 944 | — | App lifecycle, IPC routing, session management |
| `config/` + `util/` + others | ~2,050 | ~3,800 | Mostly portable logic, but still a rewrite |
| **Total** | **~7,150** | **~10,000** | |

The test suite — 22 files, ~10,000 lines of Mocha/Sinon — cannot be carried forward. All of it must be rewritten.

### The three hardest subsystems

**`zeebe-api/`**: The most tested code in the repo (4,918 test lines). It dynamically switches between gRPC and REST, extracts TLS certificates per platform (Windows via Crypt32 DLL, macOS via `security` CLI, Linux via filesystem paths), and maps low-level error codes to user-facing messages. No official Rust port of `@camunda8/sdk` exists. REST-only mode removes the gRPC complexity, but the cert extraction, error mapping, and response transformation still need full Rust implementations.

**`file-context/`**: 1,394 lines of battle-hardened file watching (chokidar) combined with streaming XML parsing (saxen + bpmn-moddle/dmn-moddle). It maintains a live in-memory index of every BPMN/DMN/Form/RPA file in watched roots, with debounced atomic-operation handling and concurrent work queues. The Rust `notify` crate can replace chokidar, but the BPMN/DMN semantic extraction (process IDs, signal definitions, user task types, form keys) has no ready-made Rust equivalent and would be a ground-up implementation.

**`menu/`**: ~30 native menu items rebuilt on every state change (active tab, unsaved changes, dev mode, plugins). macOS-specific conventions (app menu, Services submenu, Preferences placement) are non-trivial to replicate with Tauri's menu API.

### Tooling that disappears entirely

| Tool | Role | Rust replacement |
|------|------|-----------------|
| Mocha / Chai / Sinon | Test runner, assertions, mocking | `cargo test` + `mockall` (harder) |
| nyc / Istanbul | Coverage reporting | `cargo-llvm-cov` |
| proxyquire | Module-level mocking in tests | No direct equivalent |
| ESLint (bpmn-io plugin) | Main-process linting | `cargo clippy` |
| webpack (preload bundle) | Preload bundling | Eliminated |

The gap between Sinon's module-level stubs and Rust mocking is non-trivial. Rust requires traits + dependency injection to make code testable; the existing JS code structure would need to be redesigned in Rust to achieve comparable test isolation.

### Developer experience change

- Main-process work moves from JavaScript to Rust. The team needs to acquire or hire Rust proficiency.
- The repo becomes genuinely polyglot: Rust for the app shell, JavaScript/React for the client. Every piece of functionality that crosses the IPC boundary requires changes in both languages.
- Compile-test loops are slower: Rust incremental builds take seconds to minutes vs. Node.js instant module reloads.
- Debugging tools differ: Chrome DevTools still covers the renderer; gdb/lldb or `tokio-console` replaces Node.js Inspector for the app layer.

### Rewrite effort estimate

| Scenario | Estimate |
|----------|---------|
| Experienced Rust developer, full feature parity | 6–10 weeks |
| Team learning Rust while migrating | 16–24 weeks |
| Code to rewrite (JS → Rust, estimated) | ~7,150 lines JS → ~4,000–5,000 lines Rust |
| Tests to rewrite | ~10,000 lines Mocha → ~5,000 lines Rust |

The Rust shell in this experiment is ~500 lines covering happy-path IPC. The production rewrite must match the edge-case handling and test coverage of the existing JavaScript — that is the real cost.

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
| ~7,150 lines JS main-process rewrite in Rust | **High** | Incremental migration per subsystem |
| ~10,000 lines of tests to rewrite | **High** | Rewrite alongside code; accept lower coverage initially |
| System webview distribution risk | **High** | deb/rpm primary format; startup dependency check; WebView2 bootstrapper on Windows |
| Zeebe gRPC API + cert extraction | **High** | Switch to REST API (Camunda 8.6+); port cert extraction to Rust |
| Cross-platform rendering (WebKit vs Chromium) | **High** | Dedicated QA pass; test BPMN/DMN SVG on WebKit |
| file-context watcher + BPMN/DMN indexer | **High** | Rewrite with `notify` crate + custom XML parser |
| Developer Rust proficiency | **High** | Training or hiring; affects every future main-process change |
| Plugin loading | Medium | CSS/HTML-only plugins in sandboxed iframe |
| App auto-update | Medium | `tauri-plugin-updater` + manifest migration |
| Air-gapped installation guidance | Medium | Document offline dependency pre-staging |
| Dynamic native menus | Medium | Implement via Tauri `Menu` API; macOS conventions |
| Code signing | Low | Tauri native signing config |
| Windows cert store | Low | `rustls-native-certs` crate |

---

## Recommendation

The 7× bundle size reduction is real and meaningful. The migration is technically feasible — this experiment proves the IPC bridge, the build pipeline, and the bundle size claim. However, the full picture is significantly more demanding than the bundle size number suggests:

- The ~7,150-line JavaScript main-process and its ~10,000-line test suite must be fully rewritten in Rust. This is not a tweak — it is replacing the entire app layer.
- The dev team needs Rust proficiency for all future main-process work. That is a long-term capability investment, not just an upfront cost.
- Distribution now depends on OS-managed runtimes, introducing a new support surface for enterprise and air-gapped customers.

**Required safeguards before any public release**:
1. Ship `.deb`/`.rpm` as the primary Linux format; add a startup dependency check for raw binaries.
2. Bundle the WebView2 offline bootstrapper on Windows.
3. Run the full test suite against WebKit (Linux and macOS) — BPMN/SVG rendering in particular.
4. Publish installation guidance covering the system dependency, including offline/enterprise scenarios.
5. Achieve parity with the existing Mocha test suite for all rewritten subsystems.

**Suggested incremental path**:
1. **Phase 1** — Ship core modeler (file open/save, local use) in a Tauri shell. The `zeebe-api/` and `file-context/` subsystems stay as JS running in a Node.js sidecar temporarily, preserving those features while the Rust rewrites are in progress. Bundle size win is partial but immediate.
2. **Phase 2** — Port Zeebe connectivity to REST API in Rust. Retire the Node.js sidecar.
3. **Phase 3** — Port `file-context/` watcher and BPMN/DMN indexer to Rust.
4. **Phase 4** — Redesign plugin system for Tauri's security model.
5. **Phase 5** — Replace Electron-specific CI/CD (signing, publishing, auto-update).

---

## Links

- [Baseline measurements](knowledge/01-electron-baseline.md)
- [IPC surface inventory](knowledge/02-ipc-surface.md)
- [Tauri bundle measurements](knowledge/03-tauri-bundle-measurements.md)
- [Migration architecture](knowledge/04-migration-architecture.md)
- [Migration blockers + distribution risks](knowledge/05-migration-blockers.md)
- [JS codebase impact](knowledge/06-js-codebase-impact.md)
- Worktree: `~/camunda/projects/bpmn.io/.other/experiments-repositories/tauri-migration/`
- Branch: `experiment/tauri-migration` on `camunda-modeler`
