---
name: init
description: Scaffold a project's roadmap.config.md by autodetecting its stack (code host, build gate, base branch) and confirming with the user before writing. Use once at setup, invoked by /autopilot:init. The one interactive autopilot command — solo and fleet never ask.
---

# Autopilot init

Generate a project's **`roadmap.config.md`** (and, when the source is a Markdown checklist, a
starter **`ROADMAP.md`**) by detecting what the repository already tells you and confirming the
rest with the user.

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
- If `roadmap.config.md` already exists at the repo root, **stop and report it — do not
  overwrite.** Only regenerate if the user explicitly asks; even then, show what you'd change first.

## 1. Detect the code-host

From `git remote get-url origin` (fall back to `git remote -v`):

- host contains `github.com` → **GitHub** (`gh`), use `examples/bindings/github.md`.
- host contains `gitlab` → **GitLab** (`glab`), use `examples/bindings/gitlab.md`.
- no remote / something else → ask the user which of the two to use.

## 2. Detect the roadmap source

Repository signals are weak here — be honest and lean on the confirm step.

- A `ROADMAP.md` exists → **Markdown checklist** (`examples/bindings/markdown.md`).
- Otherwise, if the remote is GitHub and `gh` is available → propose **GitHub Issues**
  (`examples/bindings/github-issues.md`).
- **Jira** has no repository signal — only reachable by asking.
- Default proposal when nothing is detected: **Markdown** (zero external tooling). Always let
  the user pick among {Jira, GitHub Issues, Markdown} at the confirm step.

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
- Conventions: propose the engine defaults — branch `feat/<ID>-<slug>` (`fix/`, `chore/`,
  `refactor/`), commit & PR title `feat(<ID>): …`, artifacts in the repo's language, no
  AI-tooling attribution. The user adjusts at confirm.

## 5. Detect canonical-doc candidates

Look for stakeholder-substitute docs and offer a shortlist (don't invent): `docs/PRODUCT.md`,
`ARCHITECTURE.md`, `CONTRIBUTING.md`, `README.md`, other `docs/*.md`.

## 6. Confirm (the "+ confirm")

Show a compact summary of every detection — source, code-host, verify, base branch, canonical
docs, and the sections you'll leave as TODO (Queue, Reserved decisions, Spec gate). Let the user
correct any field. Do not write anything until they confirm.

## 7. Write the config

Assemble `roadmap.config.md` at the repo root, following `examples/roadmap.config.example.md`'s
section order:

- **`## Source binding`** and **`## Code-host binding`** — paste the chosen bindings' verb tables
  verbatim from their files (copy, don't re-summarize — keep them in sync with the source) and
  substitute the detected ids/branch/host.
- **`## Verify`**, **`## Conventions`**, **`## Canonical docs`**, plus the code-host **conflict
  protocol** — fill from the detections above.
- Leave clearly-marked `<!-- TODO: … -->` placeholders for what only the owner can supply:
  **`## Queue`** (dependency-ordered work + fleet lane surfaces), **`## Reserved decisions`**,
  and **`## Spec gate`** (include the "delete this section if you have no spec requirement" note).

## 8. Starter ROADMAP.md (Markdown source only)

If the source is the Markdown checklist and no `ROADMAP.md` exists, also write a starter
`ROADMAP.md` using the format from `examples/bindings/markdown.md` (a couple of example items
showing the `- [ ] ID — title (deps: …)` shape and a `## Decisions` section). Never overwrite an
existing `ROADMAP.md`.

## 9. Report

State the files written and what is left to do: fill the TODO sections (start with `## Queue`),
optionally run the dry-run from the README to sanity-check selection, then `/autopilot:solo`
(single agent) or `/autopilot:fleet` (parallel). Point to `docs/config-schema.md` for the full
schema of every section.
