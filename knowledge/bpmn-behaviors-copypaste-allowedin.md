# camunda-bpmn-js-behaviors: CopyPasteBehavior already cleans up extension elements on morph

**Before writing a "remove `zeebe:X` when a task is morphed away from job worker" behavior,
check whether it's redundant.** It usually is.

## Mechanism

- A morph in bpmn-js (`bpmnReplace.replaceElement`) routes the business object through
  `ModdleCopy.copyElement` (`bpmn-js/lib/features/replace/BpmnReplace.js`).
- `ModdleCopy.copyProperty` fires the `moddleCopy.canCopyProperty` event per property; for an
  array property like `extensionElements.values` it recurses per child and DROPS any child the
  listeners reject.
- The cloud module's `CopyPasteBehavior` (`lib/camunda-cloud/CopyPasteBehavior.js`,
  `ZeebeModdleExtension.canCopyProperty`) returns `false` when an extension element's moddle
  `meta.allowedIn` has no match in the new parent's type hierarchy.

**Consequence:** when a task carrying `zeebe:X` is morphed to a type NOT in `X`'s `allowedIn`
(e.g. service task → user task, where `allowedIn` excludes `bpmn:UserTask`), `CopyPasteBehavior`
strips `zeebe:X` during the copy phase — *before* any `postExecuted('shape.replace')` handler
runs. A dedicated remove-on-morph behavior then finds nothing to do.

## How to test the redundancy claim

Disable the candidate behavior's registration in `lib/camunda-cloud/index.js` and run its spec.
If the morph spec still passes, `CopyPasteBehavior` is doing the work and the behavior is dead
code. (In job-prioritization, the spec passed 7/7 with the behavior disabled.)

## When a behavior IS warranted

`CopyPasteBehavior` keys purely on `allowedIn`. A dedicated behavior is only needed when removal
must depend on something `allowedIn` can't express — e.g. an in-place implementation toggle
(removing `zeebe:TaskDefinition` from a business-rule/script task WITHOUT a morph), which is an
`element.updateModdleProperties` event, not a `shape.replace`. Even then, confirm the
properties-panel visibility gate can't already produce that state. If the only "unique" case
requires an already-invalid model the UI never creates, skip the behavior.
