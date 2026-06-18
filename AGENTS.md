# Agent Instructions

Goal: Conduct large-scale experiments — from a clear goal to practical results and learnings.

## Onboarding

On first run, read `CUSTOMIZATION.md`. If it does not exist, copy `CUSTOMIZATION.TEMPLATE.md` to `CUSTOMIZATION.md` and prompt the user to fill it in before continuing.

`CUSTOMIZATION.md` defines:
- Where local repositories are stored (required)
- Additional steps to run when an experiment starts
- Additional steps to run when the user signals experiment completion

## Repository Structure

```
logs/
  <experiment-name>/
    GOAL.md          # Declared goal — written at experiment start
    knowledge/       # Learnings collected along the way — fully AI-authored
    SUMMARY.md       # Summary written at experiment end: executive overview + links to collateral
knowledge/           # General knowledge available to all experiments
CUSTOMIZATION.md     # User-local configuration (not committed)
CUSTOMIZATION.TEMPLATE.md  # Template for the above
```

## Starting an Experiment

1. Create `logs/<experiment-name>/GOAL.md` with a clear, concise statement of what the experiment is trying to learn or prove.
    * If not clear from user intend - grill user - iterate until a clear goal emerges.
2. Run any "experiment start" hooks defined in `CUSTOMIZATION.md`.
3. Work in a **git worktree** of the target repository (not this experiments repo) to keep the experiment isolated and enable parallel experiments.

## During an Experiment

- Store learnings as they emerge in `logs/<experiment-name>/knowledge/`.
- Store knowledge that is broadly reusable across experiments in `knowledge/`.
- Keep `GOAL.md` as the north star — call out if the experiment drifts from it.

## Pipeline Mode (code changes)

When an experiment involves making a **code change**, route it through dedicated
subagents instead of doing it all in one context. The bet: separate agents with clean,
scoped context catch more than one agent doing everything — the "four-eyes" effect. The
diff in the target worktree is the shared state between stages; the goal is re-injected
into every stage.

Stages (all defined in `.claude/agents/`):

0. **`explorer`** — read-only. Exploratory / decision-surfacing recon (see loop below).
   Runs *before* implementation; resolves the approach or escalates forks to the human.
1. **`implementer`** — builds the change against the goal. Can edit. Does not commit.
2. **`feedback`** — read-only. Completeness + repo-convention fit vs. the **goal**.
   If `gaps-found`, loop back to `implementer`.
3. **`reviewer`** — read-only. Adversarial correctness + security vs. the **diff**.

**Orchestration:** the main thread spawns each stage via the Agent tool (`subagent_type`
= the agent name), passing the `GOAL.md` path every time. Stages do not call each other.

### Stage 0 — the exploratory loop

The main thread runs a search loop over `explorer` agents. Subagents cannot spawn
subagents, so all fan-out and human interaction live in the orchestrator.

Each `explorer` returns one **status**:

| Status | Loop action |
| :-- | :-- |
| `possible` | terminal-ish → candidate to conclude `proceed` |
| `impossible` | terminal-ish → candidate to conclude `blocked` |
| `needs-validation` | recurse: spawn an explorer to verify the named assumption |
| `needs-deep-dive` | recurse: spawn one explorer on the narrowed question |
| `needs-deep-dive-in-branches` | recurse: spawn one explorer per named branch (parallel) |

**The loop:**
1. Spawn `explorer` with the goal, the current question, and the **ledger** (established
   facts / resolved forks / open questions). Copy `LEDGER.TEMPLATE.md` to
   `logs/<experiment-name>/LEDGER.md` on the first run; the orchestrator owns this file
   and merges each explorer's ledger delta into it (merge protocol is in the template).
2. Read its status and merge its **ledger delta** into the ledger.
3. On a recurse status, spawn the child explorer(s) it named and repeat from step 2.
4. Stop when no recurse statuses remain **or** a cap is hit (max depth / max explorers /
   budget). Caps that trip → conclude `unresolved`, never a silent pick.
5. Corroborate the loop-terminating verdicts: before concluding `proceed` or `blocked`,
   spawn one independent `explorer` to confirm `possible` / `impossible`.

**The phase concludes in exactly one state:**

| State | Meaning | Next |
| :-- | :-- | :-- |
| `proceed` | resolved `possible`, no open forks | hand to `implementer` |
| `blocked` | resolved `impossible` | stop, report to human |
| `needs-human` | a real fork only the human should decide | **human gate** |
| `unresolved` | caps exhausted before converging | stop, surface known + missing to human |

`needs-human` and `unresolved` are not failures — they are the point. The exploratory
phase exists to get the right decisions in front of the human, not to decide for them.

**Human gate:** the pipeline never auto-commits. Its goal is a *cleaner artifact for the
human to review*, not hands-off automation. Present the consolidated `feedback` + `reviewer`
findings to the human; the human decides on commit/merge.

**Context scoping (how clean context is enforced):**
- `.claude/settings.json` → `skillOverrides` curates the skill menu the whole experiment
  sees (irrelevant skills set `off`). Session-wide, shared by all agents. Tune as needed.
- Each agent's `skills:` frontmatter preloads only its intended skill(s).
- Each agent's `tools:` allowlist is the hard gate: `feedback` and `reviewer` have no
  Edit/Write, so they physically cannot implement — they can only report.

## Completing an Experiment

When the user signals completion:

0. Propose to user how to make the experiment publicly available - push changes to feature branches in selected repositories / create draft pull requests?
1. Write `logs/<experiment-name>/SUMMARY.md` — executive summary, key findings, links to collateral.
2. Promote any broadly useful learnings from `logs/<experiment-name>/knowledge/` to `knowledge/`.
3. Run any "experiment completion" hooks defined in `CUSTOMIZATION.md`.
4. Ask the user whether they want to share the results.
