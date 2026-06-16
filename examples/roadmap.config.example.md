# roadmap.config.md (example — the project base)

> Drop this file at your repository root as `roadmap.config.md`. It is the shared **base**:
> project-wide **plumbing** (code host, verify gate, review ritual, conventions) that every
> initiative reuses — it carries **no** `## Source binding` or `## Queue` of its own. Each
> initiative is an overlay `roadmaps/<id>.md` (its own source + queue); see
> `examples/roadmaps/TICKET-101.md`.
>
> This example wires GitHub as the code-host. Swap the binding for the one under
> `examples/bindings/` that matches your stack. Full schema and verb contract:
> `docs/config-schema.md` → *The model*.

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

- Branch: tracker-backed → `feat/ACME-1-summary` (`fix/…`, `chore/…`, `refactor/…`); slug-named
  roadmap (no tracker) → `feat/<roadmap-slug>/<item-slug>`.
- Commit & PR title: `feat(ACME-1): description`.
- PR body uses the repo's template; link the work item in the footer.
- No AI-tooling mentions or co-author attribution in any artifact. All artifacts in English.

## Spec gate

Required for `backend/` and shared-package surfaces only: write the `.spec.md` with test
links before code; self-review for placeholders, contradictions, and untestable clauses.
Frontend is exempt. (Delete this whole section if your project has no spec gate.)

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
