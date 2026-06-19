# Adding a Zeebe extension element across the bpmn.io / Camunda modeling stack

Reusable map for any feature that introduces a new `zeebe:*` extension element (single or
few attributes). Validated by the `job-prioritization` experiment (camunda-modeler#5889).

## The 7 repos and their roles (dependency order)

1. **zeebe-bpmn-moddle** — defines the moddle type. ONE descriptor `resources/zeebe.json`;
   types register by membership in the top-level `types` array. Single-attribute pattern:
   `superClass:["Element"]` + `meta.allowedIn:[...]` + `{name,type:"String",isAttr:true}`.
   `xml.tagAlias:"lowerCase"` ⇒ type `FooBar` serializes to `zeebe:fooBar`. No build step;
   hand-written read/write/roundtrip tests + `*.part.bpmn` fixtures.
2. **bpmn-js-properties-panel** — UI. Groups are factories in `ZEEBE_GROUPS` returning `null`
   when empty; per-element visibility lives inside the `*Props({element})` fn. FEEL fields:
   `BpmnFeelEntry` (string-emitting) vs `FeelNumberEntry` (emits a JS **number** — avoid for a
   String moddle attr). Tooltips: pass a JSX `tooltip` prop; it renders via FeelTextfield →
   TooltipWrapper → Tooltip (incl. `<a>` links). `setValue` must ensure `bpmn:ExtensionElements`
   exists (Process/unconfigured elements lack it).
3. **camunda-bpmn-js-behaviors** — cleanup on morph. ⚠️ Often NOT needed: `CopyPasteBehavior`
   (`ZeebeModdleExtension.canCopyProperty`) already strips extension elements whose moddle
   `meta.allowedIn` excludes the morph target, and a morph (`shape.replace`) routes through
   `ModdleCopy.copyElement`. Check this BEFORE writing a remove-on-morph behavior — see
   [bpmn-behaviors-copypaste-allowedin.md](bpmn-behaviors-copypaste-allowedin.md).
4. **bpmnlint-plugin-camunda-compat** — version gate. Rule PAIR idiom: `no-<x>` (forbids below
   min version, in base config, `omit`ted at/above) + optional `<x>` (value rule at min version).
   `hasNoExtensionElement(node, type, node, '<ver>')`. `test/config/configs.spec.js` deep-equals
   EVERY config — add the rule to every bucket below the gate + `all`.
5. **linting** — turns a compat report into messages. Two surfaces keyed on `report.data.type`:
   human (`lib/utils/error-messages.js`) + panel focus/concise (`lib/utils/properties-panel.js`
   `getEntryIds` + `getErrorMessage(id, report)`). The panel entry id MUST match the
   properties-panel field id exactly. Auto-bundles via `extends:['plugin:camunda-compat/all']`
   + `npm run compile-config`.
6. **element-templates-json-schema** — template binding type. Validates the template DOCUMENT,
   not the BPMN element. Add to BOTH binding-type and field-type enums + a `binding/<name>.json`
   `property` const + `$ref` (append to `allOf` end to avoid shifting fixture `schemaPath`
   indices). Multi-element bindings (like `taskDefinition`) add NO element-type gate. `src/` →
   gitignored `resources/schema.json` via `npm run build:zeebe`; tests read the built file.
7. **bpmn-js-element-templates** — binding runtime. Wire in exactly 5–6 sites mirroring an
   existing binding (`bindingTypes.js` const + `EXTENSION_BINDING_TYPES`; `propertyUtil.js`
   get + set; a `create/*BindingProvider.js` registered in `TemplateElementFactory.js`; a
   `_update*` in `ChangeElementTemplateHandler.js` + its 2 internal helpers). Template
   ACCEPTANCE is validated by the published `@bpmn-io/element-templates-validator` (precompiled
   dist) — NOT a local list. It can't be rebuilt locally, so cross-repo acceptance is gated on
   releasing the schema → rebuilding the validator.

## Cross-repo realities

- **Find the closest existing analog and mirror it.** Most of these repos have a near-identical
  precedent (e.g. user-task `PriorityDefinition` for a priority field). Mirroring beats inventing.
- **Release ordering matters.** Downstream repos pin *published* upstream versions. For local
  end-to-end testing, overlay the new moddle/rule/schema into `node_modules` (lost on clean
  install). A real merge must publish in dependency order.
- **Node 20+** required across these repos; system Node 10 cannot install/run.
- Confirm value semantics from the **docs PR** early (e.g. valid range) — it decides whether a
  lint value-rule and panel min/max are needed.
