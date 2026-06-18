# Experiment Summary: Implement Job Prioritization in the Modeling Stack

**Source issue:** [camunda/camunda-modeler#5889](https://github.com/camunda/camunda-modeler/issues/5889)
**Branch (all repos):** `exp/job-prioritization` (local only — nothing pushed)
**Outcome:** Feature implemented across 6 of 7 repos; the 7th (behaviors) was found
unnecessary and reverted. All kept changes pass their repos' test suites and cleared an
adversarial review.

## Executive overview

Added support for the Camunda 8 `zeebe:jobPriorityDefinition` extension element (a single
`priority` attribute — a FEEL-capable signed 32-bit integer) end-to-end across the bpmn.io /
Camunda modeling tooling. Ran via the Stage-0 explorer loop + per-repo
implementer→feedback→reviewer pipeline defined in `AGENTS.md`.

The pipeline's separation of concerns paid off: every implementer reported its repo "green,"
yet the independent reviewer stage caught a redundant repo's worth of code, a correctness
blocker, and a silent type drift — none of which a single-pass build would have surfaced.

## What shipped (kept changes)

| Repo | Change | Tests |
| :-- | :-- | :-- |
| zeebe-bpmn-moddle | `JobPriorityDefinition` type (`allowedIn`: Process + service/send/business-rule/script task) | 112 |
| bpmn-js-properties-panel | "Job priority" group, FEEL-optional string field, doc-link tooltip, precise visibility gate | 1898 |
| bpmnlint-plugin-camunda-compat | gate-only `no-job-priority-definition` rule (allowed only ≥ 8.10) | 1712 |
| linting | human message + properties-panel field focus + concise message | 314 |
| element-templates-json-schema | new `zeebe:jobPriorityDefinition` binding type (no feel/element gate) | 407 |
| bpmn-js-element-templates | binding implementation (5 wiring points + change handler) | 1347 |

## What did NOT ship

- **camunda-bpmn-js-behaviors (issue scope item 3): reverted.** A dedicated
  "remove job priority on morph" behavior is redundant — the existing `CopyPasteBehavior`
  (`ZeebeModdleExtension.canCopyProperty`) already strips extension elements whose moddle
  `meta.allowedIn` excludes the morph target, and a morph routes through `ModdleCopy`.
  Verified: the behavior's own spec passed 7/7 with it disabled. **Recommendation: scope
  item 3 needs no new code** (verify the engine/UX expectation, then close it).

## Key decisions (evidence-resolved)

- Lint rule is **gate-only** — docs PR ([camunda-docs#8935](https://github.com/camunda/camunda-docs/pull/8935))
  state priority is any signed 32-bit integer with **no** `0–99` bound, so there is no value
  range to validate (contrast: user-task `PriorityDefinition` is 0–100).
- Properties-panel field stores a **string** (matches the moddle `String` attr and the
  `retries` precedent), not a JS number.
- Element-templates schema adds **no element-type gate** (mirrors `zeebe:taskDefinition`).
- Moddle `allowedIn` uses the **literal 5-type list** (matches the issue; includes Process).

## Findings the pipeline surfaced (four-eyes value)

1. An entire redundant repo change (behaviors) — empirically verified, then reverted.
2. [blocker] properties-panel visibility gate wrote schema-invalid models on event elements.
3. [major] properties-panel number↔string value drift across save/reload.

## Open items for a real merge

1. **Release ordering** (not auto-handled here): zeebe-bpmn-moddle → properties-panel & bpmnlint;
   bpmnlint → linting; element-templates-json-schema → `@bpmn-io/element-templates-validator`
   → bpmn-js-element-templates. Downstream repos here consumed upstream changes via local
   `node_modules` overlays (lost on clean install).
2. **Element-templates validator acceptance** is the one true cross-repo blocker — templates
   using the new binding are rejected until the validator is rebuilt against the released
   schema. Runtime read/write/create is fully tested; acceptance tests were deferred.
3. Requires **Node 20+** in every repo (system default Node 10 cannot install/run).

## Collateral

- [GOAL.md](GOAL.md) — declared goal & scope.
- [LEDGER.md](LEDGER.md) — Stage-0 exploration ledger (facts, resolved forks, run tree).
- [knowledge/01-architecture-map.md](knowledge/01-architecture-map.md) — cross-repo map.
- [knowledge/02-review-findings.md](knowledge/02-review-findings.md) — full feedback+reviewer record.
- Diffs: `exp/job-prioritization` worktrees under
  `/Users/gergely.juhasz/code/experiments/experiment_01/job-prioritization/`.
