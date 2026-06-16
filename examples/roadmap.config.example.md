# roadmap.config.md (example)

> Drop this file at your repository root as `roadmap.config.md` and replace the placeholders.
> This example wires Jira as the source and GitHub as the code-host. Swap the bindings for
> the ones under `examples/bindings/` that match your stack. See `docs/config-schema.md` for
> the full schema and the verb contract.
>
> Running **several roadmaps** in one repo? This file becomes the shared **base** (host, verify,
> review, conventions) and each epic gets an overlay — see `examples/roadmaps/TICKET-101.md` and
> `docs/config-schema.md` → *One roadmap or several*.

## Source binding

Jira via `acli` (see `examples/bindings/jira.md`). Epic `ACME-100` is the roadmap; its
comments are the decision log.

| Verb           | Command                                                                                   |
| -------------- | ----------------------------------------------------------------------------------------- |
| `next-ready`   | First item in `## Queue` order whose deps are `Done`; cross-check with `acli jira workitem list --jql "parent = ACME-100 AND status = 'To Do'"`. |
| `list-open`    | `acli jira workitem list --jql "parent = ACME-100 AND statusCategory != Done"` (drift check). |
| `claim`        | `acli jira workitem transition --key <KEY> --status "In Progress"`                         |
| `complete`     | `acli jira workitem transition --key <KEY> --status "Done"`                                |
| `park`         | transition to `"To Do"` + `note` the reason                                               |
| `note`         | `acli jira workitem comment create --key <KEY> --body-file <adf.json>`                     |
| `log-decision` | `note` on `ACME-100`                                                                       |
| `derive`       | `acli jira workitem create --parent ACME-100 --type Sub-task --summary "..." --description-file <adf.json>` |

## Code-host binding

GitHub via `gh` (see `examples/bindings/github.md`).

| Verb         | Command                                                                          |
| ------------ | -------------------------------------------------------------------------------- |
| `branch`     | `git fetch origin && git switch -c <type>/<KEY>-<slug> origin/main`              |
| `push`       | `git push -u origin <branch>`; confirm `HEAD` == `@{u}`                          |
| `open-pr`    | `gh pr create --base main --title "<type>(<KEY>): ..." --body-file pr.md`        |
| `comment-pr` | `gh pr comment <N> --body-file review.md`                                        |
| `merge`      | `gh pr merge <N> --squash --delete-branch`; then `git switch main && git pull --prune` |

**Conflict protocol:** regenerate generated artifacts (don't hand-merge); rename migrations
to the head's next timestamp; union i18n keys; re-run full `verify` after any source conflict.

## Verify

```bash
make ci    # lint + build + tests; the merge gate
```

`make ci` diffs against committed state — **commit before running it**.

## Review ritual

On every PR, in order, each posted via `comment-pr` *before* any prose:

1. A cheap second-opinion review on the diff.
2. A deep code review (judgment model) on the diff + spec.
3. A security review on the branch — **mandatory** when the diff touches `backend/` or
   shared code; for frontend, only when it handles auth, tokens, uploads, or payments.
   Open high-severity findings block `merge`.

## Conventions

- Branch: `feat/ACME-1-summary`, `fix/…`, `chore/…`, `refactor/…`.
- Commit & PR title: `feat(ACME-1): description`.
- PR body uses the repo's template; link the work item in the footer.
- No AI-tooling mentions or co-author attribution in any artifact. All artifacts in English.

## Reserved decisions

Reserved to a human (build around them; tag the item `PENDING-OWNER` and continue):

- The definitive product name.
- Anything requiring the owner's accounts, credentials, or signatures.

## Spec gate

Required for `backend/` and shared-package surfaces only: write the `.spec.md` with test
links before code; self-review for placeholders, contradictions, and untestable clauses.
Frontend is exempt. (Delete this whole section if your project has no spec gate.)

## Queue

Dependency order; `→` is "then", `∥` is "parallelizable". Lanes for the fleet are by surface.

```
ACME-1 (schema) → ACME-2 (api) ∥ ACME-3 (web-onboarding)
→ ACME-4 (matching) → ACME-5 (safety)
→ [cloud wall — needs owner credentials: ACME-9, ACME-10]
```

**Fleet lanes (disjoint surfaces, max 3 concurrent):**

- `backend` — anything touching `backend/` or shared code. Max one in flight.
- `frontend` — webapp-only.
- `ops/web` — backoffice-only or website-only.

## Canonical docs

When a decision is ambiguous, defer to these (they replace the human stakeholder):

- `docs/PRODUCT.md`, `docs/DATA_MODEL.md`, and the team conventions file.
- On conflict or silence, choose the most operational, simplest interpretation and
  `log-decision`.

## Environment gotchas

Try these first; don't rediscover them:

- Toolchain init: export `JAVA_HOME` and prepend it to `PATH` before any `gradle`/`make`.
- Container tests reach Docker via a specific socket — restore `~/.testcontainers.properties`
  if every container test fails fast.
- DB on a non-default port (e.g. `5433`); migration history drift → run the repair target
  before migrating.
