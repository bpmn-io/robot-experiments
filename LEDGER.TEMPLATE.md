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

# Exploration Ledger — <experiment-name>

## Meta

- **Goal:** `logs/<experiment-name>/GOAL.md`
- **Loop state:** `in-progress`   <!-- in-progress | proceed | blocked | needs-human | unresolved -->
- **Caps:** max-depth=`<n>` · max-explorers=`<n>` · budget=`<n or none>`
- **Counters:** explorers-spawned=`0` · current-depth=`0`
- **Last updated by run:** `—`

---

## Frontier — open questions queue

The work queue. The loop pops `pending` items, spawns explorers, and pushes children.
Phase concludes when the frontier holds no `pending`/`exploring` items (or a cap trips).

| id | question | parent | depth | status |
| :-- | :-- | :-- | :-: | :-- |
| Q1 | `<the whole goal, on the first run>` | — | 0 | pending |

<!-- status: pending | exploring | done | spawned-children -->

---

## Established facts

Grounded, shared truth promoted from explorer `[FINDING]`s. Append-only.

| id | fact | provenance (path:line) | falsifier | from run |
| :-- | :-- | :-- | :-- | :-- |
| F1 | `<what is true>` | `path/to/file:LINE-LINE` | `wrong if: <condition>` | R1 |

---

## Resolved forks

Trade-offs that have been decided. Record WHO decided (human vs. clear-from-evidence).
Append-only.

| id | fork | options | chosen | decided by | rationale |
| :-- | :-- | :-- | :-- | :-- | :-- |
| D1 | `<the choice>` | `A / B` | `A` | `human \| evidence` | `<why>` |

---

## Open forks — for the human

Real trade-offs the loop must NOT decide itself. These drive a `needs-human` conclusion.

| id | fork | options | deciding factor | what we'd need to choose |
| :-- | :-- | :-- | :-- | :-- |
| — | | | | |

---

## Pending assumptions — awaiting validation

`[ASSUMPTION]`s a run leaned on but did not read. Each drives a `needs-validation` child.
Removed when upgraded to a Fact or refuted.

| id | assumption | to verify, read | raised by run |
| :-- | :-- | :-- | :-- |
| A1 | `<inferred claim>` | `path/to/file` | R1 |

---

## Run log — the search tree

Audit trail of every explorer spawn. Append-only. Makes the exploration reproducible
and comparable across repeated experiment runs.

| run | question (id) | status returned | children spawned | notes |
| :-- | :-- | :-- | :-- | :-- |
| R1 | Q1 | `<status>` | `Q2, Q3` | corroboration? y/n |
