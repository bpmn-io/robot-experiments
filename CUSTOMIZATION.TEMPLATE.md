# Customization

Copy this file to `CUSTOMIZATION.md` and fill in the sections below.

## Local Repositories

Where local repositories are checked out (required):

```
REPOS_DIR=~/path/to/repos
```

## Experiment Start Hooks

Additional steps to run when an experiment starts (optional).
Examples: open a browser tab, start a dev server, create a Slack channel.

```
# on_experiment_start:
#   - echo "Starting experiment"
```

## Experiment Completion Hooks

Additional steps to run when the user signals an experiment is complete (optional).
Examples: archive logs, post a summary to a channel, stop a dev server.

```
# on_experiment_complete:
#   - echo "Experiment complete"
```
