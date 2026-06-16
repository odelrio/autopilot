---
description: Autonomous single-agent roadmap execution — works items end-to-end via autopilot:task, one at a time, without blocking, until a stop condition.
---

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

## Mission

Ship roadmap items end-to-end (optional spec → code → verify → reviews → merge → bookkeeping)
at high quality. Items blocked by an external wall (missing credentials/infra, per the
config's `## Queue`) stay parked until that wall lifts.

## Decision policy

- Take every decision yourself. Your guardrails replacing the stakeholder are the documents
  named in the config's `## Canonical docs`. Where they conflict or are silent, pick the most
  operational, simplest interpretation.
- `log-decision` for every significant call (decision + why). Decisions are yours;
  traceability is mandatory.
- Honor the config's `## Reserved decisions`: build everything around the reserved core,
  leave a clearly-tagged pending note on the item, and continue.

## Never block

- An item needs human input or credentials → ship the implementable subset; if none, `note`
  why, `park` it, take the next item.
- An external provider fails twice in a row → circuit-break it for the session, continue.
- Ambiguity → decide, `log-decision`, move.

## Quality bars (non-negotiable)

The `autopilot:task` ritual is the law — follow it literally. No `merge` with: a red
`verify`, missing tests on business logic, an open high-severity security finding, missing
required data (e.g. translation keys), or an unverified migration. Never weaken, skip, or
delete a failing test to get green — fix the cause or `park`. Respect every convention the
config names.

## Output contract (strict)

Per `autopilot:standards` §4. One line per item to the chat —
`ITEM-ID → PR #N merged` or `ITEM-ID → parked: <reason, ≤5 words>`. All prose lives in
artifacts. At session end, one digest entry on the roadmap log plus its one-line pointer in
chat. The engine's mechanics stay internal (§4): never narrate overlay/roadmap resolution,
the drift reconciliation, or how you sequenced the queue — only the contract lines, the
one-line drift report, and the digest pointer reach chat.

## Run to completion, then stop (not a daemon)

Mission = every item you *can* do is done; what remains is blocked only by an external wall or
a reserved decision. Total work is bounded by the roadmap, not a timer. No fixed per-session
item cap — a session does as many items as its context allows, then resumes.

## Stop conditions

Per `autopilot:standards` §9: **mission complete** (write the final digest, end; if on a
loop, do not reschedule) · **session boundary** (finish the current item, write a resume
digest, stop; resume next firing) · **safety** (two consecutive failed gates on one item →
park, next; two consecutive parked items → digest and stop).

To run the doable roadmap to completion unattended, launch this command on a loop; it ends
itself at mission complete.

## Drift check (queue vs source)

Once, at the start of a run, reconcile the `## Queue` against the live source — best-effort and
non-blocking, per `autopilot:standards` §3:

- `list-open` the source to get the set of open item ids.
- Compare it with the ids in the `## Queue`, and report the difference in one line, to chat and
  the digest:
  - open in the source but **not in the queue** → `drift: <ids> not in queue`. New work added
    after the queue was written. Do **not** work it — you have no dependency order or lane for
    it; leave it for the owner to place in `## Queue`.
  - in the queue but **no longer open** in the source → `drift: <ids> queue-only`. Already
    closed or removed; skip it.
- **Detect and report only.** Never invent an order or lane, never work an un-queued item,
  never block. If `list-open` fails twice, circuit-break it for the session and continue from
  the queue.

## Start

Resolve the roadmap (above). Fetch the base. Read the effective config (base ⊕ overlay). Run the
drift check, then run `autopilot:task` from the top of that roadmap's queue.
