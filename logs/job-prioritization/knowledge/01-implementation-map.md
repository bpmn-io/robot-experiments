# Job Prioritization — Implementation Map (Stage 0 output)

Grounded plan from the explorer loop. Each repo mirrors the existing **user-task priority**
(`zeebe:PriorityDefinition` / `zeebe:priorityDefinition` / entry `priorityDefinitionPriority`)
which is the near-exact analog — but job priority is a **distinct** element on job-worker
tasks + `bpmn:Process`, with **no 0-99 value bound** (any signed 32-bit int or FEEL).

Canonical names (corroborated 4×): moddle type `zeebe:JobPriorityDefinition` → wire/binding
`zeebe:jobPriorityDefinition`; single attr `priority` (String); PP entry id
`jobPriorityDefinitionPriority`; doc link
`https://docs.camunda.io/docs/components/concepts/job-workers/#job-prioritization`.

Allowed elements: `bpmn:Process`, `bpmn:ServiceTask`, `bpmn:SendTask`,
`bpmn:BusinessRuleTask`, `bpmn:ScriptTask` (BR/Script only when job-worker-implemented,
i.e. have a `zeebe:TaskDefinition`).

## Per-repo recipe

1. **zeebe-bpmn-moddle** — append type to `resources/zeebe.json` `types` array, mirroring
   `PriorityDefinition` (lines 413-430): `superClass:["Element"]`, `meta.allowedIn` = the 5
   types (literal, D6), one `{name:"priority",type:"String",isAttr:true}`. Add read/write
   tests (read.js/write.js) + roundtrip fixture(s). `npm test` (mocha, no build).

2. **bpmn-js-properties-panel** — new `JobPriorityDefinitionProps.js` (mirror
   `PriorityDefinitionProps.js`) using `FeelNumberEntry`/`BpmnFeelEntry` `feel:'optional'`,
   id `jobPriorityDefinitionPriority`, **doc-link tooltip** (JSX `<a>`), and the
   ensure-`bpmn:ExtensionElements` guard from `TaskDefinitionProps.js:69-86`. New
   `JobPriorityDefinitionGroup` in `ZEEBE_GROUPS`, visible when
   `isZeebeServiceTask(el) || is(el,'bpmn:Process')`. Re-export in `properties/index.js`.
   Spec + `.bpmn` fixture. `npm test` (karma).

3. **camunda-bpmn-js-behaviors** — new `RemoveJobPriorityDefinitionBehavior` mirroring
   `CleanUpEndEventBehavior` (`postExecuted('shape.replace')`, read `context.newShape`,
   `modeling.updateModdleProperties` to drop the element when newShape is not a job-worker
   task). Register in `lib/camunda-cloud/index.js`. Spec + `.bpmn` fixture (execute/undo/redo
   + not-removed). Job-worker discriminator = `zeebe:TaskDefinition` presence. `npm test`.

4. **bpmnlint-plugin-camunda-compat** — gate-only (D1): new
   `rules/camunda-cloud/no-job-priority-definition.js` (mirror `no-priority-definition.js`,
   `hasNoExtensionElement(node,'zeebe:JobPriorityDefinition',node,'8.10')`). Register in
   `index.js` `rules` map + base `camundaCloud10Rules`, then `omit` it in
   `camundaCloud810Rules`. Update `test/config/configs.spec.js` (exact `to.eql` — every
   config). New rule spec. `npm run all`.

5. **linting** — add a branch in `error-messages.js` `getExtensionElementNotAllowedErrorMessage`
   (`is(extensionElement,'zeebe:JobPriorityDefinition')` → generic
   "A <Type> with <Job priority> is only supported by Camunda 8.10 or newer"), and in
   `properties-panel.js` `getEntryIds` (return `['jobPriorityDefinitionPriority']` via
   `isExtensionElementNotAllowedError`) + concise message in `getErrorMessage(id,report)`
   (`getNotSupportedMessage('', allowedVersion)`). Tests mirror priority-definition specs.
   Auto-bundled via `extends:['plugin:camunda-compat/all']`; `npm run compile-config` then
   `npm test`. Needs the linked compat rule + moddle type at test time.

6. **element-templates-json-schema** — Option A (D4): add `zeebe:jobPriorityDefinition` to
   the two binding enums (`properties.json:51-69` and `:1063-1089`), add
   `src/defs/properties/binding/jobPriorityDefinition.json` (`property` const `"priority"`)
   + `$ref`, optional `error-messages.json:152` update. NO element-type gate, NO feel rule
   (D3). Build `npm run build:zeebe` then `npm test`; add validation fixtures.

7. **bpmn-js-element-templates** — wire binding in 5 places mirroring `zeebe:priorityDefinition`:
   `bindingTypes.js` (const + `EXTENSION_BINDING_TYPES`), `propertyUtil.js`
   `getRawPropertyValue` + `setPropertyValue`, new `create/JobPriorityDefinitionBindingProvider.js`
   registered in `TemplateElementFactory.js`, `_updateJobPriorityDefinition` in
   `ChangeElementTemplateHandler.js`. Acceptance validated by `@bpmn-io/element-templates-validator`
   (needs schema from repo 6). Tests = template-JSON fixture + factory/handler specs. `npm test`.

## Cross-repo build reality
No worktree has `node_modules`. Foundation (repo 1) ships first; repos 2/3/4 need the new
moddle type at test time (link the repo-1 worktree); repo 5 needs the repo-4 rule + moddle;
repo 7 needs the repo-6 schema via the validator package. Where a green test strictly
requires an unpublished upstream, local-link it; otherwise document the wiring gap for the
human (publishing is out of scope).
