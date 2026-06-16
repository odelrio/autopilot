# roadmaps/TICKET-101.md (example overlay)

> An **epic overlay**. Drop overlays at `roadmaps/<ID>.md` in your repo root when one repo runs
> several roadmaps. It carries only the epic-specific sections; everything project-wide
> (`## Code-host binding`, `## Verify`, `## Review ritual`, `## Conventions`,
> `## Environment gotchas`) is inherited from the base `roadmap.config.md`. Run this epic with
> `/autopilot:solo TICKET-101` or `/autopilot:fleet TICKET-101`. See `docs/config-schema.md` →
> *One roadmap or several*.

## Source binding

Jira via `acli` (see `examples/bindings/jira.md`). Epic `TICKET-101` is this roadmap; its
comments are the decision log.

| Verb           | Command                                                                                   |
| -------------- | ----------------------------------------------------------------------------------------- |
| `next-ready`   | First item in `## Queue` order whose deps are `Done`; cross-check with `acli jira workitem list --jql "parent = TICKET-101 AND status = 'To Do'"`. |
| `list-open`    | `acli jira workitem list --jql "parent = TICKET-101 AND statusCategory != Done"` (drift check). |
| `claim`        | `acli jira workitem transition --key <KEY> --status "In Progress"`                         |
| `complete`     | `acli jira workitem transition --key <KEY> --status "Done"`                                |
| `park`         | transition to `"To Do"` + `note` the reason                                               |
| `note`         | `acli jira workitem comment create --key <KEY> --body-file <adf.json>`                     |
| `log-decision` | `note` on `TICKET-101`                                                                        |
| `derive`       | `acli jira workitem create --parent TICKET-101 --type Sub-task --summary "..." --description-file <adf.json>` |

## Queue

Dependency order; `→` is "then", `∥` is "parallelizable". Lanes for the fleet are by surface.

```
TICKET-102 (schema) → TICKET-103 (api) ∥ TICKET-104 (web)
→ TICKET-105 (rollout)
```

**Fleet lanes (disjoint surfaces, max 3 concurrent):**

- `backend` — anything touching `backend/` or shared code. Max one in flight.
- `frontend` — webapp-only.

## Reserved decisions

Reserved to a human for this epic (build around them; tag the item `PENDING-OWNER` and continue):

- Anything requiring the owner's accounts, credentials, or signatures.

<!-- Inherits ## Canonical docs and (if any) ## Spec gate from the base roadmap.config.md.
     Add either section here only to override the base for this epic. -->

<!-- OPTIONAL — run this epic CONCURRENTLY with another roadmap. Override the base's
     `## Code-host binding` so this epic's PRs target their own integration branch instead of
     mainline; its serial merge queue then never shares a branch with the other roadmap, and a
     single `reconcile` merges to mainline at epic completion. See `examples/bindings/github.md`
     → *Integration branch* and `docs/config-schema.md` → *Concurrent roadmaps and integration
     branches*. Omit this whole section to run on mainline like a solo roadmap.

## Code-host binding

| Verb        | Command                                                                                   |
| ----------- | ----------------------------------------------------------------------------------------- |
| `branch`    | `git fetch origin && git switch -c <type>/<KEY>-<slug> origin/integration/TICKET-101`         |
| `push`      | `git push -u origin <branch>`, then confirm `git rev-parse HEAD` == `git rev-parse @{u}`.  |
| `open-pr`   | `gh pr create --base integration/TICKET-101 --title "<type>(<KEY>): ..." --body-file pr.md`   |
| `comment-pr`| `gh pr comment <N> --body-file review.md`                                                  |
| `merge`     | `gh pr merge <N> --squash --delete-branch`                                                  |
| `reconcile` | `git switch main && git pull`, `git merge --no-ff integration/TICKET-101`, run `verify`, then `git push origin main`. Once, at mission-complete. |
-->

