<!--
EXPLORATION LEDGER — shared state for the Stage 0 exploratory loop.

Copy this to `logs/<experiment-name>/LEDGER.md` when the loop starts.
The orchestrator (main thread) owns this file. Each `explorer` run RECEIVES the
current ledger and RETURNS a "ledger delta"; the orchestrator MERGES the delta here
before the next run. Explorers never write this file themselves.

Merge protocol:
- Append-only for Facts, Resolved Forks, and the Run Log. Never rewrite history.
- Frontier items move status pending -> exploring -> done (or -> spawned-children).
- A Pending Assumption is removed only when a later run upgrades it to a Fact
  (with provenance) or refutes it (record the refutation as a Fact).
- Every Fact must carry provenance (path:line) and a falsifier — same bar as the
  explorer output rules. No provenance => it is an Assumption, not a Fact.
-->

# Exploration Ledger — job-prioritization

## Meta

- **Goal:** `logs/job-prioritization/GOAL.md`
- **Loop state:** `proceed`   <!-- in-progress | proceed | blocked | needs-human | unresolved -->
- **Caps:** max-depth=`3` · max-explorers=`25` · budget=`none`
- **Counters:** explorers-spawned=`9` · current-depth=`2`
- **Last updated by run:** `R10 (Q7.1) — exploration concluded: proceed`
- **Corroboration:** load-bearing naming corroborated 4× (F12); per-repo corroboration delegated to the independent clean-context `feedback`+`reviewer` stages (re-derive intent from goal+diff). Tooltip (A1) and schema-gating (K2) independently re-verified by recursion explorers Q3.1/Q7.1.
- **Docs facts (camunda-docs#8935):** default priority 0; any signed 32-bit integer (NO 0-99 bound); static int or FEEL; tooltip link → `https://docs.camunda.io/docs/components/concepts/job-workers/#job-prioritization`.
- **Known non-blocking:** whether `bpmn:Process` is a supported element-template scope at runtime (repo 7) — does not block schema Option A; verify during impl. No worktree has node_modules — downstream test runs need `npm i` + local link to the new moddle where strictly necessary.

## Worktrees (branch `exp/job-prioritization`, based on origin/main)

Root: `/Users/gergely.juhasz/code/experiments/experiment_01/job-prioritization/`
zeebe-bpmn-moddle · bpmn-js-properties-panel · camunda-bpmn-js-behaviors ·
bpmnlint-plugin-camunda-compat · linting · element-templates-json-schema ·
bpmn-js-element-templates

---

## Frontier — open questions queue

The work queue. The loop pops `pending` items, spawns explorers, and pushes children.
Phase concludes when the frontier holds no `pending`/`exploring` items (or a cap trips).

| id | question | parent | depth | status |
| :-- | :-- | :-- | :-: | :-- |
| Q1 | whole goal — implement job prioritization across 7 repos | — | 0 | spawned-children |
| Q2 | zeebe-bpmn-moddle: define `jobPriorityDefinition` type | Q1 | 1 | done |
| Q3 | bpmn-js-properties-panel: job-priority group + FEEL number field | Q1 | 1 | spawned-children |
| Q4 | camunda-bpmn-js-behaviors: remove def on impl change/morph | Q1 | 1 | done |
| Q5 | bpmnlint-plugin-camunda-compat: 8.10+ gate rule | Q1 | 1 | done |
| Q6 | linting: integrate rule (pp focus + human message) | Q1 | 1 | done |
| Q7 | element-templates-json-schema: new binding type | Q1 | 1 | spawned-children |
| Q8 | bpmn-js-element-templates: implement binding | Q1 | 1 | done |
| Q3.1 | does FEEL field render forwarded tooltip? | Q3 | 2 | done (possible) |
| Q7.1 | schema element-type gating for multi-type set | Q7 | 2 | done (Option A) |

R1 (orchestrator): Q1 resolves to `needs-deep-dive-in-branches`. The source issue is
itself a 7-item checklist, one per repo, so the first-level branch decomposition is
established by the issue rather than re-derived by an explorer. Spawning one explorer
per repo (Q2–Q8) in parallel.

<!-- status: pending | exploring | done | spawned-children -->

---

## Established facts

Grounded, shared truth promoted from explorer `[FINDING]`s. Append-only.

| id | fact | provenance (path:line) | falsifier | from run |
| :-- | :-- | :-- | :-- | :-- |
| F1 | zeebe-bpmn-moddle has ONE descriptor; types register by membership in top-level `types` array (no registry; `associations:[]`) | `zeebe-bpmn-moddle/resources/zeebe.json:8` | wrong if: a separate registry must list types | R2 |
| F2 | Single-attr extension type pattern = `superClass:["Element"]` + `meta.allowedIn:[...]` + one `{name,type:"String",isAttr:true}`; `tagAlias:"lowerCase"` ⇒ type `JobPriorityDefinition`→`zeebe:jobPriorityDefinition`. `PriorityDefinition` (user-task) is exact mirror. | `zeebe-bpmn-moddle/resources/zeebe.json:5-7,413-430` | wrong if: tagAlias does not lowercase first letter | R2 |
| F3 | `bpmn:Process` is a valid `allowedIn` target | `zeebe-bpmn-moddle/resources/zeebe.json:650-667 (VersionTag)` | wrong if: VersionTag uses different mechanism | R2 |
| F4 | PP groups added via factory in `ZEEBE_GROUPS` returning null when empty; per-element visibility lives in the `*Props({element})` fn. FEEL-optional number = `BpmnFeelEntry({feel:'optional'})`. Closest template: `PriorityDefinitionProps.js`. | `bpmn-js-properties-panel/src/provider/zeebe/ZeebePropertiesProvider.js:53-114,163-175` | wrong if: provider stops filtering nulls | R3 |
| F5 | "Job worker impl" gate = `isZeebeServiceTask(element)` (business-rule/script require `zeebe:TaskDefinition`); does NOT cover Process ⇒ new gate = `isZeebeServiceTask(el) || is(el,'bpmn:Process')`. New setValue must use TaskDefinitionProps' ensure-extensionElements guard (PriorityDefinitionProps lacks it). | `bpmn-js-properties-panel/src/provider/zeebe/utils/ZeebeServiceTaskUtil.js:18-37`, `TaskDefinitionProps.js:69-86` | wrong if: ZeebeServiceTask.extends gains Process | R3 |
| F6 | Behaviors: morph-and-remove pattern = `extends CommandInterceptor` + `postExecuted('shape.replace', fn)` reading `event.context.newShape` → `modeling.updateModdleProperties`. Canonical: `CleanUpEndEventBehavior.js`. Job-worker discriminator = `getExtensionElementsList(bo,'zeebe:TaskDefinition')[0]`. Register in single `lib/camunda-cloud/index.js`. | `camunda-bpmn-js-behaviors/lib/camunda-cloud/CleanUpEndEventBehavior.js:10-35`, `index.js:15-57` | wrong if: morph emits no shape.replace | R4 |
| F7 | bpmnlint compat uses a RULE PAIR: `no-<x>` (forbids below min ver, in base config, carries version string) + `<x>` (validates value, added at min ver), swapped via `omit`/spread in `index.js`. Gate logic = `hasNoExtensionElement(node,type,node,'<ver>')`. `camunda-cloud-8-10` bucket ALREADY exists (ver string `'8.10'`). Precedent: `no-priority-definition`/`priority-definition` (8.6). | `bpmnlint-plugin-camunda-compat/rules/camunda-cloud/no-priority-definition.js:7-14`, `index.js:88-129,202-211` | wrong if: configs loaded externally | R5 |
| F8 | bpmnlint tests use rule-tester w/ inline XML + `config:{version}`; rules must ALSO be added to `test/config/configs.spec.js` which deep-equals every config's full rule set (`to.eql`). | `bpmnlint-plugin-camunda-compat/test/camunda-cloud/no-task-schedule.spec.js:35-64`, `test/config/configs.spec.js:594-740` | wrong if: expectRules does subset match | R5 |
| F9 | linting has 2 message surfaces keyed on `report.data.type`: human (`lib/utils/error-messages.js` getErrorMessage→getExtensionElementNotAllowedErrorMessage) + panel focus/concise (`lib/utils/properties-panel.js` getEntryIds + getErrorMessage(id,report)). Compat rule auto-bundled via `extends:['plugin:camunda-compat/all']` (compile-config). Analog: user-task priority, entry id `priorityDefinitionPriority`. | `linting/lib/utils/error-messages.js:101,342,746`, `properties-panel.js:335,624,756`, `tasks/compile-config.js:8` | wrong if: new rule not in compat `all` config | R6 |
| F10 | element-templates-json-schema validates the TEMPLATE DOCUMENT, not the BPMN element. Binding `type` enum at `properties.json:1063-1089` (duplicated 51-69 + error-messages.json:152 — edit all 3). Existing `zeebe:priorityDefinition` is USER-TASK (gated to require sibling `zeebe:userTask`), distinct from job priority. Build req: `src/`→gitignored `resources/schema.json` via `npm run build:zeebe`; tests read built file. | `element-templates-json-schema/packages/zeebe-element-templates-json-schema/src/defs/properties.json:1063-1089`, `template.json:292-349` | wrong if: enum resolved via different $ref | R7 |
| F11 | bpmn-js-element-templates wires a binding type in EXACTLY 5 places: `bindingTypes.js` (const + `EXTENSION_BINDING_TYPES`), `propertyUtil.js` getRawPropertyValue + setPropertyValue, a `create/*BindingProvider.js` registered in `TemplateElementFactory.js`, a `_update*` in `ChangeElementTemplateHandler.js`. Template ACCEPTANCE validated by `@bpmn-io/element-templates-validator` (external schema pkg), not a local list. Closest analog: `zeebe:priorityDefinition` (user-task). `ensureExtension`/`createElement` need no change. | `bpmn-js-element-templates/src/cloud-element-templates/util/bindingTypes.js:23-48`, `propertyUtil.js:339-343,1058-1088`, `create/PriorityDefinitionBindingProvider.js`, `cmd/ChangeElementTemplateHandler.js:1522-1533`, `Validator.js:9-16` | wrong if: a local binding-type allow-list gates validation | R8 |
| F12 | CROSS-REPO: 4 explorers independently converged on naming `zeebe:JobPriorityDefinition` (moddle type) → `zeebe:jobPriorityDefinition` (wire+binding), attr `priority` (String), mirroring the user-task priority convention. No worktree has `node_modules` installed; downstream test runs require `npm ci/install` and the moddle type present (local link where strictly necessary). | R2/R3/R5/R8 cross-corroboration | wrong if: a repo names it differently | R2-R8 |

---

## Resolved forks

Trade-offs that have been decided. Record WHO decided (human vs. clear-from-evidence).
Append-only.

| id | fork | options | chosen | decided by | rationale |
| :-- | :-- | :-- | :-- | :-- | :-- |
| D1 (K1) | bpmnlint gate-only vs gate+value | a/b | **gate-only** (`no-job-priority-definition` @8.10, no value rule) | evidence | docs PR: "Priority accepts any signed 32-bit integer. The engine does not enforce a fixed upper or lower bound such as `0-99`." No range to validate; issue asks only for the version gate. |
| D2 (A1) | does FEEL field render tooltip | renders / dropped | **renders** (use `tooltip` prop on (Bpmn)FeelEntry/FeelNumberEntry) | evidence | FeelTextfield→TooltipWrapper→Tooltip renders JSX incl. `<a href>` (R/Q3.1, index.esm.js:2872,318,468) |
| D3 (A2) | feel constraint on template binding | unrestricted / forbid | **unrestricted** (add no feel rule) | evidence | docs: priority may be static integer OR FEEL expr; matches `taskDefinition` (no feel rule) |
| D4 (K2) | element-templates schema element gating | A: none (mirror taskDefinition) / B: multi-type elementType gate | **A — no schema-level element gate** | evidence | `zeebe:taskDefinition` (multi-task analog) has NO element-type gate in schema; gating left to runtime/lint (Q7.1, properties.json:884-906) |
| D5 (field type) | PP priority field component | FeelEntry text / FeelNumberEntry | **FeelNumberEntry** (FEEL-optional number) | evidence | goal says "number, FEEL optional"; FeelNumberEntry renders tooltip identically (Q3.1) |
| D6 (K3) | moddle `allowedIn` shape | literal 5 types / ZeebeServiceTask aggregate | **literal: [Process, ServiceTask, SendTask, BusinessRuleTask, ScriptTask]** | evidence (confirm at gate) | matches issue's exact list + includes Process (aggregate excludes Process & over-includes events); doesn't affect round-trip |
| D7 (K4) | behaviors trigger scope | morph-only / morph + in-place toggle | **morph (`shape.replace`) — required; in-place toggle as additional if low-cost** | evidence (confirm at gate) | issue explicitly requires morph→user/undefined-task removal; "implementation changed outside of job worker" suggests also covering in-place script/BR toggle |

---

## Open forks — for the human

Real trade-offs the loop must NOT decide itself. These drive a `needs-human` conclusion.

| id | fork | options | deciding factor | what we'd need to choose |
| :-- | :-- | :-- | :-- | :-- |
| K1 | bpmnlint: gate-only vs gate+value rule | (a) only `no-job-priority-definition` 8.10 gate / (b) also a `job-priority-definition` value rule mirroring user-task priority (0–100/FEEL) | whether job priority has a documented value range | docs PR camunda-docs#8935 (does priority have a numeric range?) — fetching |
| K2 | element-templates-json-schema: how to express "only supported types" for a MULTI-type set (service/send/business-rule/script + Process) | (a) register binding type+`property` only, leave element gating to impl/lint / (b) multi-marker `contains`/anyOf rule | whether the schema is expected to gate element type at all (existing single-marker `zeebe:userTask` trick can't cover the set; Process has no marker) | recursion explorer Q7.1 + how impl repo relies on it |
| K3 | moddle `allowedIn` shape | (a) literal 5 types (matches issue exactly + includes Process) / (b) `["bpmn:Process","zeebe:ZeebeServiceTask"]` (DRY but over-allows EndEvent/IntermediateThrowEvent/AdHocSubProcess) | exact-issue-match vs repo DRY convention; does not affect XML round-trip | low stakes — orchestrator leans (a); confirm at human gate |
| K4 | behaviors trigger scope | (a) `shape.replace` morph only (the explicit verification ask) / (b) also `element.updateModdleProperties` for in-place job-worker toggle on script/business-rule tasks | whether "implementation changed outside of job worker" includes non-morph in-place toggle | issue wording leans superset (both); confirm at human gate |

---

## Pending assumptions — awaiting validation

`[ASSUMPTION]`s a run leaned on but did not read. Each drives a `needs-validation` child.
Removed when upgraded to a Fact or refuted.

| id | assumption | to verify, read | raised by run |
| :-- | :-- | :-- | :-- |
| A1 | FEEL fields (`BpmnFeelEntry`/`FeelTextfield`) actually RENDER a forwarded `tooltip` prop on the field label | `bpmn-js-properties-panel` node_modules `@bpmn-io/properties-panel` dist FeelTextfield fn | R3 → Q3.1 |
| A2 | "no constraint on feel" = add NO feel rule (vs forbid feel like versionTag) | element-templates-json-schema feel rules + docs PR intent | R7 → folded into Q7.1 |

---

## Run log — the search tree

Audit trail of every explorer spawn. Append-only. Makes the exploration reproducible
and comparable across repeated experiment runs.

| run | question (id) | status returned | children spawned | notes |
| :-- | :-- | :-- | :-- | :-- |
| R1 | Q1 (whole goal) | needs-deep-dive-in-branches | Q2–Q8 | decomposition = issue's own 7-item checklist (orchestrator) |
| R2 | Q2 zeebe-bpmn-moddle | possible | — | open fork K3 (allowedIn shape) |
| R3 | Q3 bpmn-js-properties-panel | needs-validation | Q3.1 | A1: does FEEL field render tooltip? |
| R4 | Q4 camunda-bpmn-js-behaviors | possible | — | open fork K4 (trigger scope) |
| R5 | Q5 bpmnlint-plugin-camunda-compat | possible | — | open fork K1 (gate-only vs +value) |
| R6 | Q6 linting | possible | — | depends on names from R2/R3/R5 (resolved by F12) |
| R7 | Q7 element-templates-json-schema | needs-validation | Q7.1 | K2 (multi-type gating), A2 (feel) |
| R8 | Q8 bpmn-js-element-templates | needs-validation | Q8.1/Q8.2 resolved by F12 | naming resolved by cross-corroboration |
| R1 | Q1 | `<status>` | `Q2, Q3` | corroboration? y/n |
