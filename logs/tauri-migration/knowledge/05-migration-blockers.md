# Tauri Migration Blockers

Issues that would need to be resolved for a full production migration.

## Blocker 1: Zeebe API (gRPC)

**Severity**: High — core feature

The `@camunda8/sdk` package uses gRPC to talk to Camunda 8 gateway and the
Operate/Tasklist HTTP APIs. There is no equivalent Rust SDK that covers all the
same operations.

**Options**:
1. **Port to Rust gRPC** — Use `tonic` crate to implement gRPC. The Zeebe proto
   files are public. Significant effort (~2-4 weeks per API).
2. **REST-only mode** — Camunda 8 exposes all Zeebe operations via REST since 8.6.
   Replace gRPC calls with `reqwest` HTTP calls. Simpler and supported going forward.
3. **Node.js sidecar** — Keep a stripped-down Node.js sidecar just for Zeebe comms.
   Defeats the bundle-size purpose for this subsystem.

**Recommended path**: Option 2 (REST APIs). Camunda is deprecating gRPC for client
usage in favor of REST anyway.

## Blocker 2: Plugin System

**Severity**: Medium — extension ecosystem depends on it

Electron plugins are arbitrary JS/CSS files loaded at runtime via `webContents.executeJavaScript()`. Tauri has no equivalent — it doesn't allow loading arbitrary untrusted scripts into the webview (by design, for security).

**Options**:
1. **Remove plugins** — The modeler ships with no third-party plugins in the default bundle; the plugin API is used by integrators. This could be a breaking change.
2. **Webview-only plugins** — Allow plugins to add only CSS/HTML (no privileged IPC). Most existing plugins only customize the UI.
3. **Extension protocol** — Build a custom protocol where plugins run in a sandboxed iframe and communicate via a restricted message channel.

## Blocker 3: Dynamic Native Menus

**Severity**: Low — UI polish

The modeler registers dynamic menu items from the client via `menu:register`. Tauri's menu API is synchronous at startup; the current stub silently drops all menu registrations.

**Fix**: Map the `menu:register` command to the Tauri `Menu` API at runtime. Tauri v2 supports mutable menus; the mapping is mechanical work (~1-2 days).

## Blocker 4: Windows Certificate Store

**Severity**: Low — enterprise feature

`vscode-windows-ca-certs` injects Windows CA certificates so the modeler can connect to self-signed Camunda installations in corporate networks.

**Fix**: Use the `rustls-native-certs` or `native-tls` crate in Rust to read the Windows cert store natively. Cleaner than the Node.js native addon approach.

## Blocker 5: App Auto-Update

**Severity**: Medium — required for shipping

The Electron build uses `electron-builder` S3 publishing for auto-updates.

**Tauri equivalent**: `tauri-plugin-updater` with a compatible JSON update manifest.
The manifest format differs from electron-updater; CI pipeline changes needed.

## Blocker 6: Code Signing / Notarization

**Severity**: Medium — required for shipping on macOS/Windows

Electron's `afterSign` hook uses `@electron/notarize`. Tauri has built-in signing
hooks via environment variables (`TAURI_SIGNING_PRIVATE_KEY`, etc.).

**Fix**: Replace the custom `afterSign.mjs` with Tauri's native signing config.
Mechanical work but needs Apple Developer / Windows cert setup changes.

## Blocker 7: System Webview — Distribution & Installation Risk

**Severity**: High — customer-facing; affects all platforms

This is the fundamental trade-off of moving to Tauri. Electron ships its own Chromium
and is fully self-contained. Tauri delegates rendering to an OS-managed webview, which
means customers must have the right runtime installed.

### Per-platform situation

| Platform | Webview | Availability | Risk |
|----------|---------|-------------|------|
| **Linux** | WebKit2GTK 4.1 | In distro repos; NOT pre-installed on minimal systems | **High** |
| **Windows** | WebView2 (Edge/Chromium) | Bundled with Win 11; auto-installed on Win 10 via Windows Update | **Medium** |
| **macOS** | WKWebView | Part of macOS since 10.10; always present | **Low** |

### Linux — the hardest case

- WebKit2GTK is _not_ pre-installed on Ubuntu Server, RHEL, or other minimal systems.
- Enterprise customers may use locked-down distros (RHEL 7/8, CentOS) where
  webkit2gtk-4.1 is not in the default repos.
- Installing via `apt`/`dnf` requires internet access and root.
- The deb/rpm package _lists_ the dependency, so `apt install ./camunda-modeler.deb`
  will pull it automatically — but only if the machine has internet and the repo.
- Air-gapped or offline installations need the dependency pre-staged.
- There is no guaranteed minimum version; different users run different WebKit builds.

**Mitigation options**:
1. **AppImage with bundled WebKit** — AppImage format allows bundling the webview.
   Size advantage partially lost but user gets a self-contained binary.
2. **Detect and guide** — On startup, detect missing WebKit and show a clear error
   with installation instructions.
3. **Ship a webview installer** — Tauri supports bundling a WebView2 bootstrapper
   on Windows; no equivalent for Linux.

### Windows — medium risk (enterprise)

- WebView2 ships with Windows 11 and has been auto-pushed to Windows 10 via
  Windows Update since 2021. >99% of consumer machines have it.
- **Enterprise risk**: IT policies or Group Policy Objects (GPO) can block Edge /
  WebView2 updates on managed machines. Some enterprises explicitly disable it.
- Air-gapped Windows installations will not have auto-installed WebView2.
- **Mitigation**: Tauri supports bundling a WebView2 bootstrapper (offline installer)
  that runs at first launch. This adds ~2 MB to the package but removes the
  dependency risk entirely on Windows.

### Cross-platform rendering inconsistency

**This is a significant, often underestimated risk.**

With Electron, every user runs the same Chromium version — rendering is deterministic.
With Tauri, rendering depends on the OS-provided webview version:

| Platform | Engine | Comparable to |
|----------|--------|--------------|
| Linux | WebKit (GTK) | Safari |
| macOS | WebKit (WKWebView) | Safari |
| Windows | Chromium (WebView2) | Chrome/Edge |

**Practical risks for the Modeler**:
- The BPMN/DMN/Forms client renders complex SVG canvases. WebKit has historically
  had SVG rendering bugs that Blink/Chromium does not.
- CSS Grid/Flexbox edge cases differ between WebKit and Chromium.
- JavaScript engine differences: WebKit uses JavaScriptCore, not V8. Certain
  patterns (e.g., large typed arrays, specific Promise scheduling) may behave
  differently.
- The existing test suite was validated against Chromium. Additional testing
  against WebKit (Safari-equivalent) would be required.

**Mitigation**: Run the full CI test suite against the Tauri build on all three
platforms before shipping. Pay special attention to BPMN/DMN diagram rendering.

### Installation guidance gap

Currently, the modeler ships as a `.tar.gz` / `.zip` — unzip and run.
With Tauri on Linux:

- A `.deb`/`.rpm` install via package manager handles dependencies automatically.
- A raw binary or AppImage requires the user to have WebKit2GTK already, with no
  user-facing error if it's missing (process silently exits).
- **Action needed**: Either ship only deb/rpm packages (not raw binaries), or
  add a startup shell wrapper that checks for the dependency and prints a helpful
  message if missing.

### Summary: is it acceptable?

| Scenario | Assessment |
|----------|-----------|
| Developer / SaaS laptop (Ubuntu, macOS, Win 11) | ✅ No issues |
| Windows 10 enterprise, managed | ⚠️ WebView2 may be blocked by GPO |
| RHEL / CentOS minimal install | ⚠️ WebKit2GTK must be installed separately |
| Air-gapped installation | ❌ Requires pre-staging WebKit2GTK / WebView2 |
| macOS (any modern version) | ✅ No issues |

The Linux situation is the most likely to cause customer support tickets.
The Windows situation depends heavily on the enterprise customer's IT policy.

---

## Non-Blockers (Already Handled)

- File I/O (read/write) — fully implemented in Rust
- Dialogs (open/save/message) — fully implemented via `tauri-plugin-dialog`
- Config persistence — implemented as JSON file in app config dir
- Workspace save/restore — implemented via config layer
- Clipboard write — via `tauri-plugin-clipboard-manager`
- External URL open — via `tauri-plugin-shell`
- App reload / restart — via Tauri window and process APIs
- Quit flow (allowed/aborted) — via Tauri `AppHandle::exit`
- Error tracking — Sentry Rust SDK is a drop-in for `@sentry/node`
