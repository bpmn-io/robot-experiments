# JavaScript Codebase Impact

The Desktop Modeler's main-process is ~7,150 lines of JavaScript across 65 modules,
backed by ~10,000 lines of tests. A Tauri migration **replaces this entirely with Rust**.
This is not a port — it is a full rewrite of the app layer.

## Codebase at a Glance

| Subsystem         | Lines (JS) | Test lines | Risk  |
|-------------------|-----------|------------|-------|
| `zeebe-api/`      | 1,659     | 4,918      | 🔴 Critical |
| `file-context/`   | 1,394     | 752        | 🔴 High     |
| `menu/`           | 1,197     | 546        | 🔴 High     |
| `index.js`        | 944       | —          | 🔴 High     |
| `template-updater/` | 379     | 651        | 🟡 Medium   |
| `config/`         | 453       | 544        | 🟡 Medium   |
| `util/`           | 805       | 680        | 🟡 Medium   |
| `platform/`       | 184       | —          | 🟡 Medium   |
| `window-manager/` | 146       | 210        | 🟡 Medium   |
| `plugins/`        | 169       | 177        | 🟡 Medium   |
| `file-system/`    | 204       | 389        | 🟢 Low      |
| `dialog/`         | 198       | 562        | 🟢 Low      |
| `log/`            | 213       | 60         | 🟢 Low      |
| `cli/` + `flags/` | 184       | 271        | 🟢 Low      |
| **Total**         | **~7,150** | **~10,000** | |

## Subsystems That Cannot Be Lifted-and-Shifted

### zeebe-api/ (1,659 lines, 4,918 test lines)

The most complex and most tested subsystem. It:
- Dynamically switches between gRPC (`@camunda8/sdk`) and REST based on endpoint config
- Handles TLS certificate extraction on three platforms via child_process (Windows:
  Crypt32 DLL, macOS: `security` CLI, Linux: standard cert paths) plus optional
  `vscode-windows-ca-certs` native addon on Windows
- Maps gRPC error codes and HTTP status codes to user-facing error reasons
- Deploys resources as binary or text depending on type (BPMN/DMN/Form)
- Caches the Camunda8 SDK client per endpoint config

There is no Rust port of `@camunda8/sdk`. A Rust implementation would require either
`tonic` (gRPC) or `reqwest` (REST). Switching to REST-only is the practical path;
the gRPC + cert + error-mapping logic still takes meaningful effort to replicate.

### file-context/ (1,394 lines, 752 test lines)

Real-time file watching and in-memory metadata indexing:
- Wraps `chokidar` for filesystem watching (multiple roots, debounced atomic ops,
  node_modules/.git exclusion, symlinks)
- In-memory Map of indexed files with add/update/remove events to the renderer
- XML parsing of BPMN/DMN/Form/RPA files using `saxen` (streaming SAX parser)
  and `bpmn-moddle`/`dmn-moddle` for semantic extraction (process IDs, decision
  names, signal/message definitions, user tasks, form keys)
- Work queue for concurrent file processing with configurable parallelism

Rust equivalents exist (`notify` crate for watching, `quick-xml`/`roxmltree` for
parsing) but `chokidar` is battle-hardened; feature parity will surface edge cases.
The moddle-based BPMN/DMN semantic extraction in particular has no clean Rust
equivalent — it would need to be reimplemented from scratch using the XML schema.

### menu/ (1,197 lines, 546 test lines)

~30 dynamically managed native menu items rebuilt on every state change. Includes:
- macOS-specific menu structure (app name as first item, Preferences placement)
- State-driven visibility/enabled: depends on open tabs, unsaved files, dev mode
- Dynamic context menus (right-click on tabs)
- Platform-aware keyboard shortcuts (Ctrl vs. Cmd)

Tauri v2 has mutable native menus, but the state-change → full rebuild pattern
needs to be replicated in Rust. The macOS semantics (Help menu for NSApp, Services
submenu) are documented but non-trivial to get right.

## Tooling That Disappears

The entire JS toolchain for the main process is discarded:

| Tool | Role | Rust replacement |
|------|------|-----------------|
| **Mocha** | Test runner | `cargo test` |
| **Chai** + chai-subset | Assertions | Built-in `assert!` / `assert_eq!` |
| **Sinon** | Mocking/stubbing | `mockall` crate (significantly harder) |
| **nyc / Istanbul** | Coverage | `cargo-llvm-cov` |
| **proxyquire** | Module-level mocking | No direct equivalent |
| **ESLint** (bpmn-io plugin) | Linting | `cargo clippy` |
| **webpack** (preload bundle) | Preload bundling | Eliminated (init script is plain JS) |

The test infrastructure difference is significant:
- **10,000 lines of Mocha/Sinon tests** cannot be carried forward. All must be
  rewritten as Rust tests (unit) or as integration tests via Tauri's test harness.
- Sinon's module-level stubbing (used heavily in zeebe-api and menu tests) has no
  direct analogue in Rust. Mocking in Rust requires traits + dependency injection,
  meaning the code structure must also change to enable testability.
- JS tests can exercise the full module graph in a single Node.js process with cheap
  stubs. Rust integration tests for Tauri commands require a running application
  window — test setup is heavier.

## Developer Experience Impact

The main-process JavaScript is owned and maintained by JavaScript developers. A
Tauri migration requires:
- **Rust proficiency** for all future main-process work (IPC commands, OS integration,
  file watching, menu updates). Rust has a steep learning curve.
- **Cognitive context switching**: client-side work stays in JavaScript/React; backend
  work switches to Rust. The codebase becomes a polyglot repo.
- **Debugging tooling changes**: Chrome DevTools for the renderer still works; gdb/lldb
  or `tokio-console` replaces Node.js Inspector for the main process.
- **Slower iteration loops**: Rust compile times (even incremental) are meaningfully
  slower than Node.js module reloads.

## What Stays Unchanged

The React/webpack client (`client/`) is entirely unaffected. All BPMN/DMN/Forms
modelling logic, the properties panel, the deployment UI — none of that changes.
The IPC surface (event names and argument shapes) is also preserved, so the
client code requires no changes.

## Quantified Rewrite Scope

| Scope | Estimate |
|-------|---------|
| JS code replaced by Rust | ~7,150 lines → ~3,500–5,000 lines Rust |
| Tests to rewrite | ~10,000 lines → ~4,000–6,000 lines Rust (lower: Rust tests are more concise for happy paths, but mocking is harder) |
| Elapsed dev time (experienced Rust dev) | 6–10 weeks |
| Elapsed dev time (team learning Rust) | 16–24 weeks |

The 40-line Rust IPC stub from this experiment covers the happy path. The production
rewrite must match the test coverage and edge-case handling of the existing JS.
