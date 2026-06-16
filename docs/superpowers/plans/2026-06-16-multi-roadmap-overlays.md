# Multi-roadmap (base ⊕ overlay) Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Let one repository host several independent epic-scoped roadmaps via a project base (`roadmap.config.md`) composed with optional per-epic overlays (`roadmaps/<ID>.md`), while keeping existing single-config projects working unchanged.

**Architecture:** Section-level override composition. The root `roadmap.config.md` holds project-wide sections; an overlay `roadmaps/<ID>.md` holds epic-specific sections that replace the base's same-named sections. `solo`/`fleet`/`task`/`init` resolve an effective config from base + the selected overlay. With no `roadmaps/` directory, the root file is the single roadmap (today's behavior).

**Tech Stack:** Pure Markdown. This plugin has **no build system, no package manager, no test runner** — every component is a `.md` file the agent host loads at runtime. "Verification" in each task therefore means `grep`/read assertions on the edited files and, at the end, the plugin-load smoke test (`claude --plugin-dir .` → `/help`). There are no unit tests to write; do not invent a framework. Commit after each task.

**Spec:** `docs/superpowers/specs/2026-06-16-multi-roadmap-overlays-design.md`

**Ordering rationale:** `docs/config-schema.md` is the schema of record, so it goes first and the rest conform to it. Runtime (`standards` → `task` → `solo` → `fleet`) next, then the setup layer (`init` skill + command), then the example overlay, then the top-level narrative docs (`README.md`, `AGENTS.md`), then the version bump, then a whole-repo consistency sweep.

---

### Task 1: Schema of record — base ⊕ overlay in `docs/config-schema.md`

**Files:**
- Modify: `docs/config-schema.md`

- [ ] **Step 1: Insert the "base ⊕ overlay" model section** after the intro blockquote (which ends `…by hand.`), immediately before `## The verb contract`.

Use Edit with `old_string` = the blockquote's last line plus the next heading:

```
> write it from scratch — by hand.

## The verb contract
```

Replace with:

```
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
  to `/autopilot:solo`, `/autopilot:fleet`, and `/autopilot:init`.

**Composition is section-level override.** The *effective config* for a roadmap is the base,
with each top-level `##` section present in the overlay **replacing** the base section of the
same name. Sections only in the base are inherited; sections only in the overlay are added.
This is a flat replace, not a deep merge — to know which value wins, ask only "is this section
in the overlay?".

**Backward compatible.** With no `roadmaps/` directory, the root `roadmap.config.md` is itself
the single roadmap and carries every section, exactly as before. Overlays are purely additive:
adopt them only when a repo needs more than one roadmap.

## The verb contract
```

- [ ] **Step 2: Annotate the required-sections intro** so the numbered list is read through the base/overlay lens.

Edit `old_string`:

```
Write it as Markdown with these sections. Headings are conventional; the engine reads the
content, so be explicit and literal — give exact commands, not descriptions of commands.
```

Replace with:

```
Write it as Markdown with these sections. Headings are conventional; the engine reads the
content, so be explicit and literal — give exact commands, not descriptions of commands. In a
single-roadmap repo all of them live in `roadmap.config.md`. In a multi-roadmap repo they split
between the base and each overlay as described in **One roadmap or several** above —
`## Source binding` and `## Queue` move into the overlay; the host/verify/review/conventions
sections stay in the base.
```

- [ ] **Step 3: Add the resolution rules** to "How the engine consumes it". Edit `old_string`:

```
- `autopilot:standards` is **not** configured here — it is the fixed discipline layer. The
  config can override a specific standard by saying so explicitly, but the defaults hold
  otherwise.

Start from `examples/roadmap.config.example.md` and replace each binding with your project's
tools.
```

Replace with:

```
- `autopilot:standards` is **not** configured here — it is the fixed discipline layer. The
  config can override a specific standard by saying so explicitly, but the defaults hold
  otherwise.

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
```

- [ ] **Step 4: Verify the edits landed and are self-consistent.**

Run:
```bash
grep -n "base ⊕ overlay\|One roadmap or several\|Selecting a roadmap\|roadmaps/<ID>.md" docs/config-schema.md
```
Expected: matches for the new heading, the resolution subheading, and at least two `roadmaps/<ID>.md` mentions.

- [ ] **Step 5: Commit.**

```bash
git add docs/config-schema.md
git commit -m "docs(schema): define base + overlay multi-roadmap config model"
```

---

### Task 2: Loosen the config reference in `skills/standards/SKILL.md`

Standards is project-agnostic and only names the config in passing. It must not gain resolution logic — just acknowledge that the bound config may be a base plus an overlay.

**Files:**
- Modify: `skills/standards/SKILL.md`

- [ ] **Step 1: Update the opening reference.** Edit `old_string`:

```
**project-agnostic**: they speak in the engine's abstract verbs, never in a specific
tool. Your project's `roadmap.config.md` binds the verbs to real commands. Wherever a
rule says `verify`, `push`, `merge`, `comment-pr`, etc., run the binding the config
defines for that verb.
```

Replace with:

```
**project-agnostic**: they speak in the engine's abstract verbs, never in a specific
tool. Your project's config binds the verbs to real commands — the base `roadmap.config.md`,
composed with the active `roadmaps/<ID>.md` overlay when the repo runs more than one roadmap.
Wherever a rule says `verify`, `push`, `merge`, `comment-pr`, etc., run the binding the
effective config defines for that verb.
```

- [ ] **Step 2: Update the override caveat.** Edit `old_string`:

```
Every rule below was earned the hard way. Treat them as non-negotiable unless a
`roadmap.config.md` explicitly overrides one.
```

Replace with:

```
Every rule below was earned the hard way. Treat them as non-negotiable unless the project's
config (base or overlay) explicitly overrides one.
```

- [ ] **Step 3: Verify no other bare single-file references remain in this file.**

Run:
```bash
grep -n "roadmap.config.md\|overlay\|effective config" skills/standards/SKILL.md
```
Expected: the two edited spots reference the overlay/effective config; no remaining sentence claims the config is a single file.

- [ ] **Step 4: Commit.**

```bash
git add skills/standards/SKILL.md
git commit -m "docs(standards): reference effective config (base + overlay)"
```

---

### Task 3: Make `skills/task/SKILL.md` read the effective config

**Files:**
- Modify: `skills/task/SKILL.md`

- [ ] **Step 1: Update the config-read instruction and the no-config stop.** Edit `old_string`:

```
holds the survival rules, the merge protocol, and the communication contract, and this skill
does not restate them. Read the project's **`roadmap.config.md`** for every binding,
command, and convention referenced below; resolve each verb (`claim`, `verify`, `push`,
`merge`, …) through that file.

If no `roadmap.config.md` exists, stop and say so — the engine has nothing to bind to.
```

Replace with:

```
holds the survival rules, the merge protocol, and the communication contract, and this skill
does not restate them. Read the project's **effective config** for every binding, command, and
convention referenced below — that is the base `roadmap.config.md` composed with the active
`roadmaps/<ID>.md` overlay (the caller, `/autopilot:solo` or `/autopilot:fleet`, resolves which
overlay; section-level override, overlay wins). Resolve each verb (`claim`, `verify`, `push`,
`merge`, …) through it; the `## Source binding` and `## Queue` you act on come from the overlay
when one is active, everything else from the base. The caller resolves the overlay **in the same
session** before invoking this skill, so the effective config is already in context — there is
no on-disk "active roadmap" pointer to read (the engine keeps no such state; selection is the
caller's `<ID>` argument plus the single-overlay default).

If no `roadmap.config.md` exists, stop and say so — the engine has nothing to bind to.
```

- [ ] **Step 2: Verify.**

Run:
```bash
grep -n "effective config\|roadmaps/<ID>.md\|overlay" skills/task/SKILL.md
```
Expected: the new instruction is present and names the overlay + section-level override.

- [ ] **Step 3: Commit.**

```bash
git add skills/task/SKILL.md
git commit -m "docs(task): resolve bindings from effective config (base + overlay)"
```

---

### Task 4: Add roadmap selection to `commands/solo.md`

**Files:**
- Modify: `commands/solo.md`

- [ ] **Step 1: Replace the intro + no-config line with the arg + resolution.** Edit `old_string`:

```
You are executing a project's roadmap autonomously. Read the project's `roadmap.config.md`
for all bindings, the queue, reserved decisions, and canonical docs. Load
**`autopilot:standards`** for the discipline you operate under, and drive each item through
the **`autopilot:task`** skill. Work the queue one item at a time until a stop condition.
Never ask the user anything; never wait for anyone.

If no `roadmap.config.md` exists, stop and say so.
```

Replace with:

```
You are executing a project's roadmap autonomously. You may be invoked as `/autopilot:solo` or
`/autopilot:solo <ID>`, where `<ID>` names an epic overlay (`roadmaps/<ID>.md`). Resolve the
**effective config** — the base `roadmap.config.md` composed with the selected overlay
(section-level override, overlay wins) — for all bindings, the queue, reserved decisions, and
canonical docs. Load **`autopilot:standards`** for the discipline you operate under, and drive
each item through the **`autopilot:task`** skill. Work the queue one item at a time until a stop
condition. Never ask the user anything; never wait for anyone.

**Resolve which roadmap before anything else:**

- `<ID>` given and `roadmaps/<ID>.md` exists → run the base ⊕ that overlay.
- `<ID>` given but no such overlay → stop and list the overlays that exist.
- No `<ID>`: exactly one file in `roadmaps/` → use it; more than one → stop and list them; none
  (or no `roadmaps/` directory) → use the root `roadmap.config.md` as the single roadmap.
- Once an overlay is chosen, it must contain `## Source binding` and `## Queue`. If either is
  missing, stop and report the missing section — do not silently fall back to the base.

If no `roadmap.config.md` exists at all, stop and say so. Stopping to list overlays is not
"asking the user" — it is the same fail-fast as a missing config; you still never prompt.
```

- [ ] **Step 2: Update the Start section to read the effective config.** Edit `old_string`:

```
Fetch the base. Read `roadmap.config.md`. Then run `autopilot:task` from the top of the queue.
```

Replace with:

```
Resolve the roadmap (above). Fetch the base. Read the effective config (base ⊕ overlay). Then
run `autopilot:task` from the top of that roadmap's queue.
```

- [ ] **Step 3: Verify.**

Run:
```bash
grep -n "autopilot:solo <ID>\|roadmaps/<ID>.md\|Resolve which roadmap\|effective config" commands/solo.md
```
Expected: arg form, overlay path, resolution heading, and effective-config Start line all present.

- [ ] **Step 4: Commit.**

```bash
git add commands/solo.md
git commit -m "feat(solo): accept roadmap <ID> and resolve base + overlay"
```

---

### Task 5: Add roadmap selection to `commands/fleet.md`

Fleet inherits all solo rules, so it only needs the same arg + resolution at its head; the merge-queue body is unchanged.

**Files:**
- Modify: `commands/fleet.md`

- [ ] **Step 1: Replace the intro + no-config line.** Edit `old_string`:

```
You are the FLEET ORCHESTRATOR for a project's roadmap. You never implement items yourself:
you claim, spawn, verify, merge, and account. Read the project's `roadmap.config.md` for the
bindings, the queue, and the lane surfaces. Load **`autopilot:standards`** — §5–§8 (parallel
checkout, the serial merge queue, crash recovery, cost policy) are the core of your job. All
`/autopilot:solo` rules apply to you (decision policy, canonical-docs guardrails, reserved
decisions, output contract), plus the following. Never ask the user anything.

If no `roadmap.config.md` exists, stop and say so.
```

Replace with:

```
You are the FLEET ORCHESTRATOR for a project's roadmap. You never implement items yourself:
you claim, spawn, verify, merge, and account. You may be invoked as `/autopilot:fleet` or
`/autopilot:fleet <ID>`, where `<ID>` names an epic overlay (`roadmaps/<ID>.md`). Read the
**effective config** — base `roadmap.config.md` composed with the selected overlay
(section-level override, overlay wins) — for the bindings, the queue, and the lane surfaces.
Load **`autopilot:standards`** — §5–§8 (parallel checkout, the serial merge queue, crash
recovery, cost policy) are the core of your job. All `/autopilot:solo` rules apply to you
(decision policy, canonical-docs guardrails, reserved decisions, output contract, **and the
roadmap-resolution rules**), plus the following. Never ask the user anything.

Resolve the roadmap exactly as `/autopilot:solo` does: `<ID>` → `roadmaps/<ID>.md`; missing
overlay → stop and list; no `<ID>` → single overlay used, multiple → stop and list, none → root
`roadmap.config.md`. If no `roadmap.config.md` exists at all, stop and say so. You run **one**
epic per invocation; running two epics at once is out of scope (they share the base branch and
the serial merge queue).
```

- [ ] **Step 2: Make the session marker repo-global.**

The Claim protocol's "Never run two fleets/solos at once" guard currently anchors the session
marker to the source binding — which now lives in the overlay, so two *different* epics would no
longer collide and could trample the shared base branch and serial merge queue. The guard must
become **repo-global**: one autopilot session per repository, regardless of epic. Edit
`old_string`:

```
- Never run two fleets/solos at once: check for an open session marker on the roadmap log
  (the config's source binding defines it); post one when starting, close it in the digest.
```

Replace with:

```
- Never run two fleets/solos at once **anywhere in this repository** — this guard is
  repo-global, not per-epic. With overlays the per-epic source logs differ, so do **not** anchor
  the marker to the overlay's source; anchor it to a repository-level marker the **base**
  defines (e.g. a marker on the base's code-host or a repo-level lock the base names), check it
  before starting, post one when starting, and close it in the digest. Two different epics
  running at once would share the base branch and this serial merge queue — the marker exists to
  prevent exactly that.
```

- [ ] **Step 3: Verify.**

Run:
```bash
grep -n "autopilot:fleet <ID>\|roadmaps/<ID>.md\|epic per invocation\|effective config\|repo-global" commands/fleet.md
```
Expected: arg form, overlay path, the one-epic-per-invocation note, effective-config mention, and the repo-global marker all present.

- [ ] **Step 4: Commit.**

```bash
git add commands/fleet.md
git commit -m "feat(fleet): accept roadmap <ID>, share solo resolution, repo-global session guard"
```

---

### Task 6: Overlay scaffold mode in `skills/init/SKILL.md`

This is the behavior change that fixes the reported bug: with a base already present, `/autopilot:init <ID>` scaffolds a new overlay instead of stopping.

**Files:**
- Modify: `skills/init/SKILL.md`

- [ ] **Step 1: Note the two modes in the intro.** Edit `old_string`:

```
Generate a project's **`roadmap.config.md`** (and, when the source is a Markdown checklist, a
starter **`ROADMAP.md`**) by detecting what the repository already tells you and confirming the
rest with the user.
```

Replace with:

```
Generate a project's **`roadmap.config.md`** (and, when the source is a Markdown checklist, a
starter **`ROADMAP.md`**) by detecting what the repository already tells you and confirming the
rest with the user.

This skill has **two modes**, chosen by what already exists and whether you were given an id:

- **Base mode** — no `roadmap.config.md` yet: scaffold the project base (§§1–9 below).
- **Overlay mode** — a base already exists **and** you were invoked with an id
  (`/autopilot:init BRU-101`): scaffold a new epic overlay `roadmaps/<ID>.md` instead of
  stopping (§10). A base existing with **no** id still stops and reports (§0), so a bare
  re-run never regenerates the base by surprise.
```

- [ ] **Step 2: Replace the §0 precondition so an existing base branches by mode.** Edit `old_string`:

```
- If `roadmap.config.md` already exists at the repo root, **stop and report it — do not
  overwrite.** Only regenerate if the user explicitly asks; even then, show what you'd change first.
```

Replace with:

```
- Existing `roadmap.config.md` at the repo root decides the mode:
  - invoked **with an id** (`/autopilot:init BRU-101`) → **overlay mode**, jump to §10 (do not
    touch the base).
  - invoked **with no id** → **stop and report** — do not overwrite. Only regenerate the base
    if the user explicitly asks; even then, show what you'd change first.
- No `roadmap.config.md` yet → **base mode**, continue with §1 — regardless of whether an id was
  given. There is no base yet to layer an overlay onto, so an id here is at most a naming hint
  for the base; never branch into overlay mode without an existing base. (Overlays come later,
  via a second `/autopilot:init <ID>` run once the base exists.)
```

- [ ] **Step 3: Append §10 (overlay mode) at the end of the file**, after the current §9 "Report" section (which ends `…the full schema of every section.`). Edit `old_string`:

```
optionally run the dry-run from the README to sanity-check selection, then `/autopilot:solo`
(single agent) or `/autopilot:fleet` (parallel). Point to `docs/config-schema.md` for the full
schema of every section.
```

Replace with:

```
optionally run the dry-run from the README to sanity-check selection, then `/autopilot:solo`
(single agent) or `/autopilot:fleet` (parallel). Point to `docs/config-schema.md` for the full
schema of every section.

## 10. Overlay mode (existing base + id)

You were invoked as `/autopilot:init <ID>` with a `roadmap.config.md` already present. Scaffold
an **epic overlay**, reusing the base for everything project-wide. Do not edit the base.

1. **Resolve `<ID>`** from the argument (e.g. `BRU-101`); the overlay path is `roadmaps/<ID>.md`.
   If it already exists, **stop and report** — never overwrite an overlay (same rule as the base).
2. **Read the base** `roadmap.config.md` to confirm the shared sections (`## Code-host binding`,
   `## Verify`, `## Review ritual`, `## Conventions`) the overlay will inherit. The overlay does
   **not** repeat them.
3. **Pick the source binding for this epic.** Re-run §2's detection (a `ROADMAP.md` → Markdown;
   GitHub remote + `gh` → GitHub Issues; else ask among {Jira, GitHub Issues, Markdown}). The
   epic may use a different source than other overlays.
4. **Confirm** (the `init` "+ confirm"): show the id, the overlay path, the chosen source, and
   that host/verify/review/conventions are inherited from the base. Do not write until confirmed.
5. **Write `roadmaps/<ID>.md`** with only the overlay sections, following
   `examples/roadmaps/BRU-101.md`:
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
7. **Report:** the overlay written, that the base is untouched, the TODO sections to fill
   (start with `## Queue`), and that the epic now runs via `/autopilot:solo <ID>` or
   `/autopilot:fleet <ID>`.
```

- [ ] **Step 4: Verify.**

Run:
```bash
grep -n "two modes\|Overlay mode\|roadmaps/<ID>.md\|## 10" skills/init/SKILL.md
```
Expected: the modes note, the §10 heading, the overlay path, and the `## 10` section marker all present.

- [ ] **Step 5: Commit.**

```bash
git add skills/init/SKILL.md
git commit -m "feat(init): overlay mode scaffolds roadmaps/<ID>.md when base exists"
```

---

### Task 7: Document the `<ID>` argument in `commands/init.md`

**Files:**
- Modify: `commands/init.md`

- [ ] **Step 1: Replace the "already exists → stops" paragraph.** Edit `old_string`:

```
If `roadmap.config.md` already exists, it reports that and stops rather than overwriting your
config.
```

Replace with:

```
Two ways to run it:

- **`/autopilot:init`** — first-time setup: autodetect and scaffold the project base
  `roadmap.config.md`. If a base already exists, it reports that and stops rather than
  overwriting your config.
- **`/autopilot:init <ID>`** (e.g. `/autopilot:init BRU-101`) — with a base already present,
  scaffold a new **epic overlay** `roadmaps/<ID>.md` (its own source binding and queue, reusing
  the base's code host, verify gate, and conventions). The epic then runs via
  `/autopilot:solo <ID>` or `/autopilot:fleet <ID>`. See `docs/config-schema.md` →
  *One roadmap or several*. It won't overwrite an existing overlay.
```

- [ ] **Step 2: Verify.**

Run:
```bash
grep -n "autopilot:init <ID>\|epic overlay\|roadmaps/<ID>.md" commands/init.md
```
Expected: the arg form, "epic overlay", and overlay path all present.

- [ ] **Step 3: Commit.**

```bash
git add commands/init.md
git commit -m "docs(init): document /autopilot:init <ID> overlay scaffolding"
```

---

### Task 8: Example overlay under `examples/`

**Files:**
- Create: `examples/roadmaps/BRU-101.md`
- Modify: `examples/roadmap.config.example.md`

- [ ] **Step 1: Create the example overlay.** Write `examples/roadmaps/BRU-101.md`:

```markdown
# roadmaps/BRU-101.md (example overlay)

> An **epic overlay**. Drop overlays at `roadmaps/<ID>.md` in your repo root when one repo runs
> several roadmaps. It carries only the epic-specific sections; everything project-wide
> (`## Code-host binding`, `## Verify`, `## Review ritual`, `## Conventions`,
> `## Environment gotchas`) is inherited from the base `roadmap.config.md`. Run this epic with
> `/autopilot:solo BRU-101` or `/autopilot:fleet BRU-101`. See `docs/config-schema.md` →
> *One roadmap or several*.

## Source binding

Jira via `acli` (see `examples/bindings/jira.md`). Epic `BRU-101` is this roadmap; its
comments are the decision log.

| Verb           | Command                                                                                   |
| -------------- | ----------------------------------------------------------------------------------------- |
| `next-ready`   | First item in `## Queue` order whose deps are `Done`; cross-check with `acli jira workitem list --jql "parent = BRU-101 AND status = 'To Do'"`. |
| `claim`        | `acli jira workitem transition --key <KEY> --status "In Progress"`                         |
| `complete`     | `acli jira workitem transition --key <KEY> --status "Done"`                                |
| `park`         | transition to `"To Do"` + `note` the reason                                               |
| `note`         | `acli jira workitem comment create --key <KEY> --body-file <adf.json>`                     |
| `log-decision` | `note` on `BRU-101`                                                                        |
| `derive`       | `acli jira workitem create --parent BRU-101 --type Sub-task --summary "..." --description-file <adf.json>` |

## Queue

Dependency order; `→` is "then", `∥` is "parallelizable". Lanes for the fleet are by surface.

```
BRU-102 (schema) → BRU-103 (api) ∥ BRU-104 (web)
→ BRU-105 (rollout)
```

**Fleet lanes (disjoint surfaces, max 3 concurrent):**

- `backend` — anything touching `backend/` or shared code. Max one in flight.
- `frontend` — webapp-only.

## Reserved decisions

Reserved to a human for this epic (build around them; tag the item `PENDING-OWNER` and continue):

- Anything requiring the owner's accounts, credentials, or signatures.

<!-- Inherits ## Canonical docs and (if any) ## Spec gate from the base roadmap.config.md.
     Add either section here only to override the base for this epic. -->
```

- [ ] **Step 2: Cross-link from the example base.** In `examples/roadmap.config.example.md`, edit the top blockquote `old_string`:

```
> Drop this file at your repository root as `roadmap.config.md` and replace the placeholders.
> This example wires Jira as the source and GitHub as the code-host. Swap the bindings for
> the ones under `examples/bindings/` that match your stack. See `docs/config-schema.md` for
> the full schema and the verb contract.
```

Replace with:

```
> Drop this file at your repository root as `roadmap.config.md` and replace the placeholders.
> This example wires Jira as the source and GitHub as the code-host. Swap the bindings for
> the ones under `examples/bindings/` that match your stack. See `docs/config-schema.md` for
> the full schema and the verb contract.
>
> Running **several roadmaps** in one repo? This file becomes the shared **base** (host, verify,
> review, conventions) and each epic gets an overlay — see `examples/roadmaps/BRU-101.md` and
> `docs/config-schema.md` → *One roadmap or several*.
```

- [ ] **Step 3: Verify.**

Run:
```bash
test -f examples/roadmaps/BRU-101.md && grep -n "epic overlay\|inherited from the base" examples/roadmaps/BRU-101.md && grep -n "several roadmaps\|examples/roadmaps/BRU-101.md" examples/roadmap.config.example.md
```
Expected: overlay file exists with the "epic overlay" header and inheritance note; base example links to it.

- [ ] **Step 4: Commit.**

```bash
git add examples/roadmaps/BRU-101.md examples/roadmap.config.example.md
git commit -m "docs(examples): add example epic overlay, link from base example"
```

---

### Task 9: Update top-level narrative — `README.md` and `AGENTS.md`

**Files:**
- Modify: `README.md`
- Modify: `AGENTS.md`

- [ ] **Step 1: README — add a "Multiple roadmaps" section** after the Quickstart's step 3 block. Edit `old_string`:

```
   - **Unattended to completion:** launch either on a loop. It ends itself at mission
     complete and does not reschedule.

## Dry run (optional but recommended)
```

Replace with:

```
   - **Unattended to completion:** launch either on a loop. It ends itself at mission
     complete and does not reschedule.

## Multiple roadmaps in one repo

One repo can run several independent roadmaps (one per initiative/epic). The config splits into
a shared **base** and per-epic **overlays**:

- `roadmap.config.md` (root) — the project base: code host, verify gate, review ritual,
  conventions.
- `roadmaps/<ID>.md` — an epic overlay: that epic's `## Source binding` and `## Queue` (plus
  optional overrides). The effective config is base + overlay, section-level override.

Scaffold a new epic with `/autopilot:init BRU-101`, then run it with `/autopilot:solo BRU-101`
or `/autopilot:fleet BRU-101`. With no `roadmaps/` directory the root file is simply the single
roadmap, exactly as before — overlays are additive. Full rules:
`docs/config-schema.md` → *One roadmap or several*. One epic runs per `solo`/`fleet` invocation.

## Dry run (optional but recommended)
```

- [ ] **Step 2: README — note the overlay non-overwrite in the Quickstart init step.** Edit `old_string`:

```
   checklist source). `init` is the **only** autopilot command that asks you questions;
   `solo` and `fleet` never do. It won't overwrite an existing `roadmap.config.md`.
```

Replace with:

```
   checklist source). `init` is the **only** autopilot command that asks you questions;
   `solo` and `fleet` never do. It won't overwrite an existing `roadmap.config.md` — and
   `/autopilot:init BRU-101` instead scaffolds an epic overlay (see *Multiple roadmaps* below).
```

- [ ] **Step 3: AGENTS.md — update the layer diagram's middle row.** Edit `old_string`:

```
Project config            roadmap.config.md  +  bindings  (examples/ has copyable templates)
   │  resolves verbs → real tools
```

Replace with:

```
Project config            roadmap.config.md (base)  +  roadmaps/<ID>.md (per-epic overlays)
   │  resolves verbs → real tools; effective config = base ⊕ overlay (overlay wins)
```

- [ ] **Step 4: AGENTS.md — extend the "Writing config" section.** Edit `old_string`:

```
Consumers drop a `roadmap.config.md` at their repo root. Start from
`examples/roadmap.config.example.md`. Required sections: `## Source binding`,
`## Code-host binding`, `## Verify`, `## Review ritual`, `## Conventions`,
`## Reserved decisions`, `## Queue`, `## Canonical docs`. Optional: `## Spec gate`,
`## Environment gotchas`.
```

Replace with:

```
Consumers drop a `roadmap.config.md` at their repo root. Start from
`examples/roadmap.config.example.md`. Required sections: `## Source binding`,
`## Code-host binding`, `## Verify`, `## Review ritual`, `## Conventions`,
`## Reserved decisions`, `## Queue`, `## Canonical docs`. Optional: `## Spec gate`,
`## Environment gotchas`.

**Multiple roadmaps:** one repo may run several epic-scoped roadmaps. The root
`roadmap.config.md` holds the project-wide sections (host, verify, review, conventions); each
epic adds an overlay `roadmaps/<ID>.md` with its own `## Source binding` and `## Queue`. The
effective config is base ⊕ overlay (section-level override, overlay wins); `/autopilot:solo
<ID>` / `/autopilot:fleet <ID>` select the epic. With no `roadmaps/`, the root file is the
single roadmap. Start from `examples/roadmaps/BRU-101.md`; full rules in
`docs/config-schema.md` → *One roadmap or several*.
```

- [ ] **Step 5: AGENTS.md — add an invariant** so future edits keep the split coherent. Edit `old_string`:

```
- `skills/task` drives one item; the fleet orchestrator drives the merge queue. This separation
  is load-bearing: fleet subagents do **not** merge — they stop after PR open and triage; the
  orchestrator's serial merge queue handles everything else.
```

Replace with:

```
- `skills/task` drives one item; the fleet orchestrator drives the merge queue. This separation
  is load-bearing: fleet subagents do **not** merge — they stop after PR open and triage; the
  orchestrator's serial merge queue handles everything else.
- The base ⊕ overlay split is section-level override, never a deep merge: an overlay section
  replaces the base section of the same name. `## Source binding` and `## Queue` belong to the
  overlay; host/verify/review/conventions belong to the base. Keep `docs/config-schema.md` →
  *One roadmap or several* as the authority and don't restate the resolution rules elsewhere.
```

- [ ] **Step 6: Verify both files.**

Run:
```bash
grep -n "Multiple roadmaps\|roadmaps/<ID>.md\|/autopilot:init BRU-101" README.md
grep -n "roadmaps/<ID>.md\|base ⊕ overlay\|Multiple roadmaps" AGENTS.md
```
Expected: README has the new section + the init-overlay note; AGENTS has the diagram update, the writing-config block, and the new invariant.

- [ ] **Step 7: Commit.**

```bash
git add README.md AGENTS.md
git commit -m "docs: document multiple roadmaps in README and AGENTS"
```

---

### Task 10: Version bump

Pushing commits does not update installed users — the plugin version must change for the marketplace to ship it. This is a feature, so a minor bump.

**Files:**
- Modify: `.claude-plugin/plugin.json`

- [ ] **Step 1: Bump the version.** Edit `old_string`:

```
  "version": "0.2.0",
```

Replace with:

```
  "version": "0.3.0",
```

- [ ] **Step 2: Verify it is still valid JSON and the version changed.**

Run:
```bash
python3 -c "import json;print(json.load(open('.claude-plugin/plugin.json'))['version'])"
```
Expected: prints `0.3.0`.

- [ ] **Step 3: Commit.**

```bash
git add .claude-plugin/plugin.json
git commit -m "chore: bump version to 0.3.0 for multi-roadmap support"
```

---

### Task 11: Whole-repo consistency sweep + load smoke test

A final pass to catch any place that still assumes a single config file, and to confirm the plugin still loads.

**Files:**
- (read-only sweep; fix-forward only if something is found)

- [ ] **Step 1: Sweep for stale single-config assumptions.**

Run:
```bash
grep -rn "only.*roadmap.config.md\|single .*roadmap.config\|exists, stop" \
  README.md AGENTS.md commands/ skills/ docs/config-schema.md examples/
```
Expected: every hit is either an intentional back-compat statement ("with no `roadmaps/`, the root file is the single roadmap") or the no-`roadmap.config.md`-at-all stop. If any hit still asserts there can only ever be one config, fix it to match the base/overlay model and amend the relevant task's commit.

- [ ] **Step 2: Confirm cross-references resolve.** Every doc that mentions the overlay points at a real anchor.

Run:
```bash
grep -rn "One roadmap or several" docs/config-schema.md && \
grep -rln "One roadmap or several" README.md AGENTS.md commands/init.md examples/
```
Expected: the anchor exists in `config-schema.md` and each referrer names it identically.

- [ ] **Step 3: Plugin-load smoke test.** From the repo root:

```bash
claude --plugin-dir . -p "/help" 2>&1 | grep -i "autopilot"
```
Expected: `/autopilot:init`, `/autopilot:solo`, `/autopilot:fleet` appear. (If `claude` non-interactive `-p` is unavailable in the environment, fall back to a manual `claude --plugin-dir .` session and confirm `/help` lists the three commands — the AGENTS.md "Running locally" check.)

- [ ] **Step 4: Final commit (only if the sweep changed anything).**

```bash
git add -A
git commit -m "docs: consistency sweep for multi-roadmap model"
```

---

## Self-review (completed during planning)

- **Spec coverage:** base ⊕ overlay model → Task 1; section-level override → Tasks 1, 3, AGENTS invariant (9); resolution/selection → Tasks 1, 4, 5; "required overlay section absent = config error at resolution time" → Tasks 1, 4 (and the schema rule in 1); backward compat → Tasks 1, 4; init overlay mode → Tasks 6, 7; no-persisted-pointer non-goal (handoff is in-session) → Task 3; non-goal (no parallel epics) enforced by the repo-global session marker → Task 5; files-to-change table → Tasks 1–10; version bump → Task 10; invariants preserved → Tasks 2, 9. All spec sections map to a task.
- **Placeholder scan:** the only `TODO`/`TBD` strings are the intentional scaffold placeholders written *into* generated overlays (Task 6/8), not gaps in the plan.
- **Term consistency:** "effective config", "base ⊕ overlay", "section-level override", and `roadmaps/<ID>.md` are used identically across every task and match the spec.
