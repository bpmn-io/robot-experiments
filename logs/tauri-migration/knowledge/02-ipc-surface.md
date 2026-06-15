# Electron IPC Surface Inventory

All IPC handlers registered in `app/lib/index.js` via the `renderer` utility.
The preload script (`app/lib/preload.js`) exposes these as `window.getAppPreload().backend.send(event, ...args)`.

## Sync handlers (ipcMain.on with returnValue)

| Event              | Purpose                              |
|--------------------|--------------------------------------|
| `app:get-metadata` | App version, locale, etc.            |
| `app:get-plugins`  | Loaded plugin descriptors            |
| `app:get-flags`    | Feature flags (CLI args)             |

## Async handlers (request/response pattern)

### File I/O
| Event             | Args                        | Returns          |
|-------------------|-----------------------------|------------------|
| `file:read`       | filePath, options           | FileDescriptor   |
| `file:read-stats` | file                        | FileDescriptor   |
| `file:write`      | filePath, file, options     | FileDescriptor   |

### Dialogs
| Event                    | Notes                         |
|--------------------------|-------------------------------|
| `dialog:open-files`      | Multi-file open picker        |
| `dialog:open-file-error` | Error dialog                  |
| `dialog:save-file`       | Save-as picker                |
| `dialog:show`            | Generic message dialog        |
| `dialog:open-file-explorer` | Open folder in file manager |

### Config
| Event        | Notes                     |
|--------------|---------------------------|
| `config:get` | Reads from electron-store |
| `config:set` | Writes to electron-store  |

### Workspace
| Event               | Notes                     |
|---------------------|---------------------------|
| `workspace:restore` | Restore window state      |
| `workspace:save`    | Save window state         |

### App lifecycle
| Event            | Notes                          |
|------------------|--------------------------------|
| `client:ready`   | Renderer signals ready         |
| `app:reload`     | Reload the window              |
| `app:restart`    | Relaunch the process           |
| `app:quit-allowed` / `app:quit-aborted` | Quit flow |

### System
| Event                       | Notes              |
|-----------------------------|--------------------|
| `external:open-url`         | Open browser       |
| `system-clipboard:write-text` | Write to clipboard |
| `context-menu:open`         | Show context menu  |

### File context (LSP-like tracking)
| Event                    | Notes                |
|--------------------------|----------------------|
| `file-context:add-root`  | Watch directory root |
| `file-context:remove-root` | Unwatch root       |
| `file-context:file-opened` | Track open file    |
| `file-context:file-updated` | Track file change |
| `file-context:file-closed` | Untrack file      |

### Zeebe API (gRPC via @camunda8/sdk)
| Event                              | Notes                  |
|------------------------------------|------------------------|
| `zeebe:checkConnection`            | Ping gateway           |
| `zeebe:deploy`                     | Deploy process/form/dmn |
| `zeebe:startInstance`              | Start process instance |
| `zeebe:getGatewayVersion`          | Gateway version        |
| `zeebe:searchProcessInstances`     | Operate API search     |
| `zeebe:searchElementInstances`     | Operate API search     |
| `zeebe:searchVariables`            | Operate API search     |
| `zeebe:searchIncidents`            | Operate API search     |
| `zeebe:searchJobs`                 | Operate API search     |
| `zeebe:searchMessageSubscriptions` | Operate API search     |
| `zeebe:searchUserTasks`            | Operate API search     |

### Menu
| Event           | Notes                         |
|-----------------|-------------------------------|
| `menu:register` | Register dynamic menu items   |
| `menu:update`   | Update menu state             |

### Error tracking
| Event                   | Notes              |
|-------------------------|--------------------|
| `errorTracking:turnedOn`  | Enable Sentry      |
| `errorTracking:turnedOff` | Disable Sentry     |

## Key architectural notes

- The bridge is established once per window via `window.getAppPreload()` (one-time call, prevents re-use exploits)
- All async events use a request ID suffix pattern: `event:response:<id>`
- The preload exposes `backend.send(event, ...args)` → Promise
- The preload exposes `backend.on(event, cb)` for server-push events

## Migration complexity by group

| Group        | Tauri feasibility | Notes                                               |
|--------------|-------------------|-----------------------------------------------------|
| File I/O     | Easy              | Tauri `fs` plugin handles this natively             |
| Dialogs      | Easy              | Tauri `dialog` plugin                               |
| Config       | Easy              | Tauri `store` plugin                                |
| Workspace    | Easy              | Store in Tauri app data dir                         |
| App lifecycle| Easy              | Tauri `app` and `process` plugins                   |
| System       | Easy              | Tauri `shell` and `clipboard` plugins               |
| Menu         | Medium            | Tauri supports native menus, but dynamic menus differ|
| File context | Medium            | Tauri `fs` watcher plugin                           |
| Zeebe API    | Hard              | `@camunda8/sdk` is Node.js gRPC — must port to Rust or run as sidecar |
| Plugins      | Hard              | No equivalent to Electron plugin loading model      |
| Error tracking | Easy            | Replace with Sentry Rust SDK                        |
