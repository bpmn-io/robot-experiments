# Electron Forge: npm Workspace Monorepo Prune Fix

## Problem

In an npm workspaces monorepo, production dependencies of a sub-package (e.g. `app/`)
are hoisted to root `node_modules/`. When `electron-forge package` runs with default
settings (`prune: true`), `@electron/packager` executes `npm prune --omit=dev` against
the root `package.json`. If root only lists build tooling in `dependencies`, all hoisted
app deps are pruned — the packaged binary crashes with `Cannot find module`.

## Fix

1. Point `packagerConfig.dir` at the sub-package directory (`./app`).
   electron-packager then prunes based on that package's `package.json` — the correct dep list.

2. In a `prePackage` hook, run `npm install --workspaces=false` in that directory
   to materialise all production deps into its local `node_modules/`
   (instead of hoisted root `node_modules/`):

```js
hooks: {
  prePackage: async () => {
    await execa('npm', [ 'install', '--workspaces=false' ], {
      cwd: path.resolve(__dirname, 'app'),
      stdio: 'inherit',
    });
  },
}
```

3. If app code looks for resources inside the asar (`app.getAppPath() + '/resources'`),
   also merge any root-level resources into the sub-package dir in `prePackage` and
   clean them up in `postPackage`.

## Trade-off

`app/node_modules/` accumulates production deps after each package run (not cleaned up).
This does not break the dev workflow since root `node_modules/` still takes precedence
via workspace resolution, but it wastes disk space. A `postPackage` cleanup (`rm -rf
app/node_modules && npm install` from root) restores the workspace, but adds ~30s.

## Comparison: electron-builder

electron-builder handles this via `"npmArgs": "--workspaces=false"` in
`electron-builder.json`, which re-runs npm at package time with workspace hoisting
disabled. Forge has no direct equivalent setting — the hook workaround is necessary.
