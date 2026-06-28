# Code-host binding — GitHub (via `gh` + `git`)

Resolves the code-host verbs against GitHub. Copy into your `roadmap.config.md` under
`## Code-host binding`.

| Verb         | Command                                                                                          |
| ------------ | ------------------------------------------------------------------------------------------------ |
| `branch`     | `git fetch origin && git switch -c <type>/<KEY>-<slug> origin/main`                               |
| `push`       | `git push -u origin <branch>`, then confirm: `git rev-parse HEAD` == `git rev-parse @{u}`.        |
| `open-pr`    | `gh pr create --base main --title "<type>(<KEY>): ..." --body-file pr.md`                         |
| `comment-pr` | `gh pr comment <N> --body-file review.md` (post review evidence here before any prose).           |
| `merge`      | `gh pr merge <N> --squash --delete-branch`, then `git switch main && git pull --prune`.           |
| `reconcile`  | *(integration-branch roadmaps only)* `git switch main && git pull`, `git merge --no-ff integration/<ID>`, run the full `verify` gate, resolve any conflict via the protocol below, then `git push origin main`. Run once at mission-complete. |

## Integration branch (concurrent roadmaps only)

A roadmap meant to run **alongside another** points its code-host base at its own integration
branch instead of `main`, so the two never share a merge queue. In the overlay's
`## Code-host binding`, replace the `main` token: `branch` from `origin/integration/<KEY>`,
`open-pr --base integration/<KEY>`, `merge` into it (`gh pr merge` already targets the PR's
base). Define `reconcile` (row above) for the final `integration/<KEY>` → `main` merge, and
merge `main` forward into the integration branch periodically (`git switch integration/<KEY> &&
git merge origin/main`) so that final merge stays small. A roadmap that runs alone keeps `main`
and omits all of this.

## Conflict protocol (for the merge queue)

State here how the rebase-before-merge resolves collisions. Typical rules:

- **Generated artifacts** (OpenAPI snapshots, generated clients, lockfiles): never hand-merge.
  Take the base's version and re-run the generator; commit the regenerated output.
- **Ordered/timestamped files** (e.g. DB migrations): rename yours to the current head's
  next slot per the project's ordering rule.
- **Additive data** (translation catalogs, fixtures keyed by id): union both sides' keys,
  then re-verify completeness.
- **Source conflicts**: resolve with judgment, then re-run the **full** `verify` gate.
- If resolving would override a logged decision or undo another merged PR's approach: stop,
  `park` the task with a note, take the next one.

## Push-policy notes

- Confirm acceptance after every `push` (remote tip == local tip). A rejected non-fast-forward
  push followed by a squash-merge silently drops commits.
- If your environment denies force-push from the main checkout, use the merge-commit route
  (reset to remote, merge base, resolve, plain push) instead of retrying a denied force-push.
