# Experiment Goal: Implement Job Prioritization in the Modeling Stack

Source issue: https://github.com/camunda/camunda-modeler/issues/5889
Drives: https://github.com/camunda/product-hub/issues/3573

## Statement

Add support for Camunda 8 **job prioritization** across the bpmn.io / Camunda modeling
tooling stack. A new `zeebe:jobPriorityDefinition` extension element (single `priority`
attribute, a FEEL-capable number) must be modelable, validated, surfaced in the
properties panel, cleaned up by behaviors, lint-checked for version compatibility, and
supported as an element-template binding.

## Scope — all 7 repos (dependency order)

1. **zeebe-bpmn-moddle** — Add `jobPriorityDefinition` type: single attribute `priority`
   (type String). Allowed in `bpmn:Process` and the service task, send task, business
   rule task, and script task.
2. **bpmn-js-properties-panel** — Add a "Job priority" group with a single `priority`
   field: number, FEEL optional. Shown on process, service task, send task, business
   rule task, and script task (latter two only when implemented as job worker). Tooltip
   sourced from the docs PR (camunda/camunda-docs#8935) with a proper link.
3. **camunda-bpmn-js-behaviors** — Add a behavior that removes the job priority
   definition when the task implementation changes away from job worker. Must remove the
   priority definition when a task is morphed to a user task or undefined task.
4. **bpmnlint-plugin-camunda-compat** — Add a rule verifying `zeebe:jobPriorityDefinition`
   is only allowed in Camunda 8.10+.
5. **linting** — Integrate the new rule: properties-panel integration (field focus,
   concise error message) and a human-readable error message.
6. **element-templates-json-schema** — Add a new binding type `zeebe:jobPriorityDefinition`,
   allowed only in supported types. No constraint on the `feel` field.
7. **bpmn-js-element-templates** — Implement the template binding.

## Success Criteria

1. Each repo builds and its existing test suite passes with the change.
2. New behavior is covered by tests written the way each repo writes them.
3. The XML samples from the issue round-trip:
   - `<zeebe:jobPriorityDefinition priority="50" />` on a `bpmn:process`.
   - `<zeebe:jobPriorityDefinition priority="90" />` on a service task alongside
     `zeebe:taskDefinition`.
4. Each repo's change is an isolated, reviewable diff on its own experiment branch.

## Allowed task types (job-worker–implemented)

Service task, send task, business rule task (job worker impl only), script task
(job worker impl only), and `bpmn:Process`.

## Out of Scope

- Engine-side implementation (this issue is downstream of the engine work).
- Releasing / version bumps / publishing any package.
- Cross-repo dependency wiring beyond what each repo needs to build & test locally
  (e.g. we do not publish the new moddle so downstream repos consume it via local link
  only if strictly necessary; otherwise each repo is validated against its own fixtures).

## Key Questions (for Stage 0)

- Exact moddle type definition shape & where it registers (does an analogous
  `zeebe:priorityDefinition` / `taskDefinition` pattern already exist to mirror?).
- How the properties panel decides "implemented as job worker" for business-rule/script
  tasks, and where FEEL-optional number fields are defined.
- The behaviors repo's existing "remove extension on morph" pattern to mirror.
- How the compat plugin encodes version-gated element rules (the 8.10 gate).
- The element-templates binding registration path in both schema and impl repos.
- The exact tooltip text/link from camunda-docs#8935.
