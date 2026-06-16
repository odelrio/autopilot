# Source binding — GitHub Issues (via `gh`)

Resolves the roadmap-source verbs against GitHub Issues using the GitHub CLI (`gh`). Work
items are issues; a tracking issue (or a milestone) plays the role of the roadmap log.
Dependency order comes from the `## Queue` in `roadmap.config.md`, not from GitHub.

| Verb           | Command                                                                                          |
| -------------- | ----------------------------------------------------------------------------------------------- |
| `next-ready`   | Read `## Queue` order; cross-check open/closed with `gh issue list --state open --label roadmap`; pick the first whose dependency issues are all closed. |
| `list-open`    | `gh issue list --state open --label roadmap --json number,title` — every open roadmap issue, for the drift check (not filtered by the queue). |
| `claim`        | `gh issue edit <N> --add-assignee @me --add-label in-progress`                                   |
| `complete`     | `gh issue close <N> --reason completed`                                                          |
| `park`         | `gh issue edit <N> --remove-label in-progress` + a `note` explaining why.                        |
| `note`         | `gh issue comment <N> --body "..."`                                                              |
| `log-decision` | `note` on the tracking issue `<TRACK>` — one comment line per decision.                          |
| `derive`       | `gh issue create --title "..." --body "..." --label roadmap` (link it from `<TRACK>`).           |

## Notes

- GitHub Issues have no first-class "dependency" field; the dependency graph lives in the
  `## Queue` of `roadmap.config.md`. Keep that queue current as items close — the engine's drift
  check (`list-open`) flags roadmap issues opened after the queue was written so they don't go
  unnoticed.
- Use a label (e.g. `in-progress`) plus self-assignment as the `claim` signal so a parallel
  fleet can see what is taken.
- The **per-roadmap** session marker (one solo/fleet run per roadmap at a time) is a marker
  comment on the tracking issue `<TRACK>` (`FLEET-SESSION open`/`closed`), or a label on it:
  check it before starting, post on start, close in the digest. Anchored to this roadmap's
  tracking issue, so a different roadmap's run is unaffected.
- Bodies are plain Markdown — no format conversion needed.
