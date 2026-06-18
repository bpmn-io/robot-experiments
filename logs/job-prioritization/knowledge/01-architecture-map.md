# Job Prioritization — Cross-Repo Architecture Map

How `zeebe:jobPriorityDefinition` (a single `priority` attribute, FEEL-capable integer)
threads through the 7 modeling-stack repos. Grounded by the Stage-0 explorer loop
(see `../LEDGER.md` facts F1–F12).

## Naming (corroborated 4×)
- moddle type: `JobPriorityDefinition`  → wire tag `zeebe:jobPriorityDefinition` (via `tagAlias:"lowerCase"`)
- attribute: `priority` (type String in moddle; semantically a signed 32-bit integer or FEEL expr)
- element-template binding type: `zeebe:jobPriorityDefinition`, binding `property: "priority"`
- PP field id (proposed): `jobPriorityDefinitionPriority`
- bpmnlint rule: `no-job-priority-definition` (gate) — gate-only, min version `8.10`

## Allowed elements
`bpmn:Process` + service task, send task, business rule task, script task. Business-rule
and script tasks only when implemented as a job worker (`zeebe:TaskDefinition` present).
`isZeebeServiceTask(el) || is(el,'bpmn:Process')` is the PP visibility gate.

## Dependency chain (implementation order)
1. **zeebe-bpmn-moddle** (foundation) — defines the type. No upstream deps.
2. Consumers of the moddle type (parse XML containing the element):
   - **bpmn-js-properties-panel** — "Job priority" group, `FeelNumberEntry`, doc-link tooltip.
   - **camunda-bpmn-js-behaviors** — remove def on morph away from job worker.
   - **bpmnlint-plugin-camunda-compat** — `no-job-priority-definition` 8.10 gate.
3. **linting** — consumes the compat rule (`extends: plugin:camunda-compat/all`), maps it
   to a human message + PP field focus.
4. **element-templates-json-schema** — registers binding `type` + `property:"priority"`,
   no element-type gate (mirrors `taskDefinition`), no feel constraint.
5. **bpmn-js-element-templates** — implements the binding (5 wiring points); template
   acceptance is validated via `@bpmn-io/element-templates-validator` (which bundles the
   schema from repo 4).

## Key decisions (evidence-resolved — see LEDGER Resolved Forks D1–D7)
- **gate-only** lint rule (docs: priority is any signed 32-bit int, no 0-99 bound).
- **FeelNumberEntry** for the PP field; do NOT clamp 0-100 (unlike user-task priority).
- element-templates schema adds **no element-type gate** (Option A, mirror taskDefinition).
- feel left **unconstrained** on the template binding.
- moddle `allowedIn` uses the **literal 5-type list** (matches issue; includes Process).

## Build/test reality
No worktree has `node_modules`. Downstream repos pin *published* versions of
zeebe-bpmn-moddle / bpmnlint-plugin-camunda-compat / @bpmn-io/element-templates-validator
that do NOT yet contain this feature. New tests that parse the new element therefore need
the local upstream worktree linked (`npm install` + link) — "strictly necessary" only for
repos whose new fixtures reference the new element/rule/binding.
