---
name: init
description: Start a roadmap initiative. Sets up the project's shared base (code host, build gate, conventions) the first time, then scaffolds the initiative as an overlay (its own source + queue) — autodetecting and confirming before writing. The one interactive autopilot command; solo and fleet never ask.
---

# Autopilot init

`init` is how you **start a roadmap initiative** — the normal unit of work. Every initiative is
an **overlay** (`roadmaps/<id>.md`) holding its own `## Source binding` and `## Queue`; the repo's
shared **base** (`roadmap.config.md` — code host, verify gate, review ritual, conventions) is
plumbing that init establishes **once** and every later initiative reuses.

## What init does

One run does up to two things, detecting what it can and confirming the rest:

1. **Ensure the base plumbing exists** — if there is no `roadmap.config.md`, detect the stack and
   write it (§§1, 3–5, 7). If it already exists, leave it untouched and reuse it.
2. **Create the initiative overlay** — resolve the initiative from the argument (below), pick its
   source (§2), and write `roadmaps/<id>.md` (§10). This always happens; it is the point of the run.

(Legacy: a `roadmap.config.md` that itself carries `## Source binding` + `## Queue` still runs as a
single roadmap — the engine reads it — but new initiatives scaffold as overlays.)

## Interpreting the argument

`/autopilot:init [argument]` takes a **free-form** argument describing the initiative — *the more
you say, the fewer questions init asks*. It infers the rest from the repo and this conversation,
and **always proposes what it resolved and confirms before writing** (never names a file from a
guess). The spectrum:

- **Nothing** (`/autopilot:init`) — infer the initiative from the conversation; confirm it, or ask
  what you can't tell. With no argument *and* no discernible intent (and a base already present),
  stop and report rather than guess.
- **A tracker key** (`/autopilot:init TICKET-101`) — a tracker-backed initiative. Infer *which*
  tracker from the repo (Jira / GitHub Issues / GitLab) and confirm; use the key verbatim as the
  overlay id and as the key the source binding substitutes.
- **Free-text intent** (`/autopilot:init redesign the onboarding flow`) — distill it into a short,
  lowercase, kebab-case slug (`onboarding-redesign`; 2–4 words, ASCII, no ticket prefix) as the
  overlay id, and start planning the initiative. A slug-named initiative has no tracker, so its
  source is a file-based one (Markdown checklist, spec files) and its items carry no tracker keys —
  which changes branch naming (see §4).
- **A source description** (`/autopilot:init work items are subtasks of Jira epic TICKET-101 via
  acli; the epic's comments are the log`, or `… the change specs under openspec/changes/, one item
  per change folder`) — the tracker/intent case **plus** a pre-stated source, so init skips §2's
  questions and fills the binding directly. Derive the overlay id from the key or the intent in
  the description.

This is the **only** interactive, human-in-the-loop autopilot command, and the **only** place
besides `examples/bindings/` where naming real tools (`git`, `gh`, `glab`, `make`, …) is
sanctioned: `init` is the setup layer that *produces* the project's binding config by inspecting
the repo, not the runtime engine (`solo`/`fleet`/`task`/`standards`), which stays tool-agnostic.
`/autopilot:solo` and `/autopilot:fleet` never ask the user anything; `init` is setup-time and
explicitly does. Detect what you can, ask for the rest, and **always confirm before writing**.

## Locating the bundled templates

You materialize the project config from the plugin's own packaged files. They live under this
skill's plugin root, **not** in the user's repository, and the install path is versioned (it
changes on every plugin update) — never hardcode it.

- Resolve the plugin root from this skill's **base directory** (the absolute path the harness
  gives you when it loads this skill): `examples/` is two levels up, at `<base-dir>/../../examples/`.
- Fallback: the `${CLAUDE_PLUGIN_ROOT}` environment variable points at the same plugin root, so
  `${CLAUDE_PLUGIN_ROOT}/examples/` also works (try it via Bash if the base-dir path doesn't
  resolve).

The files you read (sources of truth — never edit them, only copy from them):

- `examples/roadmap.config.example.md` — the section structure to follow.
- `examples/bindings/{jira,github-issues,markdown}.md` — roadmap-source verb tables.
- `examples/bindings/{github,gitlab}.md` — code-host verb tables.

## 0. Preconditions

- Must be inside a git work tree (`git rev-parse --is-inside-work-tree`). If not, stop and say so.
- **Resolve the initiative** from the argument (*Interpreting the argument* above). If there is
  neither an argument nor any clear conversational intent **and** a base already exists, stop and
  report (what is set up, and that init needs an initiative to scaffold) — don't guess.
- **Base plumbing:** if `roadmap.config.md` is missing, you will create it (§§1, 3–5, 7) as part
  of this run; if it exists, reuse it untouched. Never overwrite an existing base — regenerate it
  only if the user explicitly asks, showing the diff first.
- **The overlay:** the initiative is written to `roadmaps/<id>.md` (§10). If that file already
  exists, **stop and report** — never overwrite an overlay.

## 1. Detect the code-host

From `git remote get-url origin` (fall back to `git remote -v`):

- host contains `github.com` → **GitHub** (`gh`), use `examples/bindings/github.md`.
- host contains `gitlab` → **GitLab** (`glab`), use `examples/bindings/gitlab.md`.
- no remote / something else → ask the user which of the two to use.

## 2. Detect the initiative's source

This picks the source binding for the **initiative overlay**. If the argument already described
the source (*Interpreting the argument*), use that and skip the questions. Otherwise detect —
repository signals are weak here, so lean on the confirm step:

- A **tracker key** argument → a tracker source; infer which from the remote (`gh` → GitHub
  Issues; GitLab remote → GitLab; otherwise ask — Jira has no repo signal).
- A spec/checklist location the intent names (e.g. `openspec/changes/`, an existing `ROADMAP.md`)
  → **file-based source** (`examples/bindings/markdown.md`, adapted to scan a directory).
- Default when nothing is detected: **Markdown** (zero external tooling). Always let the user pick
  among {Jira, GitHub Issues, Markdown / file-based} at the confirm step.

## 3. Detect the verify gate

Scan the repo root for build manifests, first match wins; propose the command, let the user edit:

- `Makefile` with a `ci:` target → `make ci`
- `package.json` → the present scripts among `lint`, `test`, `build` (e.g. `npm ci && npm run lint && npm test && npm run build`)
- `Cargo.toml` → `cargo clippy --all-targets && cargo test`
- `pyproject.toml` / `tox.ini` / `noxfile.py` → `tox` / `nox` / `pytest`
- `go.mod` → `go vet ./... && go test ./... && go build ./...`
- `gradlew` / `build.gradle` → `./gradlew check`
- `pom.xml` → `mvn verify`
- nothing matches → leave a `<!-- TODO: your lint + build + test gate -->` placeholder.

## 4. Detect base branch & conventions

- Base branch: `git symbolic-ref refs/remotes/origin/HEAD` (fall back to whichever of `main` /
  `master` exists).
- Conventions: **detect the project's own rules before defaulting.** Read the commit/branch
  guidance the project already states (`AGENTS.md`, `CLAUDE.md`, `CONTRIBUTING.md`, a `.gitmessage`
  template) and the shape of recent history (`git log --oneline -20`, the existing branch names);
  adopt that commit-message style, branch-naming scheme, and language. Only when the project says
  nothing, fall back to the engine defaults (Conventional Commits): commit & PR title
  `feat(<ID>): …`, artifacts in the repo's language, no AI-tooling attribution; branch naming in
  two forms — a **tracker-backed** roadmap uses `<type>/<ITEM-KEY>-<slug>` (`feat/`, `fix/`,
  `chore/`, `refactor/`); a **slug-named** roadmap (no tracker — id derived per *Interpreting the
  argument*) groups its items under the roadmap slug as `<type>/<roadmap-slug>/<item-slug>`. The
  user adjusts at confirm.

## 5. Detect canonical-doc candidates

Look for stakeholder-substitute docs and offer a shortlist (don't invent): `docs/PRODUCT.md`,
`ARCHITECTURE.md`, `CONTRIBUTING.md`, `README.md`, other `docs/*.md`.

## 6. Confirm (the "+ confirm")

Show a compact summary: the **resolved initiative id** (and that you'll write `roadmaps/<id>.md`),
its source, whether the base plumbing already exists or you'll create it (and if so: code-host,
verify, base branch, canonical docs), and the overlay sections you'll leave as TODO (`## Queue`,
`## Reserved decisions`). Let the user correct any field — especially the id. Do not write
anything until they confirm.

## 7. Write the base plumbing *(only when `roadmap.config.md` is missing)*

Assemble the shared base `roadmap.config.md` at the repo root — **plumbing only**, no
`## Source binding` and no `## Queue` (those are the initiative's, written to the overlay in §10).
Follow `examples/roadmap.config.example.md`'s section order:

- **`## Code-host binding`** — paste the chosen binding's verb table verbatim from its file (copy,
  don't re-summarize), substituting the detected branch/host, plus the code-host **conflict
  protocol**.
- **`## Verify`**, **`## Review ritual`**, **`## Conventions`**, **`## Canonical docs`** — fill
  from the detections above.
- Optional, project-wide: **`## Spec gate`** (with the "delete this section if you have no spec
  requirement" note) and **`## Environment gotchas`** — add if they apply, else omit.

If the base already exists, skip this step and reuse it untouched.

## 8. Starter checklist (file-based source)

The initiative's starter checklist is written **with the overlay, not the base** — see §10 step 6
(`roadmaps/<id>.ROADMAP.md` for a Markdown/spec source, using the `- [ ] ID — title (deps: …)`
shape from `examples/bindings/markdown.md`). Never overwrite an existing one.

## 9. Report

State the files written — the initiative overlay `roadmaps/<id>.md` (and the base
`roadmap.config.md` if you created it this run) — that they were **committed** (§11, not yet
pushed), and what's left: fill the overlay's TODO sections
(start with `## Queue`), optionally run the README dry-run to sanity-check selection, then run the
initiative with `/autopilot:solo <id>` (single agent) or `/autopilot:fleet <id>` (parallel). Point
to `docs/config-schema.md` for the full schema.

## 10. Write the initiative overlay

This is the point of the run, and it **always** happens — the base (from §§1, 3–5, 7) is now in
place, freshly written or pre-existing. Reuse the base for everything project-wide; do not edit it.

1. **Resolve the initiative id** per *Interpreting the argument* above — a tracker key used
   verbatim, or a slug derived from the argument / conversation (kebab-case, confirmed). The
   overlay path is `roadmaps/<id>.md`. If it already exists, **stop and report** — never overwrite
   an overlay.
2. **Read the base** `roadmap.config.md` to confirm the shared sections (`## Code-host binding`,
   `## Verify`, `## Review ritual`, `## Conventions`) the overlay will inherit. The overlay does
   **not** repeat them.
3. **Pick the source binding for this initiative** (§2): use what the argument already stated, or
   detect/ask. Each initiative may use a different source than the others.
4. **Confirm** (the `init` "+ confirm"): show the id, the overlay path, the chosen source, and
   that host/verify/review/conventions are inherited from the base. Do not write until confirmed.
5. **Write `roadmaps/<ID>.md`** with only the overlay sections, following
   `examples/roadmaps/TICKET-101.md`:
   - `## Source binding` — paste the chosen source binding's verb table verbatim, substituting
     this epic's ids.
   - `## Queue` — a `<!-- TODO: dependency-ordered work for <ID> + fleet lane surfaces -->`
     placeholder (only the owner can supply it).
   - `## Reserved decisions` — `<!-- TODO: decisions reserved to a human for this epic -->`.
   - Optionally `## Canonical docs` / `## Spec gate` **only if** this epic overrides the base's;
     otherwise omit them and inherit the base.
6. **Markdown source only:** if the epic's source is the Markdown checklist, also write a starter
   `roadmaps/<ID>.ROADMAP.md` (the `- [ ] ID — title (deps: …)` shape from
   `examples/bindings/markdown.md`) and point the overlay's source binding at it. Never overwrite
   an existing one.
7. **Commit the scaffold** (§11), then **report** per §9.

## 11. Commit the scaffold

Scaffolding the roadmap is not done until it is committed. Uncommitted config is invisible to
freshly-branched worktrees (solo/fleet subagents branch off the committed base) and is discarded
when a worktree is removed — so an uncommitted roadmap is silently lost. After writing the files
in this run, stage and commit exactly them on the current branch:

- The overlay `roadmaps/<id>.md` (and `roadmaps/<id>.ROADMAP.md` if written), plus
  `roadmap.config.md` **only if** you created it this run.
- `git add <those paths>` then commit — stage the paths explicitly, never `git add -A`, so you
  commit only what init wrote. **Phrase the message in the project's resolved `## Conventions`
  commit style** (§4): with the engine default of Conventional Commits that's
  `chore(roadmap): scaffold <id>` (or `chore(roadmap): set up base config + scaffold <id>` when
  you created the base this run); a project with a different commit style gets the same intent
  written its way. The message names the initiative by id and says "scaffold" — never the word
  "overlay" (our internal term; see `autopilot:standards`). It is "scaffold `<id>`", not "scaffold
  roadmap overlay".
- Per `autopilot:standards` §5, `branch`/`commit` can lose an index-lock race — retry after a few
  seconds. If the working tree had unrelated staged changes, leave them; only add init's own paths.
- Pushing is the owner's call; committing is what keeps the roadmap from being lost. Note in the
  report (§9) that the scaffold was committed and is not yet pushed.
