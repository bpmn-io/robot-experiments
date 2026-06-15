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

## Completing an Experiment

When the user signals completion:

1. Write `logs/<experiment-name>/SUMMARY.md` — executive summary, key findings, links to collateral.
2. Promote any broadly useful learnings from `logs/<experiment-name>/knowledge/` to `knowledge/`.
3. Run any "experiment completion" hooks defined in `CUSTOMIZATION.md`.
4. Ask the user whether they want to share the results.
