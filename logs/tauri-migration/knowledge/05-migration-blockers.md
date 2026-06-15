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
