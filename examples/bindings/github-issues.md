# Source binding — GitHub Issues (via `gh`)

Resolves the roadmap-source verbs against GitHub Issues using the GitHub CLI (`gh`). Work
items are issues; a tracking issue (or a milestone) plays the role of the roadmap log.
Dependency order comes from the `## Queue` in `roadmap.config.md`, not from GitHub.

| Verb           | Command                                                                                          |
| -------------- | ----------------------------------------------------------------------------------------------- |
| `next-ready`   | Read `## Queue` order; cross-check open/closed with `gh issue list --state open --label roadmap`; pick the first whose dependency issues are all closed. |
| `claim`        | `gh issue edit <N> --add-assignee @me --add-label in-progress`                                   |
| `complete`     | `gh issue close <N> --reason completed`                                                          |
| `park`         | `gh issue edit <N> --remove-label in-progress` + a `note` explaining why.                        |
| `note`         | `gh issue comment <N> --body "..."`                                                              |
| `log-decision` | `note` on the tracking issue `<TRACK>` — one comment line per decision.                          |
| `derive`       | `gh issue create --title "..." --body "..." --label roadmap` (link it from `<TRACK>`).           |

## Notes

- GitHub Issues have no first-class "dependency" field; the dependency graph lives in the
  `## Queue` of `roadmap.config.md`. Keep that queue current as items close.
- Use a label (e.g. `in-progress`) plus self-assignment as the `claim` signal so a parallel
  fleet can see what is taken.
- Bodies are plain Markdown — no format conversion needed.
