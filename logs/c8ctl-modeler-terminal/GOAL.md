# Goal: Quake-style c8ctl terminal in Camunda Desktop Modeler

## What we are trying to prove

That the **declarative functional command architecture** of [`c8ctl`](https://github.com/camunda/c8ctl)
can be cleanly transplanted into the Camunda Desktop Modeler as an **isolated in-tree
module**, exposing a Quake-style pop-over terminal that executes Camunda 8 orchestration
commands against the user's existing connection profiles — without forking the CLI or
adding new external runtime surfaces beyond what the Modeler already ships.

## Concrete deliverable

A feature in `~/workspace/camunda-modeler` (branch `feat/c8ctl-doom`) where:

- Pressing the backtick key (`` ` ``) toggles a Quake-style pop-over terminal in the renderer.
- The terminal prompt is `c8ctl >`.
- Commands available are those from the c8ctl command-metadata registry whose **only**
  side-effect is an **orchestration-cluster-api REST call** (no filesystem side-effects),
  plus the **one exception: `profile` read** (listing/inspecting connection profiles).
- Commands execute on the **Electron main process (backend)** using the
  `@camunda8/orchestration-cluster-api` library — the same library c8ctl uses.
- The backend resolves connection profiles from **both** the Modeler configuration
  (`settings.json` → `connectionManagerPlugin.c8connections`) **and** the c8ctl
  configuration file on disk (`profiles.json`), if present.
- Command output is streamed back to the terminal UI.

## Constraints / design rules

- **Copy c8ctl's declarative functional architecture**: a single command-metadata registry
  drives validation, dispatch, help, and completion; handlers are pure functions returning
  a `CommandResult` discriminated union; rendering and error-handling are centralised.
- **Cleanly isolated module**: all new backend code under `app/lib/c8ctl/`, all new
  renderer code under its own directory; core Modeler files touched only minimally
  (IPC registration, preload allow-list, one mount point, globals wiring).
- **REST-only**: exclude every c8ctl command with filesystem / shell / subprocess
  side-effects (deploy, run, watch, open, plugins, mcp-proxy, completion, session
  persistence, profile *writes*). Keep `profile` *read* as the sole filesystem exception.

## Non-goals

- Re-implementing c8ctl's plugin system, MCP proxy, deploy/run/watch, or shell completion.
- Persisting session state (active profile/tenant) to disk from within the Modeler.
