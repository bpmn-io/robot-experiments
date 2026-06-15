---
name: bpmn-io-discovery
updated_at: 2026-05-20
description: Maps bpmn.io and Camunda Modeling repositories to find where a feature lives, trace a dependency chain, or decide where new functionality should go. Use when looking up which repo owns a package, understanding how a library fits the ecosystem, or tracing a chain like `feel-editor` → `properties-panel` → `bpmn-js-properties-panel`.
---

# bpmn.io Repository Discovery

Understand how the bpmn.io and Camunda modeling eco-system composes.

## Quick orientation

The ecosystem is built in layers: shared primitives → meta-modeling → diagram infrastructure → domain modelers → panels/extensions → Modeler application. Camunda repos plug into the same layers (moddle extensions in Layer 2, distributions in Layer 4, linting in Layer 7) rather than forming a separate stack. Everything uses dependency injection via `didi`. The shared utility layer (`min-dash`, `min-dom`, `tiny-svg`) appears in virtually every package.

## Programmatic lookup

Filter `repositories.json` (same directory) by `layer`, `repo`, or `npm`.

---

## Layer 1 — Primitives (no bpmn-io deps)

| Repo                        | npm                      | Purpose                                                   |
| :-------------------------- | :----------------------- | :-------------------------------------------------------- |
| `bpmn-io/min-dash`          | `min-dash`               | Minimal lodash-like utility belt                          |
| `bpmn-io/min-dom`           | `min-dom`                | Minimal DOM utilities                                     |
| `bpmn-io/tiny-svg`          | `tiny-svg`               | Thin SVG element creation/manipulation                    |
| `bpmn-io/ids`               | `ids`                    | Unique ID generation                                      |
| `bpmn-io/object-refs`       | `object-refs`            | Bidirectional object references (memory-efficient graphs) |
| `bpmn-io/path-intersection` | `path-intersection`      | SVG path intersection math                                |
| `bpmn-io/diagram-js-ui`     | `@bpmn-io/diagram-js-ui` | Preact-based UI components for diagram-js                 |
| `bpmn-io/a11y`              | `@bpmn-io/a11y`          | Accessibility utilities for diagram-js-based editors      |
| `bpmn-io/draggle`           | `@bpmn-io/draggle`       | Drag and drop primitive used by diagram-js                |
| `bpmn-io/bpmn-font`         | `bpmn-font`              | BPMN 2.0 icon font (SVG icons as a web font)              |

External dep worth knowing: `didi` — IoC/DI container used throughout.

---

## Layer 2 — Meta-Modeling / XML (moddle stack)

Moddle is a meta-model system: you describe a schema (e.g. BPMN 2.0 XSD) as JSON descriptors and get typed JS objects + serialization for free.

| Repo                                    | npm                                   | Purpose                                                                       |
| :-------------------------------------- | :------------------------------------ | :---------------------------------------------------------------------------- |
| `bpmn-io/moddle`                        | `moddle`                              | Core meta-model (typed JS objects from descriptors)                           |
| `bpmn-io/moddle-xml`                    | `moddle-xml`                          | XML read/write for moddle-described schemas                                   |
| `bpmn-io/moddle-utils`                  | `@bpmn-io/moddle-utils`               | Helpers (traversal, lookup)                                                   |
| `bpmn-io/bpmn-moddle`                   | `bpmn-moddle`                         | BPMN 2.0 meta-model wrapping moddle + moddle-xml                              |
| `bpmn-io/dmn-moddle`                    | `dmn-moddle`                          | DMN 1.3 meta-model                                                            |
| `camunda/camunda-bpmn-moddle`           | `camunda-bpmn-moddle`                 | Camunda 7 moddle descriptor extensions for BPMN 2.0                           |
| `camunda/camunda-dmn-moddle`            | `camunda-dmn-moddle`                  | Camunda 7 moddle descriptor extensions for DMN                                |
| `camunda/zeebe-bpmn-moddle`             | `zeebe-bpmn-moddle`                   | Zeebe / Camunda 8 moddle descriptor extensions for BPMN 2.0                   |
| `camunda/modeler-moddle`                | `modeler-moddle`                      | Camunda Modeler namespace extensions for BPMN 2.0 (modeler-specific metadata) |
| `camunda/element-templates-json-schema` | `element-templates-json-schema-build` | JSON Schema definitions for Camunda element templates (C7 + C8/Zeebe)         |
| `camunda/execution-platform`            | `@camunda/execution-platform`         | Shared execution platform metadata (engine version constants, etc.)           |

__Key connection__: `bpmn-js` depends on `bpmn-moddle` for parsing/serializing `.bpmn` files.

---

## Layer 3 — Diagram Infrastructure

| Repo                                | npm                         | Purpose                                                                       |
| :---------------------------------- | :-------------------------- | :---------------------------------------------------------------------------- |
| `bpmn-io/diagram-js`                | `diagram-js`                | Core SVG diagram toolkit: canvas, elements, interaction, layouting, DI wiring |
| `bpmn-io/diagram-js-direct-editing` | `diagram-js-direct-editing` | Inline label editing (peer dep on diagram-js)                                 |
| `bpmn-io/diagram-js-minimap`        | `diagram-js-minimap`        | Minimap overlay plugin                                                        |
| `bpmn-io/diagram-js-grid`           | `diagram-js-grid`           | Background grid plugin                                                        |
| `bpmn-io/diagram-js-origin`         | `diagram-js-origin`         | Origin marker plugin                                                          |
| `bpmn-io/table-js`                  | `table-js`                  | Table editing toolkit (same DI approach, used by dmn-js decision tables)      |

`diagram-js` is the foundation for `bpmn-js` and the DRD part of `dmn-js`.

---

## Layer 4 — Domain Modelers

| Repo                      | npm                | Purpose                                                                                                                                                                                         |
| :------------------------ | :----------------- | :---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `bpmn-io/bpmn-js`         | `bpmn-js`          | BPMN 2.0 viewer + modeler; uses `diagram-js` + `bpmn-moddle`; extended via `additionalModules` / `moddleExtensions`                                                                             |
| `bpmn-io/dmn-js`          | `dmn-js`           | DMN 1.3 viewer/modeler (monorepo: `dmn-js-drd`, `dmn-js-decision-table`, `dmn-js-boxed-expression`, `dmn-js-literal-expression`, `dmn-js-shared`); DRD uses `diagram-js`, tables use `table-js` |
| `bpmn-io/form-js`         | `@bpmn-io/form-js` | JSON Schema-driven form viewer + visual editor (monorepo: `form-js-viewer`, `form-js-editor`, `form-js-carbon-styles`); independent of diagram-js                                               |
| `camunda/camunda-bpmn-js` | `camunda-bpmn-js`  | Camunda-flavored bpmn-js distributions; three variants: `base/Modeler` (generic), `camunda-platform/Modeler` (C7), `camunda-cloud/Modeler` (C8/Zeebe)                                           |
| `camunda/camunda-dmn-js`  | `camunda-dmn-js`   | Camunda-flavored dmn-js distributions                                                                                                                                                           |

---

## Layer 5 — Properties Panels

| Repo                               | npm                         | Purpose                                                                                                         |
| :--------------------------------- | :-------------------------- | :-------------------------------------------------------------------------------------------------------------- |
| `bpmn-io/properties-panel`         | `@bpmn-io/properties-panel` | Base panel framework (groups, entries, FEEL editor integration)                                                 |
| `bpmn-io/bpmn-js-properties-panel` | `bpmn-js-properties-panel`  | BPMN-specific properties panel (peer deps: `bpmn-js`, `@bpmn-io/properties-panel`, `camunda-bpmn-js-behaviors`) |
| `bpmn-io/dmn-js-properties-panel`  | `dmn-js-properties-panel`   | DMN-specific properties panel                                                                                   |

`@bpmn-io/properties-panel` contains the FEEL editor (`feel-editor`) and is the base for both BPMN and DMN panels.

---

## Layer 6 — FEEL Language Tooling (DMN Expression Language)

FEEL = Friendly Enough Expression Language, part of the DMN spec.

| Repo                    | npm                      | Purpose                                                                 |
| :---------------------- | :----------------------- | :---------------------------------------------------------------------- |
| `bpmn-io/lezer-feel`    | `@bpmn-io/lezer-feel`    | Lezer (CodeMirror's parser framework) grammar for FEEL                  |
| `bpmn-io/lang-feel`     | `@bpmn-io/lang-feel`     | CodeMirror 6 language extension for FEEL (syntax highlighting, grammar) |
| `bpmn-io/feelin`        | `@bpmn-io/feelin`        | FEEL parser + interpreter (uses lezer-feel)                             |
| `bpmn-io/feel-editor`   | `@bpmn-io/feel-editor`   | CodeMirror-based FEEL editor widget (uses lang-feel, feel-lint)         |
| `bpmn-io/feel-lint`     | `@bpmn-io/feel-lint`     | FEEL expression linter                                                  |
| `bpmn-io/feel-analyzer` | `@bpmn-io/feel-analyzer` | Static analysis of FEEL expressions                                     |
| `bpmn-io/feelers`       | `feelers`                | Text templating solution built on FEEL                                  |
| `bpmn-io/cm-theme`      | `@bpmn-io/cm-theme`      | CodeMirror theme used by feel-editor and other CM-based editors         |
| `camunda/feel-builtins` | `@camunda/feel-builtins` | Camunda extensions to the FEEL built-in function library                |

__Key chain__: `lezer-feel` → `lang-feel` / `feelin` → `feel-editor` → `properties-panel` → `bpmn-js-properties-panel`

---

## Layer 7 — Linting / Validation

| Repo                                     | npm                                    | Purpose                                                                                                                                                                            |
| :--------------------------------------- | :------------------------------------- | :--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `bpmn-io/bpmnlint`                       | `bpmnlint`                             | Rule-based BPMN diagram linter (uses `bpmn-moddle`)                                                                                                                                |
| `bpmn-io/bpmnlint-utils`                 | `bpmnlint-utils`                       | Utilities for authoring bpmnlint rules                                                                                                                                             |
| `bpmn-io/bpmn-js-bpmnlint`               | `bpmn-js-bpmnlint`                     | bpmnlint plugin for bpmn-js (shows lint errors in canvas)                                                                                                                          |
| `bpmn-io/dmnlint`                        | `dmnlint`                              | DMN equivalent of bpmnlint                                                                                                                                                         |
| `bpmn-io/element-templates-validator`    | `@bpmn-io/element-templates-validator` | Validates element template JSON against schema                                                                                                                                     |
| `camunda/bpmnlint-plugin-camunda-compat` | `bpmnlint-plugin-camunda-compat`       | bpmnlint rules enforcing Camunda engine compatibility                                                                                                                              |
| `camunda/linting`                        | `@camunda/linting`                     | Linting orchestration for Camunda Desktop and Web Modeler; auto-configures rules from `modeler:executionPlatform` + `modeler:executionPlatformVersion` attributes in the BPMN file |
| `camunda/form-linting`                   | `@camunda/form-linting`                | Linting rules for form-js forms                                                                                                                                                    |

---

## Layer 8 — Extensions & Features

| Repo                                       | npm                                         | Purpose                                                                                                                                                                           |
| :----------------------------------------- | :------------------------------------------ | :-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `camunda/camunda-bpmn-js-behaviors`        | `camunda-bpmn-js-behaviors`                 | Camunda-specific bpmn-js behaviors ensuring model consistency (e.g. mutually exclusive extension elements); separate modules for C7 (`camunda-platform`) and C8 (`camunda-cloud`) |
| `bpmn-io/bpmn-js-element-templates`        | `bpmn-js-element-templates`                 | Element templates for bpmn-js; two providers: `ElementTemplatesPropertiesProviderModule` (C7) and `CloudElementTemplatesPropertiesProviderModule` (C8)                            |
| `bpmn-io/element-template-chooser`         | `@bpmn-io/element-template-chooser`         | UI dialog to pick/apply element templates (peer: `bpmn-js`)                                                                                                                       |
| `bpmn-io/element-template-icon-renderer`   | —                                           | Renders icons on elements that have a template applied                                                                                                                            |
| `bpmn-io/variable-resolver`                | `@bpmn-io/variable-resolver`                | Exposes the diagram's data model to other components; `ZeebeVariableResolverModule` (C8) / `CamundaVariableResolverModule` (C7); `variableResolver.getVariablesForElement(el)`    |
| `bpmn-io/extract-process-variables`        | `@bpmn-io/extract-process-variables`        | Statically extracts Camunda process variables from a BPMN diagram                                                                                                                 |
| `bpmn-io/bpmn-auto-layout`                 | `bpmn-auto-layout`                          | Auto-layout for BPMN diagrams (works without diagram-js, pure moddle)                                                                                                             |
| `bpmn-io/bpmn-js-token-simulation`         | `bpmn-js-token-simulation`                  | Simulates BPMN token flow for spec conformance testing                                                                                                                            |
| `bpmn-io/bpmn-js-differ`                   | `bpmn-js-differ`                            | Semantic diff of two BPMN documents                                                                                                                                               |
| `bpmn-io/dmn-js-differ`                    | `dmn-js-differ`                             | Semantic diff of two DMN documents                                                                                                                                                |
| `bpmn-io/bpmn-js-create-append-anything`   | `bpmn-js-create-append-anything`            | Extension for custom create/append menus                                                                                                                                          |
| `bpmn-io/align-to-origin`                  | `@bpmn-io/align-to-origin`                  | Aligns diagram elements to the canvas origin on save/export                                                                                                                       |
| `bpmn-io/bpmn-js-color-picker`             | `bpmn-js-color-picker`                      | Color picker extension — adds fill/stroke color support to bpmn-js elements                                                                                                       |
| `bpmn-io/bpmn-js-tracking`                 | `bpmn-js-tracking`                          | Analytics/tracking hooks for bpmn-js canvas interactions                                                                                                                          |
| `bpmn-io/variable-outline`                 | `@bpmn-io/variable-outline`                 | Renders a visual variable-scope outline overlay in the diagram                                                                                                                    |
| `bpmn-io/add-exporter`                     | `@bpmn-io/add-exporter`                     | Injects exporter name + version into BPMN `<exporter>` metadata on export                                                                                                         |
| `bpmn-io/bpmn-js-executable-fix`           | `bpmn-js-executable-fix`                    | Ensures `isExecutable=true` on BPMN processes (Camunda engine requirement)                                                                                                        |
| `bpmn-io/bpmn-js-copy-as-image`            | `bpmn-js-copy-as-image`                     | Copies the BPMN canvas as an image to the clipboard                                                                                                                               |
| `bpmn-io/form-variable-provider`           | `@bpmn-io/form-variable-provider`           | Feeds BPMN process variables into form-js for variable-aware form editing                                                                                                         |
| `bpmn-io/dmn-variable-resolver`            | `@bpmn-io/dmn-variable-resolver`            | Resolves input/output variable bindings across DMN decision tables                                                                                                                |
| `camunda/improved-canvas`                  | `@camunda/improved-canvas`                  | Camunda-specific canvas enhancements (keyboard shortcuts, drag behavior, toolbar)                                                                                                 |
| `camunda/task-testing`                     | `@camunda/task-testing`                     | Test utilities for Camunda user task forms                                                                                                                                        |
| `camunda/rpa-frontend`                     | `@camunda/rpa-integration`                  | RPA task integration for the Camunda Modeler                                                                                                                                      |
| `camunda/example-data-properties-provider` | `@camunda/example-data-properties-provider` | Provides example payload data in the properties panel for development/testing                                                                                                     |
| `bpmn-io/bpmn-js-i18n`                     | `bpmn-js-i18n`                              | Community-contributed translations / i18n strings for bpmn-js                                                                                                                     |

---

## Layer 9 — Tooling / CLI

| Repo                                              | npm                     | Purpose                                                             |
| :------------------------------------------------ | :---------------------- | :------------------------------------------------------------------ |
| `bpmn-io/bpmn-js-headless`                        | `bpmn-js-headless`      | Headless bpmn-js for server-side / Node.js environments             |
| `bpmn-io/eslint-plugin-bpmn-io`                   | `eslint-plugin-bpmn-io` | Shared ESLint rules for bpmn.io projects                            |
| `bpmn-io/sr`                                      | `@bpmn-io/sr`           | Setup and run utility for bpmn.io project development               |
| `bpmn-io/bpmn-to-image`                           | `bpmn-to-image`         | CLI: render BPMN to PNG/PDF (uses headless bpmn-js)                 |
| `bpmn-io/element-templates-cli`                   | `element-templates-cli` | CLI: apply element templates to BPMN files                          |
| `bpmn-io/bpmn-anonymize`                          | `bpmn-anonymize`        | CLI: anonymize labels in BPMN files                                 |
| `bpmn-io/dmn-migrate` / `bpmn-io/dmn-migrate-cli` | `dmn-migrate`           | Migrate DMN files between versions                                  |
| `bpmn-io/replace-ids`                             | `replace-ids`           | Replace ID placeholders in template files                           |
| `bpmn-io/bpmn-js-cli`                             | `bpmn-js-cli`           | CLI: inspect and manipulate bpmn-js diagrams from the command line  |
| `camunda/camunda-docs-modeler-screenshots`        | —                       | Automation: keep Camunda docs Modeler screenshots up to date        |
| `camunda/zeebe-connection-test`                   | —                       | Dev tool: test Zeebe broker connectivity from the Modeler           |
| `bpmn-io/svg-to-image`                            | `@bpmn-io/svg-to-image` | Converts SVG diagrams to PNG/JPEG (used for export and screenshots) |
| `camunda/feel-copilot`                            | `feel-copilot`          | AI-assisted FEEL expression authoring                               |

---

## Layer 10 — Modeler Application

The Camunda desktop and web modelers are the top-level apps that orchestrate all lower layers.

| Repo                                                      | npm                              | Purpose                                                                                                 |
| :-------------------------------------------------------- | :------------------------------- | :------------------------------------------------------------------------------------------------------ |
| `camunda/camunda-modeler`                                 | —                                | Camunda Modeler desktop app (Electron); integrates bpmn-js, dmn-js, form-js, linting, element templates |
| `camunda/camunda-hub`                                     | —                                | Camunda Web Modeler app (React/web); the web counterpart to camunda-modeler                             |
| `camunda/camunda-modeler-plugin-helpers`                  | `camunda-modeler-plugin-helpers` | Utilities for building Modeler plugins                                                                  |
| `camunda/camunda-modeler-plugin-example`                  | —                                | Starter template for Modeler plugins                                                                    |
| `camunda/camunda-modeler-plugins`                         | —                                | Official Modeler plugins collection                                                                     |
| `camunda/camunda-modeler-token-simulation-plugin`         | —                                | Token simulation Modeler plugin (wraps `bpmn-js-token-simulation`)                                      |
| `camunda/camunda-modeler-custom-linter-rules-plugin`      | —                                | Example: add custom bpmnlint rules to the Modeler                                                       |
| `camunda/camunda-modeler-process-io-specification-plugin` | —                                | Document input/output specs on BPMN processes                                                           |
| `camunda/cloud-connect-modeler-plugin`                    | —                                | Cloud Connect plugin for the Modeler                                                                    |
| `camunda/camunda-modeler-update-server`                   | `camunda-modeler-update-server`  | Update server for Modeler auto-update                                                                   |

---

## Notable consumers / integrations

| Repo                      | npm                        | Purpose                                              |
| :------------------------ | :------------------------- | :--------------------------------------------------- |
| `bpmn-io/vs-code-bpmn-io` | —                          | VS Code extension: view/edit BPMN, DMN, forms        |
| `bpmn-io/react-bpmn`      | —                          | React wrapper for bpmn-js viewer                     |
| `bpmn-io/vue-bpmn`        | —                          | Vue.js wrapper for bpmn-js viewer                    |
| `bpmn-io/react-form-js`   | —                          | React wrapper for form-js viewer                     |
| `camunda/form-playground` | `@camunda/form-playground` | Simulate forms with input/output data (uses form-js) |
| `bpmn-io/bpmn.io`         | —                          | Website source (handlebars templates)                |
| `bpmn-io/awesome-bpmn-io` | —                          | Curated list of community extensions                 |

---

## Dependency graph (simplified)

```text
primitives (min-dash, min-dom, tiny-svg, ids, didi)
    │
    ├─► moddle ──► moddle-xml ──► bpmn-moddle ───────────────────┐
    │                              dmn-moddle                     │
    │                              camunda-bpmn-moddle            │
    │                              zeebe-bpmn-moddle              │
    │                              modeler-moddle                 │
    │                                                             │
    ├─► diagram-js ──► diagram-js-direct-editing                  │
    │       │          diagram-js-minimap                         │
    │       │                                                     │
    │       ├──────────────────────────────► bpmn-js ◄────────────┘
    │       │                                   │
    │       │           camunda-bpmn-js ◄────────┘  (bundles behaviors + panels)
    │       │                   │
    │       │           bpmn-js-properties-panel
    │       │           bpmn-js-element-templates
    │       │           bpmn-js-bpmnlint
    │       │
    │       └─► table-js ──► dmn-js ──► camunda-dmn-js
    │                           │
    │                       dmn-js-properties-panel
    │
    ├─► lezer-feel ──► lang-feel / feelin
    │                       │
    │                   feel-editor ──► properties-panel
    │                   feel-lint
    │
    ├─► bpmnlint (uses bpmn-moddle)
    │       │
    │   bpmn-js-bpmnlint
    │   bpmnlint-plugin-camunda-compat ──► linting
    │
    └─► [all of the above] ──► camunda-modeler (Electron app)
                                    │
                                camunda-modeler-*-plugin
```

---

## Key conventions

* __DI via didi__: All diagram-js/bpmn-js modules export a plain object `{ __init__, __depends__, MyService }` consumed by the injector. Extensions are passed as `additionalModules`.
* __Moddle extensions__: Custom BPMN attributes are declared as moddle descriptor JSON and passed as `moddleExtensions`.
* __Monorepos__: `dmn-js`, `form-js` use npm workspaces with a `packages/` layout.
* __Peer deps__: Heavy use of peer dependencies — properties panels, element templates, and linting integrations declare `bpmn-js` and `diagram-js` as peers to avoid version conflicts.
* __Package prefixes__:
  * `@bpmn-io/` — newer/scoped packages
  * `bpmn-js-*` — bpmn-js plugins/extensions
  * `diagram-js-*` — diagram-js plugins
  * No prefix — original / foundational packages (`bpmn-js`, `diagram-js`, `moddle`, etc.)

---

See [UPDATE.md](./UPDATE.md) for instructions on how to update this skill.
