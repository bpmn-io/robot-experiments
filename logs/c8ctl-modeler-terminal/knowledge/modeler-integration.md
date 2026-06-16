# Modeler integration: why in-tree, not a plugin

The Camunda Modeler renderer↔backend bridge enforces a **hardcoded allow-list**
in `app/lib/preload.js` (`allowedEvents`). `backend.send(event)` throws
`Disallowed event` for any channel not in that array. Plugins only expose
`style` / `script` / `menu` entry points and **cannot register new IPC
channels**, so a pure plugin cannot delegate command execution to the Electron
main process (where the `@camunda8/orchestration-cluster-api` SDK must run).

=> An isolated **in-tree module** under `app/lib/c8ctl/` plus minimal core
touch-points is the only viable approach for backend delegation.

## Core touch-points (the minimal surface to add a backend-delegating feature)

- `app/lib/index.js` — instantiate the module in `bootstrap()`, return it, and
  register `renderer.on('<ns>:<action>', async (options, done) => ...)` handlers.
  `done(null, result)` / `done(err)`. Handlers share closure scope with
  bootstrap-returned vars.
- `app/lib/preload.js` — add each `<ns>:*` channel to `allowedEvents`.
  `backend.send` is async/promise-based; only metadata/plugins/flags use the
  sync `sendSync` path. So even "getInfo"-style reads must be async `renderer.on`.
- `client/src/remote/<Name>.js` — thin renderer bridge over `backend.send`.
- `client/src/globals.js` — construct the bridge and add it to BOTH the
  `export const` list AND the `globals` object literal at the bottom (App reads
  via `getGlobal(name)`, which throws for unknown globals).
- `client/src/app/App.js` — mount the component. **Guard the mount** with
  `this.props.globals.<name> ? <Component/> : null` so the ~10 specs that build
  their own `globals` (AppSpec, AppParentSpec, IntegrationSpec,
  MigrateConnectionsSpec) don't crash on a missing global.

## Tests

- Backend: mocha `app/**/*-spec.js` via `npm run app:test`. Put specs in
  `app/lib/<module>/__tests__/*-spec.js`. Inject deps (fs dirs, SDK factory) for
  hermetic tests with temp dirs.
- Renderer: karma/mocha, file name `*Spec.js` under a `__tests__/` dir
  (`require.context(../src, true, /\/__tests__\/[A-Za-z0-9]+Spec\.js$/)`).
  Use `@testing-library/react` + `fireEvent`. Window-level key listeners are
  testable via `fireEvent.keyDown(window, { key: '`' })`.

## c8ctl config schema (as consumed by the module)

- `profiles.json` → `{ profiles: [ { name, baseUrl, clientId, ... } ] }`
  (an **array**, not a map). `activeProfile` is session-only (in-memory), never
  persisted — keeps `use profile` free of filesystem side-effects.
- Modeler `settings.json` → `connectionManagerPlugin.c8connections` is an array
  of connections; only entries with an `id` are kept, surfaced with a
  `modeler:` name prefix and `source: 'modeler'`. Self-hosted base URL comes
  from `contactPoint`; cloud from `camundaCloudClusterUrl`.
