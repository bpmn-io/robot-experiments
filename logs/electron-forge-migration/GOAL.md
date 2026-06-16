# Experiment Goal: Migrate Desktop Modeler Build System to Electron Forge

## Hypothesis

Replacing `electron-builder` with Electron Forge as the packaging and distribution
toolchain for the Camunda Modeler will simplify the build pipeline, enable better
integration with the Electron ecosystem tooling, and continue to work with the
existing webpack bundlers (preload + client) without requiring a rewrite.

## Success Criteria

1. Electron Forge is configured as the primary build/package/make tool.
2. The existing webpack bundlers (preload via `webpack.config.js`, client via its
   own webpack config) are reused via `@electron-forge/plugin-webpack` or equivalent.
3. `npm run make` (Forge) produces the same platform artefacts as `electron-builder`
   currently does: `.zip` on Windows/macOS, `.tar.gz` on Linux, `.dmg` on macOS.
4. `npm start` (Forge dev mode) launches the app with hot-reload for the renderer.

## Approach

- Run `electron-forge import` on a worktree of the camunda-modeler to generate
  initial Forge configuration and adopt makers equivalent to the current
  `electron-builder.json` targets.
- Integrate the existing webpack configurations via the Forge webpack plugin
  rather than replacing them.
- Assess what custom `afterPack`/`afterSign` hooks and lerna version-bump logic
  from `tasks/distro.js` need to be ported.

## Out of Scope

- Replacing or rewriting the webpack configurations themselves.
- CI pipeline changes beyond local verification.
- Code signing / notarization setup (macOS afterSign hook).
- Production readiness — this is a feasibility spike.

## Key Questions

- Does `electron-forge import` successfully detect and configure the existing
  webpack bundlers?
- Which electron-builder targets have direct Forge maker equivalents?
- What custom build logic in `tasks/distro.js` / `tasks/after-pack.js` cannot
  be expressed in Forge hooks?
- Does the dev workflow (`npm start`) work smoothly with Forge + existing webpack?
