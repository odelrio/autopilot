# Code-host binding — GitLab (via `glab` + `git`)

Resolves the code-host verbs against GitLab merge requests using the GitLab CLI (`glab`).
Copy into your `roadmap.config.md` under `## Code-host binding`.

| Verb         | Command                                                                                          |
| ------------ | ------------------------------------------------------------------------------------------------ |
| `branch`     | `git fetch origin && git switch -c <type>/<KEY>-<slug> origin/main`                               |
| `push`       | `git push -u origin <branch>`, then confirm: `git rev-parse HEAD` == `git rev-parse @{u}`.        |
| `open-pr`    | `glab mr create --source-branch <branch> --target-branch main --title "..." --description-file mr.md` |
| `comment-pr` | `glab mr note <IID> --message "..."` (post review evidence here before any prose).                |
| `merge`      | `glab mr merge <IID> --squash --remove-source-branch`, then `git switch main && git pull --prune`. |
| `reconcile`  | *(integration-branch roadmaps only)* `git switch main && git pull`, `git merge --no-ff integration/<ID>`, run the full `verify` gate, resolve any conflict via the protocol below, then `git push origin main`. Run once at mission-complete. |

## Integration branch (concurrent roadmaps only)

A roadmap meant to run **alongside another** points its code-host base at its own integration
branch instead of `main`, so the two never share a merge queue. In the overlay's
`## Code-host binding`, replace the `main` token: `branch` from `origin/integration/<KEY>`,
`open-pr --target-branch integration/<KEY>`, `merge` into it. Define `reconcile` (row above) for
the final `integration/<KEY>` → `main` merge, and merge `main` forward into the integration
branch periodically (`git switch integration/<KEY> && git merge origin/main`) so that final
merge stays small. A roadmap that runs alone keeps `main` and omits all of this.

## Conflict protocol (for the merge queue)

Identical in spirit to the GitHub binding — adapt to your project:

- **Generated artifacts**: regenerate, never hand-merge.
- **Ordered/timestamped files**: rename to the current head's next slot.
- **Additive data**: union keys, then re-verify.
- **Source conflicts**: resolve, then re-run the full `verify` gate.
- Resolving that would override a logged decision: stop and `park`.

## Notes

- GitLab MRs use `IID` (project-scoped id), not the global id — pass the IID to `glab mr`.
- Confirm acceptance after every `push`; the same rejected-push-then-squash hazard applies.
- If your project gates merges on GitLab CI rather than a local `verify`, the merge-queue
  step "run `verify`" maps to "wait for the pipeline to pass on the rebased branch" — keep it
  in the foreground per the standards.
