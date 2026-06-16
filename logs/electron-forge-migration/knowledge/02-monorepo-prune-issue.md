# Finding: Monorepo Workspace Dependency Pruning Issue

## Problem

With the default `packagerConfig.prune: true` (the default), `electron-forge package`
produces a broken app — production dependencies that are **workspace-hoisted** to the
root `node_modules/` are pruned out.

### Why This Happens

The camunda-modeler is an npm workspace monorepo with `app` and `client` sub-packages.
npm hoists production deps (e.g. `@sentry/node`, `@camunda8/sdk`, etc.) from
`app/package.json` to the root `node_modules/`. When Forge's packager prunes, it
uses the root `package.json`'s dependency graph. Since these hoisted packages are not
listed in root `package.json`'s `dependencies`, they get pruned.

### Symptom

```
Error: Cannot find module '@sentry/node'
Require stack:
- app.asar/app/lib/index.js
```

The ASAR contains `/node_modules/@sentry` as an **empty directory** — the directory
entry is there but no files.

## Quick Workaround

```js
packagerConfig: {
  asar: true,
  prune: false,
},
```

This works (app boots) but **balloons the package size**:
- With `prune: true` (default, broken): 171 MB
- With `prune: false` (working): **845 MB** — includes all devDependencies

## Proper Fix Needed

The packager needs workspace-aware pruning. Options:
1. **`prePackage` hook**: Run `npm install --workspaces=false` in a temp copy of `app/`
   to flatten workspace deps into `app/node_modules/`, then set `packagerConfig.dir = './app'`
2. **Custom `ignore` function**: Walk the combined dep tree of all workspaces and prune
   only true devDependencies
3. **Forge workspace plugin**: May be addressed in newer Forge releases

## Comparison with electron-builder

electron-builder handled this via `"npmArgs": "--workspaces=false"` in
`electron-builder.json`, which re-ran npm install without workspaces before packaging.
Forge does not have a direct equivalent config option.
