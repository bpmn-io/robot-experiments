# Experiment Goal: Migrate Desktop Modeler to Tauri

## Hypothesis

Replacing Electron (~42.x) with Tauri as the application shell for the Camunda Modeler
will significantly reduce the distributed bundle size, because Tauri uses the OS-native
webview instead of bundling Chromium.

## Success Criteria

1. The app boots and renders the modeler UI via Tauri.
2. Core IPC features work: file open/save, dialog, config, workspace restore/save.
3. Bundle sizes (Linux tar.gz, macOS dmg/zip, Windows zip) are measured and compared
   before and after the migration.

## Out of Scope

- Feature parity for every Electron IPC event (Zeebe API, plugins, error tracking, etc.).
- Production readiness or code quality — this is a feasibility spike.
- Windows-specific certificate store (`vscode-windows-ca-certs`) integration.

## Key Questions

- How large is the current Electron bundle vs. a Tauri bundle?
- How much of the Electron IPC surface maps cleanly to Tauri commands/events?
- What are the blockers for a full migration (e.g. native modules, plugin API)?
