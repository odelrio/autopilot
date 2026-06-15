# Source binding — Jira (via `acli`)

Resolves the roadmap-source verbs against a Jira project using the Atlassian CLI (`acli`).
Copy the verb table into your `roadmap.config.md` under `## Source binding` and adjust the
project key, epic key, and transitions to your workflow.

Assumes work items are subtasks of one parent epic (the "roadmap"), and the epic's comments
double as the decision log.

| Verb           | Command                                                                                              |
| -------------- | --------------------------------------------------------------------------------------------------- |
| `next-ready`   | Read the `## Queue` order in `roadmap.config.md`; cross-check status with `acli jira workitem list --jql "parent = <EPIC> AND status = 'To Do'"`; pick the first whose dependencies are all `Done`. |
| `claim`        | `acli jira workitem transition --key <KEY> --status "In Progress"`                                   |
| `complete`     | `acli jira workitem transition --key <KEY> --status "Done"`                                          |
| `park`         | `acli jira workitem transition --key <KEY> --status "To Do"` + a `note` explaining why.              |
| `note`         | `acli jira workitem comment create --key <KEY> --body-file <adf.json>` (the `create` subcommand is mandatory; body is ADF JSON). |
| `log-decision` | `note` on the epic key `<EPIC>` — one comment line per decision.                                     |
| `derive`       | `acli jira workitem create --parent <EPIC> --type Sub-task --summary "..." --description-file <adf.json>`. |

## Notes

- Descriptions and comments use **ADF** (Atlassian Document Format) JSON, not Markdown.
  Keep a small `md → adf` converter in your repo and pipe through it.
- `next-ready` is queue-driven, not status-driven: the `## Queue` in `roadmap.config.md` is
  the source of dependency order; Jira status only tells you what is already taken or done.
- The fleet's "one open session" guard can be a marker comment on the epic
  (`FLEET-SESSION open`/`closed`) checked before spawning.
