# Experiment Summary: Migrate Desktop Modeler Build System to Electron Forge

## TL;DR

A working Electron Forge packaging pipeline was produced in ~4h. The migration is
**worth pursuing** as a straightforward swap of `electron-builder` for Forge makers,
keeping the existing webpack pipeline intact. Forge's webpack plugin is technically
possible but adds complexity with no practical benefit for this codebase.

---

## electron-builder vs. Electron Forge

| Aspect | electron-builder | Electron Forge |
|---|---|---|
| **Primary role** | Package + distribute | Package + distribute + (optionally) bundle |
| **Bundler support** | Agnostic — runs your build steps | Optional webpack/vite plugin takes over bundling |
| **Monorepo support** | Built-in (`npmArgs: --workspaces=false`) | Requires a `prePackage` hook workaround |
| **Makers/targets** | Rich built-in set (dmg, squirrel, deb, rpm, tar.gz, …) | Smaller set; no tar.gz maker out of the box |
| **Config** | JSON (`electron-builder.json`) | JS (`forge.config.js`) — more flexible, hooks included |
| **Hooks** | `afterPack`, `afterSign` | `prePackage`, `postPackage`, `generateAssets`, … |
| **Ecosystem** | Community-maintained | Electron team–maintained |
| **Dev workflow** | Separate (not part of electron-builder) | `electron-forge start` with webpack plugin; HMR |

The key conceptual difference: **electron-builder is purely a packager**. Forge can
optionally be a full toolchain (bundler + packager), but only if you use its webpack
or vite plugin. With Option B (Forge as packager only), the two tools are almost
equivalent in scope.

---

## What Was Proven

### Works well

- **`electron-forge package` / `electron-forge make`** produce a correctly named,
  correctly sized package (349 MB, down from 845 MB without the fix).
- **Full boot verified**: `received ready` → `received client-ready` → `file-context watcher:ready`.
- **`generateAssets` hook** cleanly integrates the existing webpack pipeline
  (preload bundle + client build) before each package run.
- **`prune` fix** via `packagerConfig.dir = './app'` + `prePackage` npm install:
  `app/package.json` is exactly the right source of truth for production deps.
- **`@electron-forge/plugin-fuses`** (Electron security hardening) added for free.
- **JS config** (`forge.config.js`) is more expressive than `electron-builder.json`;
  hooks run inline instead of requiring separate task scripts.

### Does not work out of the box

- **`electron-forge import`** does not auto-detect existing webpack bundlers — it
  generates a generic config with no webpack plugin, contrary to the original hypothesis.
- **No tar.gz maker** — Linux currently falls back to `.deb`. A custom maker or
  `postMake` hook is needed to restore the current `.tar.gz` target.
- **afterPack / afterSign hooks** from electron-builder are not yet ported
  (`add-platform-files`, `add-version`, `set-permissions`, notarization).

---

## Remaining Caveats

### Must-fix before shipping

| Caveat | Effort |
|---|---|
| Port `afterPack` handlers (`add-platform-files`, `add-version`, `set-permissions`) to Forge `postPackage` hook | Small–Medium |
| Port `afterSign` (macOS notarization) to Forge `postPackage` hook | Small |
| Restore Linux `.tar.gz` target (custom maker or `postMake` repackage) | Small |
| Add `@electron-forge/maker-dmg` for macOS `.dmg` target | Trivial |
| Wire up file associations (`.bpmn`, `.dmn`, `.form`, `.rpa`) in `packagerConfig` | Small |
| Wire up macOS `hardenedRuntime` + entitlements plist in `packagerConfig.osxSign` | Small |
| `app/node_modules/` grows after each `forge package` run (not cleaned up) | Small |

### Nice-to-have

| Caveat | Effort |
|---|---|
| `electron-forge start` does a full webpack build each time (slow dev loop) | Medium — needs a watch-mode integration |
| Lerna version-bump logic from `tasks/distro.js` not ported | Medium |
| Custom publish step (S3 upload via `electron-publisher-s3`) not ported | Medium |

---

## Is the Migration Worth It?

**Yes — with the right scope.**

The case for migrating:

- **Forge is Electron-team maintained**; electron-builder is community-maintained and
  has had stretches of slow maintenance. Long-term reliability favors Forge.
- **JS hooks** are strictly more powerful than electron-builder's JSON config + external
  task files. The `prePackage`/`postPackage` hooks replace `tasks/after-pack.js`,
  `tasks/distro.js`, etc. with inline, type-safe logic.
- **`plugin-fuses`** is a free security improvement not available in electron-builder.
- **The migration is incremental** — Option B (Forge as packager only) requires no
  changes to the bundler, the app code, or the dev workflow. The diff is ~100 lines
  confined to `forge.config.js`.

The case against migrating (or deferring):

- **No tar.gz out of the box** is a regression for Linux users; requires extra work.
- **afterPack porting** is non-trivial to test without real Mac/Win builds.
- **Not zero risk** — the existing electron-builder pipeline is battle-tested and
  well-understood; any packaging regression would be caught late (CI / release).

**Recommendation**: migrate, scoped to Option B. The must-fix list above is 1–2 days
of focused work. The dev workflow remains unchanged. The biggest risk is the afterPack/
afterSign porting — that should be the focus of any follow-up validation pass.

---

## Collateral

- **Experiment branch**: `experiment/electron-forge-migration` in `camunda-modeler`
- **Working directory**: `experiments-repositories/electron-forge-migration/`
- **Key files changed**: `forge.config.js` (+115 lines), `package.json` (deps swap)
- **Knowledge**: `logs/electron-forge-migration/knowledge/` (4 documents)
