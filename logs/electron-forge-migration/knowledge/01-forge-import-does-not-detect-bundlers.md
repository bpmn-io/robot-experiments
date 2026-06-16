# Finding: electron-forge import Does Not Auto-Detect Existing Bundlers

## What We Tried

Running `npx @electron-forge/cli import` on the camunda-modeler repo (which uses
`electron-builder` + webpack).

## What Happened

`electron-forge import` generated a minimal `forge.config.js` **without** the
`@electron-forge/plugin-webpack` plugin. It only added:

- Generic makers: `maker-squirrel`, `maker-zip` (darwin only), `maker-deb`, `maker-rpm`
- `plugin-auto-unpack-natives`
- `plugin-fuses` (Electron security hardening)
- An `electron-squirrel-startup` runtime dependency
- Replaced `electron-builder` in devDependencies

It did **not**:
- Detect the existing `webpack.config.js` for the preload
- Detect the `client/webpack.config.js` for the renderer
- Generate any webpack plugin configuration

## Implication

"Generate existing bundlers" cannot be accomplished by `electron-forge import` alone.
The webpack plugin must be added and configured manually.

## Generated forge.config.js (baseline)

```js
module.exports = {
  packagerConfig: { asar: true },
  rebuildConfig: {},
  makers: [
    { name: '@electron-forge/maker-squirrel', config: {} },
    { name: '@electron-forge/maker-zip', platforms: ['darwin'] },
    { name: '@electron-forge/maker-deb', config: {} },
    { name: '@electron-forge/maker-rpm', config: {} },
  ],
  plugins: [
    { name: '@electron-forge/plugin-auto-unpack-natives', config: {} },
    new FusesPlugin({ ... }),
  ],
};
```
