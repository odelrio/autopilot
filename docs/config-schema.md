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

## The model: a base + initiative overlays

Work is organized as **initiatives**, and an initiative is the normal unit you run. The config
splits in two:

- **`roadmap.config.md` (repo root) — the project base.** Shared **plumbing** every initiative
  reuses: `## Code-host binding`, `## Verify`, `## Review ritual`, `## Conventions`,
  `## Environment gotchas`, a default `## Canonical docs`, and an optional project-wide
  `## Spec gate`. It carries **no** `## Source binding` or `## Queue` — those are per-initiative.
- **`roadmaps/<id>.md` — an initiative overlay.** Holds what is specific to one initiative:
  `## Source binding` and `## Queue` (both **required**), plus `## Reserved decisions`, and
  optional overrides of `## Canonical docs`, `## Spec gate`, `## Conventions`, and — when the
  initiative runs concurrently on its own integration branch — `## Code-host binding` (to retarget
  `branch`/`open-pr`/`merge` at that branch; see *Concurrent roadmaps and integration branches*
  below). `<id>` is both the filename stem and the argument you pass to `/autopilot:solo`,
  `/autopilot:fleet`, and `/autopilot:init`. It is a **tracker key** when one exists, or a short
  **kebab-case slug** that `init` derives from your intent when there is no ticket
  (`/autopilot:init redesign onboarding` → `roadmaps/onboarding-redesign.md`); `init` always
  confirms the resolved id before writing. When the initiative's source is a file-based one
  (Markdown checklist / spec files, no tracker), its checklist lives beside the overlay at
  `roadmaps/<id>.ROADMAP.md`.

A repo accumulates as many initiatives as it needs: `/autopilot:init` sets up the base **once**
(reusing it thereafter) and scaffolds each initiative as an overlay.

**Composition is section-level override.** The *effective config* for an initiative is the base,
with each top-level `##` section present in the overlay **replacing** the base section of the same
name. Sections only in the base are inherited; sections only in the overlay are added. A flat
replace, not a deep merge — to know which value wins, ask only "is this section in the overlay?".

**Backward compatible (legacy single roadmap).** A `roadmap.config.md` that itself carries
`## Source binding` + `## Queue` (with no `roadmaps/` directory) still runs as a single roadmap —
the engine reads it exactly as before. New setups scaffold initiatives as overlays; this legacy
shape keeps existing configs working.

### Concurrent roadmaps and integration branches

Two roadmaps may run at the same time — two agents, one roadmap each. (Never two agents on one
roadmap; never two roadmaps in one invocation — both share a queue.) To keep their merges from
racing on a shared mainline, give each concurrently-run roadmap its own **integration branch**
and point its `## Code-host binding` there: `branch` from `origin/integration/<ID>`, `open-pr
--base integration/<ID>`, `merge` into `integration/<ID>`. Each roadmap's serial merge queue
then operates **only on its own branch**, so concurrent fleets cannot collide — the isolation is
structural, not a matter of lane discipline.

A roadmap on an integration branch defines one extra code-host verb, **`reconcile`**: at
mission-complete it merges the integration branch into the repo's mainline base, once, behind
the full `verify` gate and the conflict protocol. To keep that final merge small, the engine
also merges mainline **forward** into the integration branch periodically while the roadmap runs
(the fleet whenever a lane drains, solo at the start of each run). A roadmap that runs alone
needs none of this: it targets the mainline base directly and has no `reconcile`.

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
| `reconcile`   | *(integration-branch roadmaps only)* Merge this roadmap's integration branch into the repo's mainline base — once, at mission-complete, behind the full `verify` gate and the conflict protocol. |

## Required sections of `roadmap.config.md`

Write it as Markdown with these sections. Headings are conventional; the engine reads the
content, so be explicit and literal — give exact commands, not descriptions of commands. They
split between the **base** (plumbing) and each **initiative overlay** as described in *The model*
above: `## Source binding` and `## Queue` live in the overlay; the
host/verify/review/conventions/canonical-docs sections stay in the base. (A legacy single-roadmap
`roadmap.config.md` carries all of them in one file.)

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
   house rules (no AI attribution, language, etc.). Branch naming has two forms: a tracker-backed
   roadmap uses `<type>/<ITEM-KEY>-<slug>`; a slug-named roadmap (no tracker) groups its items
   under the roadmap slug as `<type>/<roadmap-slug>/<item-slug>`.

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

### The session marker (one run per roadmap)

`/autopilot:solo` and `/autopilot:fleet` run **at most one session per roadmap** at a time: two
agents may drive two *different* roadmaps concurrently, but never the same one (they would share
its queue). Each enforces this with a **session marker** anchored to that roadmap's own source —
a small open/closed signal the source binding provides (e.g. a `FLEET-SESSION open`/`closed`
comment on the epic, a label, or a line in the checklist). The engine checks it before starting,
posts it on start, and closes it in the end-of-run digest. It is **per-roadmap**, never a
repo-wide lock — see each binding in `examples/bindings/` for where it lives.

### Selecting a roadmap

`/autopilot:solo [id]` and `/autopilot:fleet [id]` resolve which initiative to run:

- **`id` given, `roadmaps/<id>.md` exists** → run base ⊕ that overlay.
- **`id` given, no such overlay** → stop and list the available initiatives.
- **No `id`:** exactly one overlay in `roadmaps/` → use it; more than one → stop and list them;
  none → fall back to the legacy single-roadmap root `roadmap.config.md` (if it carries a queue).

Once an overlay is selected, it **must** contain the required overlay sections (`## Source
binding` and `## Queue`). If either is missing, that is a config error: stop and report which
section the overlay lacks rather than silently falling back to the base.

This keeps `solo`/`fleet` non-interactive: an ambiguous launch fails fast with the valid IDs,
the same way a missing config does.

Start from `examples/roadmap.config.example.md` (the project base / plumbing) and
`examples/roadmaps/TICKET-101.md` (an initiative overlay). Replace each binding with your tools.
