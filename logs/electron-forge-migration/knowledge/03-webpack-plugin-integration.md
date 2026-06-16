# Finding: Integrating @electron-forge/plugin-webpack with Existing Bundlers

## Overview

The webpack plugin CAN be made to work with the existing webpack configurations,
but requires several non-trivial adaptations. It does NOT "drop in" to the existing
setup without changes.

## Required Changes

### 1. Main Process: Minimal webpack config (new file)

The main process (`app/lib/index.js`) is not webpack-bundled in the original setup.
The webpack plugin requires a `mainConfig`. A minimal passthrough was created at
`webpack.main.config.js`:

```js
module.exports = {
  entry: './app/prod.js',
  target: 'electron-main',
  resolve: { mainFields: ['main'] },
  externals: {
    '@sentry/node': 'commonjs @sentry/node',
    '@camunda8/sdk': 'commonjs @camunda8/sdk',
    'electron': 'commonjs electron',
  },
};
```

The `externals` are critical — without them, webpack tries to bundle native modules
and large SDKs, causing build errors or huge bundles.

### 2. Renderer: Wrapper config with explicit `context` (new file)

The client's `webpack.config.js` uses relative entry paths (`./src/index.js`,
`./public`) that resolve against webpack's `context`, which defaults to `process.cwd()`
(the repo root). Setting explicit `context` fixes this:

```js
// webpack.renderer.config.js
const clientConfig = require('./client/webpack.config.js');
const clientDir = path.resolve(__dirname, 'client');

module.exports = {
  ...clientConfig,
  context: clientDir,
  output: { ...clientConfig.output, path: path.resolve(__dirname, '.webpack/renderer') },
};
```

### 3. Root Babel config (new file)

Babel's `.babelrc` in `client/` is a package-relative config. When webpack runs from
the root, it won't pick up a nested `.babelrc` unless `babelrcRoots` is configured.
A root `babel.config.js` with the same presets fixes this:

```js
module.exports = {
  presets: [
    ['@babel/preset-env', { targets: { electron: '34' } }],
    '@babel/preset-react',
  ],
};
```

### 4. forge.config.js: entryPoints relative to renderer context

When `context: clientDir` is set in the renderer config, all paths in
`entryPoints[]` are resolved relative to `clientDir`. Paths must match:

```js
entryPoints: [{
  name: 'main_window',
  html: './public/index.html',   // relative to client/
  js: './src/index.js',          // relative to client/
  preload: {
    js: path.resolve(__dirname, 'app/lib/preload.js'),  // must be absolute
  },
}],
```

### 5. App code: Use MAIN_WINDOW_WEBPACK_ENTRY (app/lib/index.js change)

The webpack plugin injects `MAIN_WINDOW_WEBPACK_ENTRY` as a webpack `DefinePlugin`
constant (resolved to `http://localhost:3000/main_window/index.html` in dev,
and a `file://` path in production). The app must use this instead of constructing
the renderer URL manually:

```js
// Before:
let url = 'file://' + path.resolve(__dirname + '/../public/index.html');
if (process.env.NODE_ENV === 'development') {
  url = 'file://' + path.resolve(__dirname + '/../../client/build/index.html');
}

// After:
const url = typeof MAIN_WINDOW_WEBPACK_ENTRY !== 'undefined'
  ? MAIN_WINDOW_WEBPACK_ENTRY
  : 'file://' + path.resolve(__dirname + '/../public/index.html');
```

## Verified Outcomes

After the above changes:
- ✅ `electron-forge start` compiles both main and renderer webpack bundles
- ✅ The webpack dev server starts on port 3000
- ✅ The renderer is served at `http://localhost:3000/main_window/index.html`
  (verified: HTTP 200, correct Camunda Modeler HTML)
- ✅ The Electron main process boots (`starting Camunda Modeler v5.48.0`, `received ready`)
- ✅ Hot reload infrastructure is in place (HMR via webpack dev server)

## Still Open

- The full UI render in the Electron window was not visually confirmed (headless env)
- `MAIN_WINDOW_PRELOAD_WEBPACK_ENTRY` is not yet wired up in app code
- The app code changes need to be backward-compatible (non-forge builds must still work)
