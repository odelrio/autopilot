# `roadmap.config.md` — configuration schema

The autopilot engine is project-agnostic. It speaks in **abstract verbs**; your project's
`roadmap.config.md`, placed at the consuming repository's root, binds those verbs to real
commands and supplies the project's conventions, gates, and queue.

The engine reads this file at the start of every run. Anything the engine needs that is
specific to your project lives here — and **only** here. Keep tool names (`gh`, `acli`,
`glab`, `make`, …) out of the engine; they belong in this file.

## The verb contract

The engine never names a provider. It emits these verbs, and your bindings resolve them.

### Roadmap-source verbs (where work items live)

| Verb           | Meaning                                                                 |
| -------------- | ----------------------------------------------------------------------- |
| `next-ready`   | The next workable item: first in queue order whose dependencies are all done and whose surface fits the lane. |
| `claim`        | Mark an item as owned/in-progress so no other lane takes it.            |
| `complete`     | Mark an item done.                                                      |
| `park`         | Return an item to the backlog with a reason (blocked, failed, deferred).|
| `note`         | Attach a freeform comment to an item.                                   |
| `log-decision` | Record a durable decision on the roadmap log (the parent epic/index).   |
| `derive`       | Create a new follow-up item discovered during the work.                 |

### Code-host verbs (where code review and merge happen)

| Verb          | Meaning                                                    |
| ------------- | ---------------------------------------------------------- |
| `branch`      | Create a working branch from the fresh base.               |
| `push`        | Publish the branch; the engine then confirms acceptance.   |
| `open-pr`     | Open a pull/merge request from the branch.                 |
| `comment-pr`  | Post a comment (review evidence) on the PR.                |
| `merge`       | Merge the PR into the base and clean up the branch.        |

## Required sections of `roadmap.config.md`

Write it as Markdown with these sections. Headings are conventional; the engine reads the
content, so be explicit and literal — give exact commands, not descriptions of commands.

1. **`## Source binding`** — one resolution per roadmap-source verb. The concrete command or
   procedure for `next-ready`, `claim`, `complete`, `park`, `note`, `log-decision`, `derive`.
   See `examples/bindings/jira.md`, `github-issues.md`, or `markdown.md`.

2. **`## Code-host binding`** — one resolution per code-host verb (`branch`, `push`,
   `open-pr`, `comment-pr`, `merge`), plus the **conflict protocol**: which artifacts are
   regenerated rather than hand-merged, how migration/ordering collisions are resolved, and
   which data files are unioned. See `examples/bindings/github.md` or `gitlab.md`.

3. **`## Verify`** — the exact gate command(s) that must pass before review and before merge,
   and any environment setup they need (toolchain paths, service sockets, ports). This is the
   project's merge gate. Include the "diffs against committed state" caveat if it applies.

4. **`## Review ritual`** — which reviewers run on each PR, in what order, and which are
   mandatory for which surfaces (e.g. a security review mandatory on backend diffs). Restate
   that every review is posted via `comment-pr` before any prose (the engine enforces this,
   but naming the reviewers here makes it concrete).

5. **`## Conventions`** — branch naming, commit message format, PR template, and any
   house rules (no AI attribution, language, etc.).

6. **`## Reserved decisions`** — the classes of decision reserved to a human, how to tag a
   pending item, and what the engine builds around them. Empty is allowed.

7. **`## Spec gate`** *(optional)* — if the project requires a spec before code (e.g. SDD),
   name the surfaces it applies to and the self-review checklist. Omit the section entirely
   to disable the gate.

8. **`## Queue`** — the dependency-ordered list of work, the lane surfaces for the fleet
   (default: by code surface), and any external "walls" (work blocked on credentials/infra).
   The fleet derives its lanes from the surfaces named here.

9. **`## Canonical docs`** — the documents that stand in for a human stakeholder when the
   engine must decide. The engine defers to these on ambiguity.

10. **`## Environment gotchas`** *(optional)* — project-specific setup traps the engine
    should try first rather than rediscover (toolchain init, sockets, ports, repair
    commands).

## How the engine consumes it

- `/autopilot:solo` and `/autopilot:fleet` read `## Queue`, `## Reserved decisions`,
  `## Canonical docs`, and (fleet) the lane surfaces.
- `autopilot:task` reads every section to run one item end-to-end.
- `autopilot:standards` is **not** configured here — it is the fixed discipline layer. The
  config can override a specific standard by saying so explicitly, but the defaults hold
  otherwise.

Start from `examples/roadmap.config.example.md` and replace each binding with your project's
tools.
