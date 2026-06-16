# `roadmap.config.md` — configuration schema

The autopilot engine is project-agnostic. It speaks in **abstract verbs**; your project's
`roadmap.config.md`, placed at the consuming repository's root, binds those verbs to real
commands and supplies the project's conventions, gates, and queue.

The engine reads this file at the start of every run. Anything the engine needs that is
specific to your project lives here — and **only** here. Keep tool names (`gh`, `acli`,
`glab`, `make`, …) out of the engine; they belong in this file.

> The fastest way to produce this file is **`/autopilot:init`**, which autodetects your stack
> and writes most of it for you. This document explains every section so you can edit it — or
> write it from scratch — by hand.

## One roadmap or several: base ⊕ overlay

A repository may run **one** roadmap or **several** independent ones (one per
initiative/epic). The config splits accordingly:

- **`roadmap.config.md` (repo root) — the project base.** Holds what every roadmap in the repo
  shares: `## Code-host binding`, `## Verify`, `## Review ritual`, `## Conventions`,
  `## Environment gotchas`, and a default `## Canonical docs`.
- **`roadmaps/<ID>.md` — an epic overlay** *(optional)*. Holds what is specific to one
  initiative: `## Source binding` and `## Queue` (both **required** in an overlay), plus
  `## Reserved decisions`, and optional overrides of `## Canonical docs`, `## Spec gate`, and
  `## Conventions`. `<ID>` (e.g. `BRU-101`) is both the filename stem and the argument you pass
  to `/autopilot:solo`, `/autopilot:fleet`, and `/autopilot:init`. When the overlay's source is
  the Markdown checklist (no tracker), that epic's checklist lives beside it at
  `roadmaps/<ID>.ROADMAP.md` — the per-epic counterpart of a single-roadmap repo's root
  `ROADMAP.md`.

**Composition is section-level override.** The *effective config* for a roadmap is the base,
with each top-level `##` section present in the overlay **replacing** the base section of the
same name. Sections only in the base are inherited; sections only in the overlay are added.
This is a flat replace, not a deep merge — to know which value wins, ask only "is this section
in the overlay?".

**Backward compatible.** With no `roadmaps/` directory, the root `roadmap.config.md` is itself
the single roadmap and carries every section, exactly as before. Overlays are purely additive:
adopt them only when a repo needs more than one roadmap.

## The verb contract

The engine never names a provider. It emits these verbs, and your bindings resolve them.

### Roadmap-source verbs (where work items live)

| Verb           | Meaning                                                                 |
| -------------- | ----------------------------------------------------------------------- |
| `next-ready`   | The next workable item: first in queue order whose dependencies are all done and whose surface fits the lane. |
| `list-open`    | Enumerate the open (not-done) work items currently under this roadmap's scope in the source, independent of the `## Queue`. Powers the drift check (see *How the engine consumes it*). |
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
content, so be explicit and literal — give exact commands, not descriptions of commands. In a
single-roadmap repo all of them live in `roadmap.config.md`. In a multi-roadmap repo they split
between the base and each overlay as described in **One roadmap or several** above —
`## Source binding` and `## Queue` move into the overlay; the host/verify/review/conventions
sections stay in the base.

1. **`## Source binding`** — one resolution per roadmap-source verb. The concrete command or
   procedure for `next-ready`, `list-open`, `claim`, `complete`, `park`, `note`,
   `log-decision`, `derive`. See `examples/bindings/jira.md`, `github-issues.md`, or
   `markdown.md`.

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

### Drift between the queue and the source

The `## Queue` is owner-maintained: it encodes dependency order and lane surfaces, which the
source (Jira, GitHub Issues, a checklist) does not. So `next-ready` is **queue-driven** — it
never works an item the queue doesn't list. That leaves a gap: work added to the source *after*
the queue was written is invisible to the engine.

To close the gap without surrendering the queue's authority, `/autopilot:solo` and
`/autopilot:fleet` run a **drift check** at the start of a run (and the fleet re-checks whenever
a lane drains): `list-open` the source, compare against the `## Queue` ids, and **report** the
difference — items open in the source but absent from the queue (`drift: <ids> not in queue`)
and items in the queue already closed or gone (`drift: <ids> queue-only`). The check is
**detect-and-report only**: it never invents a dependency order or lane, never works an
un-queued item, and never blocks. It surfaces the drift in the run's chat output and digest so
the owner can place the new items in the `## Queue`. If `list-open` fails, it circuit-breaks per
`autopilot:standards` §3 and the run continues from the queue.

### Selecting a roadmap

`/autopilot:solo [ID]` and `/autopilot:fleet [ID]` resolve which roadmap to run:

- **`ID` given, `roadmaps/<ID>.md` exists** → run base ⊕ that overlay.
- **`ID` given, no such overlay** → stop and list the available overlays.
- **No `ID`:** exactly one overlay in `roadmaps/` → use it; more than one → stop and list them;
  none (or no `roadmaps/` directory) → use the root `roadmap.config.md` as the single roadmap.

Once an overlay is selected, it **must** contain the required overlay sections (`## Source
binding` and `## Queue`). If either is missing, that is a config error: stop and report which
section the overlay lacks rather than silently falling back to the base.

This keeps `solo`/`fleet` non-interactive: an ambiguous launch fails fast with the valid IDs,
the same way a missing config does.

Start from `examples/roadmap.config.example.md` (single-roadmap base) and, for a multi-roadmap
repo, `examples/roadmaps/BRU-101.md` (overlay). Replace each binding with your project's tools.
