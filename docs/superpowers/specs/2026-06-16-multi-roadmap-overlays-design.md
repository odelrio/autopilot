# Multi-roadmap support via base ⊕ overlay

**Status:** Approved design — ready for implementation plan
**Date:** 2026-06-16

## Problem

Autopilot assumes a single `roadmap.config.md` at the repo root. The whole runtime
(`solo`, `fleet`, `task`, `standards`), plus `init` and the docs, are written around that one
file. A consumer hit two symptoms of this assumption:

1. `/autopilot:init Vamos a trabajar en TICKET-101` refused to do anything because
   `roadmap.config.md` already existed (the init contract stops rather than overwrite), and had
   no way to act on the "work on TICKET-101" intent.
2. The underlying need: **a single repository should host several independent roadmaps** — one
   per initiative/epic (e.g. `TICKET-101`), each with its own queue and source binding, runnable
   one at a time.

## Goals

- Let one repo carry multiple epic-scoped roadmaps without duplicating project-wide config.
- Target a specific roadmap when launching `solo`/`fleet`.
- Keep `solo`/`fleet` non-interactive.
- Full backward compatibility: an existing single-config project keeps working untouched.
- Make `init` able to scaffold an additional roadmap instead of refusing.

## Non-goals (YAGNI for now)

> **Superseded (2026-06-16, v0.5.0).** The first bullet below no longer holds. Two epics may now
> run concurrently: the session guard became **per-roadmap** (not repo-global), and each
> concurrently-run roadmap targets its own **integration branch** so the serial merge queues
> never share a branch. See `docs/config-schema.md` → *Concurrent roadmaps and integration
> branches* and `autopilot:standards` §10–§11.

- Running **two epics in parallel simultaneously** with `fleet`. Each `fleet <ID>` still
  parallelizes lanes *within* one epic. Launching two epics at once shares the base branch and
  the serial merge queue, so it is documented as a known limitation, not solved here.
- Deep/recursive merging of config sections. Override is section-level only (see below).
- A "switch active roadmap" command or persisted active-roadmap pointer. Selection is by
  explicit argument plus the single-overlay default.

## Design

### Config model: base ⊕ overlay

Two kinds of file, composed at runtime:

- **`roadmap.config.md` (repo root) — the project base.** Holds what is shared across every
  epic in the repo:
  - `## Code-host binding`
  - `## Verify`
  - `## Review ritual`
  - `## Conventions`
  - `## Environment gotchas`
  - `## Canonical docs` (default; an overlay may override)

- **`roadmaps/<ID>.md` — an epic overlay.** Holds what is specific to one initiative:
  - `## Source binding` (required)
  - `## Queue` (required)
  - `## Reserved decisions`
  - optionally overrides `## Canonical docs`, `## Spec gate`, `## Conventions`

`<ID>` (e.g. `TICKET-101`) is simultaneously the overlay filename stem and the argument passed to
`solo`/`fleet`/`init`. Filenames use `.md` (not `.config.md`) to distinguish overlays from the
base.

### Composition rule: section-level override

The **effective config** for a roadmap is the base, with each top-level `##` section that is
present in the overlay *replacing* the base's section of the same name. This is a flat,
section-level override — not a deep/key-level merge. The rule to determine which value wins is
simply: "is this section present in the overlay?"

Sections present only in the base are inherited. Sections present only in the overlay are added.
A required overlay section absent from the overlay is a config error reported at resolution time.

### Resolution and selection

Given `/autopilot:solo [ID]` (and identically `/autopilot:fleet [ID]`):

1. **ID given, `roadmaps/<ID>.md` exists** → effective = base ⊕ overlay; run that epic.
2. **ID given, no matching overlay** → stop and list the available overlays.
3. **No ID:**
   - `roadmaps/` contains **exactly one** overlay → use it without asking.
   - `roadmaps/` contains **more than one** overlay → stop and list them (same pattern as the
     existing "no `roadmap.config.md` exists" stop).
   - `roadmaps/` is empty or absent → use the root `roadmap.config.md` as today.

Consequence — backward compatibility: **when no overlays exist, the root file IS the single
roadmap** (today's exact behavior, including its own `## Source binding` and `## Queue`). The
moment `roadmaps/` gains overlays, overlays become the unit of execution and the root file is
treated as the project base.

This stop-and-list behavior preserves the non-interactive contract: `solo`/`fleet` still never
prompt; an ambiguous launch fails fast with the list of valid IDs, exactly like a missing
config does today.

### `init` in "add an epic" mode

- `/autopilot:init` with no ID and no base → current behavior (scaffold the base
  `roadmap.config.md`, autodetect + confirm).
- `/autopilot:init <ID>` with a base already present → **does not stop.** Enters overlay mode:
  scaffolds `roadmaps/<ID>.md` with `## Source binding`, `## Queue`, and `## Reserved decisions`
  (TODO placeholders for owner-supplied content), reusing the code-host / verify / conventions
  already in the base. Still confirms before writing; still never overwrites an existing
  overlay.
- `/autopilot:init` with no ID and a base present → keep the current stop-and-report (no
  unintended base regeneration).

This directly resolves the reported bug: "Vamos a trabajar en TICKET-101" becomes "I'll scaffold
the TICKET-101 overlay."

## Files to change

All Markdown — this plugin has no build system.

| File | Change |
|------|--------|
| `skills/init/SKILL.md` | Add overlay scaffold mode (`<ID>` arg); update the existing-config precondition so it branches to overlay mode instead of always stopping. |
| `commands/init.md` | Document the `<ID>` argument and the overlay-vs-base behavior. |
| `commands/solo.md` | Accept optional `[ID]`; implement the resolution/selection rules. |
| `commands/fleet.md` | Same resolution/selection rules as solo. |
| `skills/task/SKILL.md` | "Read `roadmap.config.md`" → "read the effective config (base ⊕ overlay)." |
| `skills/standards/SKILL.md` | Update config references to the base ⊕ overlay model. |
| `docs/config-schema.md` | Document base vs. overlay, which sections live where, and the composition + resolution rules. |
| `README.md` | Update the architecture/layers description and add a multi-roadmap walkthrough. |
| `AGENTS.md` | Update the layer diagram and invariants to mention base ⊕ overlay. |
| `examples/` | Add an example overlay (`examples/roadmaps/<ID>.md` or equivalent) alongside the existing example base. |
| `.claude-plugin/plugin.json` | Version bump on release (pushing commits does not update installed users — the version must change). |

## Invariants to preserve

- `autopilot:standards` remains the single source of discipline rules; reference it, don't
  restate.
- The runtime engine (`solo`/`fleet`/`task`/`standards`) stays tool-agnostic; real tool names
  remain confined to the setup layer (`init`), `examples/`, and consumer configs.
- `docs/config-schema.md` stays the schema of record and is updated in the same change as any
  section/verb change.

## Open questions

None blocking. Cross-epic parallel `fleet` is explicitly deferred (see Non-goals).
