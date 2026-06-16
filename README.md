# autopilot

A Claude Code plugin: a **generic engine for autonomous roadmap execution**. It ships a
single-agent executor and a parallel fleet orchestrator, both backed by a strictly-serial
merge queue, crash recovery, and a layer of survival rules earned the hard way running long
unattended sessions.

The engine is **project-agnostic**. It speaks in abstract verbs; a `roadmap.config.md` in
your repository binds those verbs to your actual tools (issue tracker, code host, build
gate). Nothing about your stack lives in the engine — it lives in your config.

## Three layers

```
Engine (this plugin)      commands/solo · commands/fleet · skills/task · skills/standards
   │  speaks abstract verbs only
Project config            roadmap.config.md  +  bindings  (examples/ has copyable templates)
   │  resolves verbs → real tools
Your repository           the roadmap actually gets executed
```

## The verb contract

The engine never names a provider. Your config resolves these:

- **Roadmap-source:** `next-ready`, `list-open`, `claim`, `complete`, `park`, `note`,
  `log-decision`, `derive`.
- **Code-host:** `branch`, `push`, `open-pr`, `comment-pr`, `merge`.

## What's inside

| Component               | Invocation            | Role                                                       |
| ----------------------- | --------------------- | ---------------------------------------------------------- |
| `commands/init.md`      | `/autopilot:init`     | Scaffold your `roadmap.config.md` (the one interactive command). |
| `commands/solo.md`      | `/autopilot:solo`     | Single-agent: works the queue one item at a time.          |
| `commands/fleet.md`     | `/autopilot:fleet`    | Orchestrator: lanes, worktree subagents, serial merge queue.|
| `skills/task/`          | `autopilot:task`      | One roadmap item end-to-end.                               |
| `skills/standards/`     | `autopilot:standards` | The discipline layer every component references.           |

(Skills are namespaced by the plugin, hence the `autopilot:` prefix.)

## Install

From the marketplace:

```
/plugin marketplace add odelrio/autopilot
/plugin install autopilot@autopilot
```

Confirm it loaded: `/help` should list `/autopilot:init`, `/autopilot:solo`, and
`/autopilot:fleet`.

> Developing the plugin itself? Run it from a clone without installing: `claude --plugin-dir .`
> from the repo root.

## Quickstart

1. **Scaffold your config.** From your repository root:

   ```
   /autopilot:init
   ```

   It autodetects what your repo already reveals — code host (GitHub/GitLab), build gate,
   base branch, likely canonical docs — shows you what it found, lets you correct anything,
   then writes `roadmap.config.md` (and a starter `ROADMAP.md` if you pick the Markdown
   checklist source). `init` is the **only** autopilot command that asks you questions;
   `solo` and `fleet` never do. It won't overwrite an existing `roadmap.config.md` — and
   `/autopilot:init TICKET-101` instead scaffolds an epic overlay (see *Multiple roadmaps* below).

2. **Fill the TODOs it leaves.** A few sections only you can supply — chiefly `## Queue`
   (your dependency-ordered work and the fleet's lane surfaces), `## Reserved decisions`, and
   the optional `## Spec gate`. See `docs/config-schema.md` for what each section means.

3. **Run.**
   - **Single agent:** `/autopilot:solo` — works the queue one item at a time, posting one
     line per item, until a stop condition.
   - **Parallel:** `/autopilot:fleet` — claims one item per lane, spawns a worktree subagent
     for each, and merges through a strictly-serial queue.
   - **Unattended to completion:** launch either on a loop. It ends itself at mission
     complete and does not reschedule.

## Example prompts

Autopilot is driven by three slash commands. What changes between projects is the **roadmap
source** — where your work items live — which `init` captures in `## Source binding`. Describe
your source to `init` in plain language and it writes the matching binding.

**Scaffold the config (run once):**

```
/autopilot:init
```

…then tell it where work lives. A few sources, from heaviest to lightest:

- **Jira** — *"Work items are subtasks of Jira epic TICKET-101, driven through `acli`; the
  epic's comments are the decision log."* → see `examples/bindings/jira.md`.
- **GitHub Issues** — *"Work items are issues labeled `roadmap`; the tracking issue is the
  log."* → see `examples/bindings/github-issues.md`.
- **Spec files (openspec-style)** — *"My roadmap is the change specs under `openspec/changes/` —
  each change folder is one item, `done` when its tasks are checked."* Resolves like the
  Markdown source, scanning a directory instead of a single file.
- **A product checklist** — *"My roadmap is the checklist in `PRODUCT.md`."* The lightest
  source: no tracker, items are `- [ ]` lines. → see `examples/bindings/markdown.md`.

**Scaffold an epic overlay** (when one repo carries several roadmaps):

```
/autopilot:init TICKET-101
```

*"Set up an overlay for epic TICKET-101 — it has its own queue and Jira source, but inherits
the project's verify gate and review ritual from the base config."*

**Run a roadmap:**

```
/autopilot:solo               # single roadmap, one item at a time
/autopilot:solo TICKET-101    # a specific epic overlay
/autopilot:fleet TICKET-101   # parallel lanes + strictly-serial merge queue
```

*"Run the PRODUCT.md roadmap solo until you hit something blocked or reserved."* ·
*"Fleet the TICKET-101 epic — parallelize the backend and frontend lanes."* Launch either on a
loop to run unattended to completion.

## Multiple roadmaps in one repo

One repo can run several independent roadmaps (one per initiative/epic). The config splits into
a shared **base** and per-epic **overlays**:

- `roadmap.config.md` (root) — the project base: code host, verify gate, review ritual,
  conventions.
- `roadmaps/<ID>.md` — an epic overlay: that epic's `## Source binding` and `## Queue` (plus
  optional overrides). The effective config is base + overlay, section-level override.

Scaffold a new epic with `/autopilot:init TICKET-101`, then run it with `/autopilot:solo TICKET-101`
or `/autopilot:fleet TICKET-101`. With no `roadmaps/` directory the root file is simply the single
roadmap, exactly as before — overlays are additive. Full rules:
`docs/config-schema.md` → *One roadmap or several*. One epic runs per `solo`/`fleet` invocation.

Two roadmaps may run **at the same time** (two agents, one epic each — a per-roadmap session
marker blocks two runs on the *same* roadmap). To keep their merges from racing, give each
concurrently-run roadmap its own **integration branch**: each epic's PRs target that branch, so
the two serial merge queues never share a branch, and a single `reconcile` merges it to mainline
when the epic completes. See `docs/config-schema.md` → *Concurrent roadmaps and integration
branches*.

## Dry run (optional but recommended)

Before an unattended session, sanity-check selection with the lightest binding. Put a
`ROADMAP.md` checklist in a scratch directory (the shape is in `examples/bindings/markdown.md`)
and ask the agent to resolve `next-ready` — it should pick the first unchecked item whose
dependencies are all checked, touching no tracker or code host. This is the engine's
agnosticism in isolation: dependency-gated selection with zero provider coupling.

## Stop conditions (it's not a daemon)

A run stops on its own. It ends at **mission complete** (only externally-blocked or reserved
work remains), at a **session boundary** (context degraded — it writes a resume digest and
picks up next time from the roadmap source), or on a **safety** trip (repeated failed gates).
Full definitions live in `autopilot:standards` §9.

## Where the rules live

The fixed discipline — survival rules, the merge protocol, crash recovery, the communication
contract — is in the `autopilot:standards` skill, not in your config. Your config supplies only
project specifics. You can override a specific standard by saying so explicitly in the config,
but the defaults are what keep long runs alive — change them deliberately.

## Optional: enforce some checks with hooks

The `autopilot:standards` rules are instructions the agent follows. Most are **judgment**
calls — circuit-breaking a flaky provider, telling an environmental failure from a real one,
deciding under ambiguity — and cannot be mechanized; they stay as prose. But a few are
**mechanical** (a binary pass/fail) and benefit from deterministic enforcement via Claude Code
hooks.

Those hooks do **not** live in this engine. A hook is a shell command bound to a real tool
(`git`, `gh`), and baking tool names into the engine would break the very abstraction the verb
contract protects. They belong in **your** repository's `.claude/settings.json`, alongside
your bindings — the project layer that already knows your tools. Good candidates, all straight
from the standards:

- **`PreToolUse`** before your `verify` command → block if `git branch --show-current` is the
  base branch, so a silently-failed checkout can't "pass" the gate on the base (§5).
- **`PostToolUse`** after your `push` command → assert the remote tip equals the local tip, so
  a rejected push followed by a squash-merge can't silently drop commits (§1).
- **`Stop` / `SubagentStop`** → sweep for orphaned processes (§2), or flag a final message that
  is raw review text instead of the report (§4).

```jsonc
// .claude/settings.json — in YOUR project, not in this plugin.
// Illustrative: refuse to run the verify gate while on the base branch (standards §5).
{
  "hooks": {
    "PreToolUse": [{
      "matcher": "Bash",
      "hooks": [{
        "type": "command",
        "command": ".claude/hooks/guard-gate.sh"  // reads the tool call on stdin; exit 2 blocks
      }]
    }]
  }
}
```

Rule of thumb: hook only the checks whose outcome is binary, and keep the judgment rules as
prose. A hook that needs judgment will fire wrong and train the agent to ignore it.

## Config reference

`/autopilot:init` is the fastest way to produce `roadmap.config.md`. To write or tune it by
hand, see **`docs/config-schema.md`** for the full schema and the verb contract. The
`examples/bindings/` directory has ready-made bindings:

- Source: `jira.md` (`acli`), `github-issues.md` (`gh`), `markdown.md` (a checklist, no
  tracker).
- Code-host: `github.md` (`gh`), `gitlab.md` (`glab`).

## License

MIT — see `LICENSE`.
