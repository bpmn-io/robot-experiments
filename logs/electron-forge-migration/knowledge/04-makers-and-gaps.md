# Finding: Electron Forge Makers vs electron-builder Targets

## Current electron-builder Targets

From `electron-builder.json`:

| Platform | Target    | Arch          |
|----------|-----------|---------------|
| Windows  | zip       | x64, ia32     |
| Linux    | tar.gz    | x64           |
| macOS    | dmg       | x64, arm64    |
| macOS    | zip       | x64, arm64    |

## Electron Forge Maker Equivalents

| electron-builder target | Forge maker                           | Notes                                   |
|------------------------|---------------------------------------|-----------------------------------------|
| zip (Win/mac)           | `@electron-forge/maker-zip`           | Platform-agnostic                       |
| tar.gz (Linux)          | `@electron-forge/maker-zip` + rename? | No native tar.gz maker — needs wrapper  |
| dmg (macOS)             | `@electron-forge/maker-dmg`           | Requires `electron-installer-dmg`       |
| squirrel (Win)          | `@electron-forge/maker-squirrel`      | Different install UX than plain zip     |

### Key Gap: No tar.gz Maker

Forge has no built-in tar.gz maker. Options:
1. Use zip for Linux (change from current behavior)
2. Write a custom maker that calls `tar`
3. Use `@electron-forge/maker-zip` + a `postMake` hook to repackage as tar.gz

### Key Gap: DMG Maker

The default forge import does NOT add `@electron-forge/maker-dmg`. It must be
installed separately: `npm install --save-dev @electron-forge/maker-dmg`

## Configuration Gaps vs electron-builder.json

| electron-builder.json feature          | Forge equivalent                          |
|----------------------------------------|-------------------------------------------|
| `appId: com.camunda.CamundaModeler`    | `packagerConfig.appBundleId`              |
| `productName: Camunda Modeler`         | `packagerConfig.name`                     |
| `fileAssociations` (.bpmn, .dmn, etc.) | `packagerConfig.protocols` / maker config |
| `afterPack: ./tasks/after-pack.js`     | `hooks.postPackage`                       |
| `afterSign: ./tasks/after-sign.mjs`    | `hooks.postPackage` (after signing)       |
| `npmArgs: --workspaces=false`          | No direct equivalent → custom prePackage hook |
| `linux.category: Development`          | `makers.config.categories` in maker-deb  |
| `hardenedRuntime: true` (macOS)        | `packagerConfig.osxSign.hardenedRuntime`  |
| `entitlements: .../entitlements.plist` | `packagerConfig.osxSign.entitlements`     |

## Root Package Name Issue

The generated app binary is named `camunda-modeler-builder` (from root
`package.json` name) instead of `Camunda Modeler`. Fix:

```js
packagerConfig: {
  name: 'Camunda Modeler',
  appBundleId: 'com.camunda.CamundaModeler',
  // ...
}
```
