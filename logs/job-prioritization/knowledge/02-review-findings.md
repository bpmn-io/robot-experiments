# Job Prioritization — Consolidated Feedback + Reviewer Findings

Pipeline run: explorer loop → implementer (×7) → feedback (×7) → reviewer (×7).
All 7 repos build and their suites pass with the change (with a local moddle overlay +
Node 20). Below is the human-gate summary. Diffs live on branch `exp/job-prioritization`
in each worktree under `/Users/gergely.juhasz/code/experiments/experiment_01/job-prioritization/`.

## Per-repo verdicts (after resolution)

| repo | feedback | reviewer | net | final |
| :-- | :-- | :-- | :-- | :-- |
| zeebe-bpmn-moddle | complete | approve (0) | clean | ✅ kept |
| bpmn-js-properties-panel | complete | **changes-requested → approve (re-review)** | 1 blocker + 1 major, both FIXED | ✅ kept |
| camunda-bpmn-js-behaviors | complete | **changes-requested** | redundant with existing `CopyPasteBehavior` | ⏪ REVERTED |
| bpmnlint-plugin-camunda-compat | complete | approve (0) | clean | ✅ kept |
| linting | complete | approve (0) | clean | ✅ kept |
| element-templates-json-schema | complete | approve (0) | clean | ✅ kept |
| bpmn-js-element-templates | complete | approve (0) | clean (1 known cross-repo blocker) | ✅ kept |

## Resolution (human-gate decisions applied)

- **properties-panel**: both findings FIXED, then re-reviewed → **approve (0 findings)**. Gate
  narrowed to a precise `hasJobPriority` helper (the 5 allowed types only; BR/script require a
  `zeebe:TaskDefinition`); field switched to string-emitting `BpmnFeelEntry` like `retries`
  (dead `BpmnFeelNumberEntry.js` deleted); hidden-on {user task, message end/throw event}
  tests added. 1898 tests pass.
- **camunda-bpmn-js-behaviors**: REVERTED. The new `RemoveJobPriorityDefinitionBehavior` is
  redundant for all valid models — existing `CopyPasteBehavior` strips `jobPriorityDefinition`
  on morph via `allowedIn` (spec passed 7/7 with the behavior disabled). Its only unique case
  (morphing a BR/script task that has `jobPriorityDefinition` but no `taskDefinition`) requires
  an already-invalid model that the properties panel never produces. **Net finding: issue scope
  item 3 is already satisfied by existing infrastructure; no new code is needed.** Worktree
  restored to clean `origin/main`.
- **Publishing**: none yet — all changes remain local on `exp/job-prioritization` branches.

## Actionable findings

### properties-panel — [BLOCKER] visibility gate broader than schema
`src/provider/zeebe/properties/JobPriorityDefinitionProps.js:32`
Gate `isZeebeServiceTask(element) || is(element,'bpmn:Process')` also returns true for
message end events, message intermediate throw events, and ad-hoc subprocesses (all part
of the `zeebe:ZeebeServiceTask` trait). The moddle `allowedIn` and the GOAL permit only
Process + service/send/business-rule/script task. Editing the field on those extra
elements writes a `zeebe:JobPriorityDefinition` the schema/lint forbid. Untested.
**Fix direction:** gate explicitly on the 5 allowed types (Process, ServiceTask, SendTask,
and BusinessRuleTask/ScriptTask only when they carry a `zeebe:TaskDefinition`).

### properties-panel — [MAJOR] number-vs-string value drift
`src/entries/BpmnFeelNumberEntry.js` + `JobPriorityDefinitionProps.js` setValue.
`FeelNumberEntry` static input emits a JS number; the moddle attr is String. In-memory
`90` (number) vs `"90"` after save+reload. The `retries` precedent uses the string-emitting
`BpmnFeelEntry`. New specs assert the number, so they bake in the drift.
**Fix direction:** use `BpmnFeelEntry` like `retries`, or coerce the static value to a
string in setValue; assert the string `'90'`/`'50'` to match the moddle + XML fixtures.

### behaviors — [DESIGN FORK] new behavior is largely redundant
`lib/camunda-cloud/RemoveJobPriorityDefinitionBehavior.js`
Verified empirically: `RemoveJobPriorityDefinitionBehaviorSpec.js` passes 7/7 even with the
behavior DISABLED, because `CopyPasteBehavior` (`ZeebeModdleExtension.canCopyProperty`)
already strips extension elements whose `allowedIn` excludes the morph target — and a morph
routes through `ModdleCopy.copyElement`. So for morph→user/undefined task the behavior is a
no-op. Its ONE unique case: morphing a business-rule/script task that has a
`jobPriorityDefinition` but no `zeebe:TaskDefinition` (allowedIn keeps it; `isJobWorker`
strips it). That case is untested and relies on an unusual source model.
**Options:** (a) keep the behavior + add a test for its unique BRT/ScriptTask-without-
taskDefinition case; (b) drop it as redundant and rely on `CopyPasteBehavior`; (c) revisit
whether the moddle `allowedIn` should be conditional. The committed spec gives false
coverage either way (it passes without the behavior).

## Non-blocking should-fixes
- properties-panel: add a "hidden on user task" assertion (the `UserTask_1` fixture node is
  currently unused).
- behaviors: add a script/business-rule-source morph test (only service-task source covered).
- element-templates-json-schema: cosmetic fixture key-quoting drift.

## Known accepted cross-repo blocker (not a code defect)
- bpmn-js-element-templates: template ACCEPTANCE for `zeebe:jobPriorityDefinition` is
  validated by the published `@bpmn-io/element-templates-validator` (precompiled dist, not
  rebuildable locally). Validator-acceptance tests were intentionally omitted; runtime
  read/write/create/change-handler is fully tested. Proper fix: release
  element-templates-json-schema → rebuild the validator → bump it here.

## Build/test environment notes
- All worktrees need Node 20+ (system default is Node 10, which can't install/run).
- Downstream repos need the new moddle (and linting needs the new compat rule) copied into
  `node_modules` as a local link; lost on a clean `npm install` until upstream is published.
