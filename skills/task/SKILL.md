---
name: task
description: Execute one roadmap item end-to-end — claim, optional spec gate, implement, verify, review ritual, open PR, merge, bookkeeping. Use when told to work the roadmap queue or a specific item id, driven by the project's roadmap.config.md. Invoked per-item by /autopilot:solo and /autopilot:fleet.
---

# Autopilot task loop

One invocation = one roadmap item, end to end. Load **`autopilot:standards`** first — it
holds the survival rules, the merge protocol, and the communication contract, and this skill
does not restate them. Read the project's **`roadmap.config.md`** for every binding,
command, and convention referenced below; resolve each verb (`claim`, `verify`, `push`,
`merge`, …) through that file.

If no `roadmap.config.md` exists, stop and say so — the engine has nothing to bind to.

**Fleet mode:** when `/autopilot:fleet` invokes you, its overrides apply — do **not** merge,
do **not** transition the item, skip any manual QA the config defers to the orchestrator, and
your final message is the fleet report. The overrides you receive in the spawn prompt win
over the merge/bookkeeping steps below.

## 0. Pick & open

- `next-ready` (or take the explicit item id given). The order source is the config's
  `## Queue`, cross-checked against live status; skip anything already done or in progress.
- `claim` the item.
- `branch` from the freshly fetched base, named per the config's `## Conventions`.

## 1. Spec gate *(only if the config defines `## Spec gate`)*

- For the surfaces the gate names, write/extend the spec with test links **before** code;
  self-review for placeholders, contradictions, and untestable clauses.
- If part of the item is a **reserved decision** (per `## Reserved decisions`): do not
  implement that part — tag it pending, `note` it for the next checkpoint, and continue with
  the rest.

## 2. Implement

- The item's description (endpoints, schema, signatures) is the contract; document justified
  deviations in the PR and via `note`.
- **Documentation impact:** up front, list which docs this item creates/changes/deletes, and
  apply the edits on the right surface as the project routes them.
- Honor every project rule the config names: data/i18n completeness, generated-artifact
  regeneration, migration verification on a fresh **and** an upgraded state.

## 3. Verify (before requesting any review)

- Run the config's `## Verify` gate to green — in the **foreground**, with output captured to
  a file (see `autopilot:standards` §1). This is the merge gate.
- Run any project-specific checks the config lists (component stories, manual QA at the
  required viewport, accessibility pass, a self-check on auth/PII for new endpoints).

## 4. Reviews (per the config's `## Review ritual`)

- Run each reviewer the config names, on the diff (and spec where relevant).
- **Post each review via `comment-pr` the moment its triage is done, before any prose**
  (`autopilot:standards` §4 — this is the most common way these agents die). Triage with
  receiving-review discipline: verify before adopting; record rejected findings and why in
  the PR.
- Apply the config's cost policy to any helper subagents (mechanical → cheap model; judgment
  stays on the default). Triage is never delegated to the cheap model.

## 5. PR & merge

- `open-pr` using the project's template, in the project's language, with the item linked in
  the footer. Honor the no-attribution rule.
- `comment-pr` (or `note` on the item) with the PR reference.
- Before merging, rebase onto the fresh base and apply the config's **conflict protocol**
  (`autopilot:standards` §6): regenerate generated artifacts, fix ordering collisions, union
  additive data, re-run the **full** `verify` after any source conflict. If resolving would
  override a logged decision or undo another merged PR, stop and `park`.
- `merge` only when: `verify` green on the rebased branch, all required reviews triaged with
  no open high-severity finding, and the step-3 checklist done. *(Fleet mode: stop here — the
  orchestrator's merge queue owns the rest.)*

## 6. Bookkeeping (after merge)

- `complete` the item.
- **Docs check:** do the implementation decisions contradict the canonical docs? Clarifying
  edits go on the right surface in the same change or a tracked follow-up; decision-level
  conflicts are queued for a checkpoint, not silently rewritten.
- `log-decision` for anything significant.
- `derive` follow-up work discovered during the task.
- Record durable learnings where the project keeps them; append to the session digest.

## Stop conditions

Follow `autopilot:standards` §9. In short: two consecutive failed `verify` on this item →
`note` the findings, `park` it, stop the loop; a reserved decision that blocks the whole item
→ `park` and pick the next; context degraded → finish cleanly, write the digest, stop.

## Per-session digest

One `log-decision` on the roadmap log at session end: items completed, decisions taken, docs
touched, derived work, and the queue left for the owner (specs to skim, reserved decisions
pending, credentials due as their wave approaches).
