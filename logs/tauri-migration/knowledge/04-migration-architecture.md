# Tauri Migration Architecture

## What Was Built

A working Tauri 2.x shell for the Desktop Modeler in ~500 lines of Rust + a JS bridge,
reusing the existing unmodified React/webpack client bundle.

### Key Files

| File | Role |
|------|------|
| `src-tauri/src/lib.rs` | Rust backend: all IPC commands, app setup |
| `src-tauri/src/bridge.js` | JS initialization script injected before page load |
| `src-tauri/tauri.conf.json` | App config: identifier, window, bundle targets |
| `src-tauri/capabilities/default.json` | Permission grants for dialog, shell, clipboard |

## How the IPC Bridge Works

```
Electron (old)                    Tauri (new)
─────────────────                 ──────────────────────────
preload.js                        bridge.js (init script)
  contextBridge.exposeInMainWorld   window.getAppPreload = ...
  window.getAppPreload              window.__APP_METADATA__ = { version, name }

backend.send(event, ...args)      send(event, ...args)
→ ipcRenderer.send(event, id, args)  → invoke(command, { ...args })
→ ipcMain.on(event, handler)      → #[tauri::command] fn command(...)

backend.on(event, callback)       on(event, callback)
→ ipcRenderer.on(event, callback)  → listen(event, callback)
→ mainWindow.webContents.send()   → app.emit(event, payload)
```

The client (`globals.js`) calls `window.getAppPreload()` unchanged.
The bridge intercepts this and maps each IPC event to a Tauri `invoke()`.

## IPC Event → Tauri Command Mapping

All ~30 IPC events were mapped. The switch in `bridge.js` dispatches by event name:

```javascript
case 'file:read':    invoke('file_read',   { path, options })
case 'config:get':  invoke('config_get',  { key })
case 'dialog:open-files': invoke('dialog_open_files', { options })
// ...
```

## Initialization Script vs. Electron Preload

Electron's `preload.js` runs in an isolated context before page JS. Tauri's
`WebviewWindowBuilder::initialization_script()` provides the same guarantee: it
runs before HTML parsing, so `window.getAppPreload` is available when `globals.js`
calls it during module load.

**Important**: The `index.html` has `<meta http-equiv="Content-Security-Policy"
content="script-src 'self'">`. Tauri initialization scripts bypass the page CSP
because they run at the webview engine level, before HTML is parsed.

## Frontend Asset Embedding

Tauri embeds the frontend at compile time:
```
frontendDist: "../../../../camunda-modeler/app/public"
```

All files in `app/public/` are read at compile time and compressed into the binary.
At runtime, they are served via the `tauri://localhost` protocol.
This is why the deb bundle has only one file (the binary) — no separate assets dir.

## Config / Workspace Persistence

Electron used `electron-store` (JSON file in `$HOME/.config/camunda-modeler/`).
The Tauri implementation uses the same approach:
- `AppConfig` is a `Mutex<Map<String, Value>>`
- Loaded from `$HOME/.config/com.camunda.CamundaModeler/config.json` on startup
- Written to disk on every `config:set` and `workspace:save`

## What Was Stubbed (Out of Scope)

These IPC handlers return no-op or stub values:

| Feature | Reason |
|---------|--------|
| Zeebe API | `@camunda8/sdk` uses Node.js gRPC — no direct Rust equivalent |
| Plugin loading | Electron plugins are arbitrary JS/CSS files loaded at runtime |
| Menu (dynamic) | Implemented as stubs; native Tauri menus need rework |
| File context / LSP | Stub; actual watcher would use `notify` crate |
| Error tracking (Sentry) | Stub; Sentry Rust SDK is a drop-in replacement |
| `context-menu:open` | Stub; Tauri has a context menu API but needs separate work |

## Platform Notes

- **Linux**: Uses `WebKit2GTK 4.1`, available in all major distros since ~2022
- **macOS**: Uses `WKWebView`, part of the OS since macOS 10.10
- **Windows**: Uses `WebView2`, shipped with Windows 11, auto-installed on Win10
