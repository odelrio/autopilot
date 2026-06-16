# Source binding — Markdown checklist (no tracker)

The lightest possible roadmap source: a checklist in a Markdown file, versioned in the repo.
No external tracker, no API. Good for solo projects and for testing the engine's selection
logic in isolation.

Assume a `ROADMAP.md` whose items look like:

```markdown
- [ ] T1 — Set up the database schema  (deps: —)
- [ ] T2 — Seed reference data         (deps: T1)
- [x] T0 — Repo scaffolding            (deps: —)
```

`[x]` = done, `[ ]` = open. `(deps: …)` names prerequisite item ids (or `—` for none).

| Verb           | Resolution                                                                                       |
| -------------- | ------------------------------------------------------------------------------------------------ |
| `next-ready`   | The first `[ ]` item whose every listed dependency id is `[x]`. (Pure text scan — no tools.)      |
| `list-open`    | Every `[ ]` item id in the file. (For a Markdown source the file *is* the queue, so drift is usually empty — it only shows up if you keep the checklist and `## Queue` as separate lists.) |
| `claim`        | No-op, or append ` (claimed)` to the line. A single-writer checklist rarely needs claiming.        |
| `complete`     | Flip the item's `[ ]` to `[x]`, commit `ROADMAP.md`.                                              |
| `park`         | Leave the item `[ ]`; append ` — parked: <reason>` to the line.                                   |
| `note`         | Append a sub-bullet under the item.                                                               |
| `log-decision` | Append a line to a `## Decisions` section at the bottom of `ROADMAP.md`.                          |
| `derive`       | Add a new `- [ ]` item with its deps.                                                             |

## Notes

- This binding is the engine's reference case for `next-ready`: dependency-gated selection
  over a flat list, with zero provider coupling. The agnosticism dry-run in the README uses
  exactly this binding.
- Because the source is a file in the repo, `complete`/`park`/`note` are ordinary commits —
  fold them into the task's PR or a tiny bookkeeping commit.
- The **per-roadmap** session marker (one solo/fleet run per roadmap at a time) is a line in
  the checklist file (e.g. `<!-- SESSION: open -->` / `closed`): check it before starting, post
  on start, close in the digest. With one checklist per roadmap the marker is naturally
  per-roadmap; a different roadmap's file is untouched.
