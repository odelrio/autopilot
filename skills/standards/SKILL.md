---
name: standards
description: The discipline layer for autonomous roadmap execution — survival rules, verification gates, the strictly-serial merge protocol, crash recovery, and the communication contract. Load this whenever running autopilot:task, /autopilot:solo, or /autopilot:fleet; every other autopilot component references it instead of restating it.
---

# Autopilot standards

These are the rules that keep an autonomous agent alive and trustworthy. They are
**project-agnostic**: they speak in the engine's abstract verbs, never in a specific
tool. Your project's `roadmap.config.md` binds the verbs to real commands. Wherever a
rule says `verify`, `push`, `merge`, `comment-pr`, etc., run the binding the config
defines for that verb.

Every rule below was earned the hard way. Treat them as non-negotiable unless a
`roadmap.config.md` explicitly overrides one.

## 1. Verify before you claim

The cardinal sin is reporting success you did not observe.

- **Run gates in the foreground.** Never end your turn while a build, test, or `verify`
  command runs in the background waiting for a notification. Agents die there. A gate that
  takes twenty minutes in the foreground is correct; a backgrounded one that "completes"
  while you were composing a message means you died, not finished.
- **A "completed" notification with a mid-flight final message means death, not done.**
  If an agent's last words are "waiting for CI" or a half-written report, it crashed. Treat
  its branch as work-in-progress to recover, never as finished.
- **Capture long-command output to a file**, then read the file:
  `cmd > run.log 2>&1; echo "exit=$?"`. Piping through `tail`/`head` discards the real exit
  code, and temp filesystems hit transient out-of-space errors that a pipe will hide.
- **`verify` must run against committed state.** Many verify commands diff against the
  remote base and see only committed changes — commit first, or you will "pass" having
  tested nothing.
- **A post-rebase build is a new build.** After resolving any source conflict, re-run the
  full `verify` gate. It is not a formality.
- **Confirm every `push` was accepted.** After pushing, check the remote tip equals your
  local tip. A rejected (non-fast-forward) push followed by a squash-merge silently drops
  your commits. Never treat a branch as current until the push is confirmed.

## 2. Scope and safety

- **Never weaken, skip, or delete a failing test to get a green gate.** Fix the cause or
  `park` the task. A disabled test is a lie told to the next agent.
- **`merge` only when every gate is satisfied:** `verify` green on the rebased branch, all
  required reviews triaged with no open high-severity finding, and the task's own checklist
  done. If a project marks a class of decision as reserved to a human, build everything
  around the reserved core, leave a clearly-tagged pending note, and continue — never
  implement the reserved part yourself.
- **The base branch is untouchable directly.** All work flows through `branch` → `push` →
  `open-pr` → `merge`. Never commit to or force-push the base.
- **Reversible-by-default.** Prefer additive, reversible changes. New dependencies are a
  last resort; if unavoidable, pin exact versions and justify them in the PR.
- **No orphaned background processes.** Never leave detached load generators, dev servers,
  watchers, or busy-loops alive. Any synthetic load dies with the test that needed it (via
  `trap`/`finally`). Orphans starve every later gate on the machine — one stray pool of
  busy-loops once held an entire CI host hostage for hours.

## 3. Judgment

- **Never block; decide, log, move.** When the spec is ambiguous, take the most
  operational, simplest interpretation, record the decision via `log-decision`, and proceed.
  Your replacement for a human stakeholder is the set of canonical documents the
  `roadmap.config.md` names.
- **Circuit-break flaky externals.** If an external provider (a review model, an API) fails
  twice in a row, drop it for the session and continue without it rather than retrying into
  a wall.
- **Distinguish environmental from real failures.** Container-launch or "under load" gate
  failures are often environmental — re-run the single failing unit once before counting it
  as a real strike. But before blaming the environment, sweep for orphaned processes
  (`ps aux | sort -rk3 | head`): a dead sibling's leftovers, not the test, are often the
  cause.
- **Treat digests and notifications as leads, not facts.** A resume digest or a hand-off
  describes state at write time, and the writer may have died mid-action. Before acting on a
  claim ("branch pushed", "PR open"), verify it against the remote.

## 4. Craft and communication

- **The output contract is strict.** Emit one line per task to the chat
  (`TASK-ID → PR #N merged` or `TASK-ID → parked: <reason, ≤5 words>`). All prose lives in
  artifacts: specs, PR descriptions, tracker comments, memory. Do not narrate, plan aloud,
  or summarize what you are doing.
- **Post evidence before prose.** Every review — especially the last one — is posted via
  `comment-pr` the instant its triage is done, *before* you write any sentence about it.
  Producing a review's text without posting it is the single most common way these agents
  die: the turn ends on the review text and someone else has to finish your job. If your
  last action generated a review, your turn is **not** over.
- **Your final message is the report, never raw tool output.** Before composing it, run the
  **Before you send** checklist below. The report is: PR reference, triage summary, decisions
  one line each, derived work, parked parts. If a review is the last thing you produced and is
  unposted, post it first.

### Before you send

Run this every time, before you compose any final message. It is the single gate the rest of
these standards converge on — most agent deaths are a missed line here:

1. **Unposted evidence?** If your last action produced a review or any evidence, it is posted
   via `comment-pr` and the URL came back. If not, post it now — your turn is **not** over.
2. **Background gate?** Nothing load-bearing is still running in the background. Gates ran in
   the foreground with output captured to a file, and you read the file (§1).
3. **Claims observed, not assumed?** Every "done" was seen, not hoped: `push` accepted (remote
   tip == local tip), `verify` green against committed state, `merge` confirmed to contain
   your work.
4. **Bookkeeping done?** `complete`, `log-decision`, `derive`, and any pending-human `note`
   are written to artifacts — not merely described in chat.
5. **Then, and only then, the report.** One line per item to chat (the output contract above);
   all prose lives in artifacts.

Failing any check means you are not done: fix it before sending, don't narrate it.

## 5. Working in a shared or parallel checkout

These apply whenever a fleet runs agents in parallel against one repository.

- **Lock contention is real.** `branch`/`commit`/`checkout` can fail when siblings hold the
  index lock — retry after a few seconds. Critically, **verify the current branch
  (`git branch --show-current`) before any gate**: a silently-failed checkout leaves you on
  the base branch, and the gate then "passes" on the base instead of your work.
- **Commit early, push early.** If you crash, recovery starts from the pushed branch.
  Work-in-progress commits are fine; they get squashed at merge.
- **Worktrees change tool context.** Directory-named tooling (compose projects, caches)
  resolves differently inside a worktree. Run such commands from the real repository root or
  pass an explicit project/path. The shell's working directory dies with a removed worktree —
  anchor every command with an absolute path.

## 6. The strictly-serial merge queue (fleet only)

This is what makes parallel execution safe. The orchestrator merges **one PR at a time**,
in completion order. For each ready PR:

1. **Rebase** onto the fresh remote base. Apply the project's conflict protocol: regenerate
   (never hand-merge) generated artifacts; resolve ordering collisions per the config;
   union additive data (e.g. translation keys); re-run the **full** `verify` after any
   source conflict. Anchor commands with an absolute path and verify the current branch
   before any gate.
2. **`verify`** with full log capture to a file — never trust a piped exit code. Perform any
   deferred manual QA now.
3. If the queue added fix commits: **`push` and confirm acceptance** (remote tip == local
   tip) *before* `merge`. A rejected push plus a squash-merge silently drops the fixes.
4. **`merge`**, then sync the base and confirm the merged commit actually contains what you
   think it does when fixes were involved. **Only now** remove the agent's worktree and local
   branch — never on a completion notification, which can fire while the agent is still
   reviving; removing a live agent's worktree kills it.
5. **Bookkeeping:** `complete` the task, post its decision lines via `log-decision`, `derive`
   any follow-up work, post pending-human notes.
6. **Refill the lane:** `claim` the next fitting task and spawn.

If steps 1–2 fail twice on the same PR for non-environmental reasons, `park` it and free the
lane.

## 7. Crash recovery (fleet only)

Agents die ending their turn on a background gate or on review text.

- **General case:** a "completed" notification whose final message is mid-flight means the
  agent died. Inspect its worktree, commit+push any state as WIP, and spawn a **finisher**
  scoped to the remaining steps (verify → reviews-posted-on-PR → open-pr → report). Never
  reimplement; never let two agents drive one branch — the finisher owns it, and later ghost
  revivals of the dead agent are read-only echoes.
- **Most common death — died on the review text:** the final message is a finished review
  instead of the report. The PR is almost always complete (branch pushed, earlier reviews
  posted) — only this review comment and the report are missing. Do **not** spawn a finisher:
  the orchestrator posts the review from the agent's result via `comment-pr` (with a one-line
  attribution note), confirms remote tip == worktree tip, and runs the normal queue pass.
  Diagnose by reading the PR's comments and the worktree before assuming anything is missing.

## 8. Cost policy (fleet only)

- Implementation agents and any triage of judgment inherit the session's default model —
  they need full judgment.
- Spawn **mechanical** verification agents on a cheaper model (e.g. `sonnet`). Mechanical =
  the outcome is extractive or binary, not a judgment call: evidence extraction ("did review
  X run? quote the verdict"), data cross-checks (key-completeness, id↔enum parity),
  "does X exist" sweeps, false-positive filtering against an explicit exclusion list.
- When unsure whether a check is mechanical, it isn't — use the default. Triage decisions are
  never delegated to the cheap model.

## 9. Stop conditions (loop guards)

Autopilot runs to completion, then stops — it is not a daemon.

- **Mission complete** — the only remaining ready work is blocked by an external wall
  (missing credentials/infra) or reserved for a human. Write the final digest and end. If
  launched on a loop, do **not** reschedule; this is the intended terminal state.
- **Session boundary** — a context summarization happened or context is degrading. Finish the
  current task cleanly, write a resume digest, and stop. On a loop you resume next firing;
  the queue state lives in the roadmap source, not in your memory.
- **Safety** — two consecutive failed gates on the same task: `park` it and take the next.
  Two consecutive parked tasks in a row: likely systemic — write a digest and stop rather
  than burning the queue.
