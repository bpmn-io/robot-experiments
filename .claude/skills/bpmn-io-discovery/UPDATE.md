# Updating this skill

Instructions how to update [repositories.json](./repositories.json) and [SKILL.md](./SKILL.md) files.

## Source: Repositories owned by the bpmn-io team

* `gh api orgs/bpmn-io/repos --paginate` — full repo list with descriptions and archive status
* `gh api orgs/camunda/teams/modeling/repos --paginate` — Camunda Modeling team repo list
* `gh api repos/<org>/<repo>/contents/package.json` — `name`, `dependencies`, `peerDependencies` per repo
* `gh api repos/<org>/<repo>/readme` — README for key repos (API surface, distributions, variants)
* `gh api repos/<org>/<repo> -q '.archived'` — to filter out archived repos

## Source: Lock file analysis (finding blind spots)

Cross-reference found repositories against the two modeler [lock](https://github.com/camunda/camunda-modeler/blob/develop/package-lock.json) [files](https://github.com/camunda/camunda-hub/blob/main/frontend/package-lock.json) to catch them:

```bash
gh api repos/camunda/camunda-modeler/contents/package-lock.json \
  -H "Accept: application/vnd.github.raw" | node -e "
const d = JSON.parse(require('fs').readFileSync('/dev/stdin', 'utf8'));
Object.keys(d.packages || {})
  .filter(p => p.startsWith('node_modules/'))
  .map(p => p.replace('node_modules/', ''))
  .filter(n => n.startsWith('@bpmn-io/') || n.startsWith('@camunda/') ||
               /^(bpmn-|diagram-|dmn-|camunda-|zeebe-)/.test(n))
  .sort().forEach(n => console.log(n));
"
```

Run the same against `repos/camunda/camunda-hub/contents/frontend/package-lock.json` for Web Modeler. Packages appearing in both are high-priority candidates.

## Instructions

1. Re-run the repo list commands above and diff against `repositories.json` to find additions/removals.
2. Run the lock file analysis above and compare against `repositories.json` — packages in both modeler lock files that have no entry are blind spots.
3. Check archive status of any new or unfamiliar repos before adding them.
4. For new repos, fetch `package.json` to confirm the npm package name and where it fits in the layer model.
5. Update both `SKILL.md` (tables + dependency graph if the layer model shifts) and `repositories.json` in sync.
