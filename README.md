# autopilot

A **roadmap plugin for autonomous engineering work** — a generic engine that reads a project's
roadmap config and drives work items end-to-end across agent hosts. Claude Code is the
native host; Codex and OpenCode adapters are available. It ships a
single-agent executor and a parallel fleet orchestrator, both backed by a strictly-serial
merge queue, crash recovery, and a layer of survival rules earned the hard way running long
unattended sessions.

The engine is **project-agnostic**. It speaks in abstract verbs; a `roadmap.config.md` in
your repository binds those verbs to your actual tools (issue tracker, code host, build
gate). Nothing about your stack lives in the engine — it lives in your config.

## Three layers

```
Autopilot core          roadmap protocol, commands, skills, standards
   │  speaks abstract verbs only; host-agnostic
Host adapter            Claude Code (native) / Codex / OpenCode — installation and invocation shape
   │  packages core instructions for the host
Project config          roadmap.config.md + roadmaps/<id>.md source/code-host bindings
   │  resolves verbs → real tools
Your repository         the roadmap actually gets executed
```

## The verb contract

The engine never names a provider. Your config resolves these:

- **Roadmap-source:** `next-ready`, `list-open`, `claim`, `complete`, `park`, `note`,
  `log-decision`, `derive`.
- **Code-host:** `branch`, `push`, `open-pr`, `comment-pr`, `merge`.

## What's inside

| Component               | Invocation            | Role                                                       |
| ----------------------- | --------------------- | ---------------------------------------------------------- |
| `commands/plan.md`      | `/autopilot:plan`     | Scaffold your `roadmap.config.md` (the one interactive command). |
| `commands/solo.md`      | `/autopilot:solo`     | Single-agent: works the queue one item at a time.          |
| `commands/fleet.md`     | `/autopilot:fleet`    | Orchestrator: lanes, helper agents, serial merge queue.     |
| `skills/task/`          | `autopilot:task`      | One roadmap item end-to-end.                               |
| `skills/standards/`     | `autopilot:standards` | The discipline layer every component references.           |

(Skills are namespaced by the plugin, hence the `autopilot:` prefix.)

## Install

From the marketplace:

```
/plugin marketplace add odelrio/autopilot
/plugin install autopilot@autopilot
```

Confirm it loaded: `/help` should list `/autopilot:plan`, `/autopilot:solo`, and
`/autopilot:fleet`.

Claude Code is the native host. See `docs/adapters/` for Codex and OpenCode installation.

> Developing the plugin itself? Run it from a clone without installing: `claude --plugin-dir .`
> from the repo root.

## Agent hosts

Autopilot is host-agnostic. The same roadmap engine runs on multiple agent hosts:

| Host | Status | Installation |
|------|--------|-------------|
| **Claude Code** | Native | `/plugin marketplace add odelrio/autopilot` — see `docs/adapters/claude-code.md` |
| **Codex** | Adapter | Skill-based — see `docs/adapters/codex.md` |
| **OpenCode** | Adapter | Skill + command — see `docs/adapters/opencode.md` |

Each host adapter documents installation, invocation, and host-specific capabilities.

## Quickstart

1. **Start your first initiative.** From your repository root:

   ```
   /autopilot:plan redesign the onboarding flow
   ```

   `plan` sets up the project's shared **base** — autodetecting code host (GitHub/GitLab), build
   gate, base branch, likely canonical docs, shown for you to correct — then scaffolds the
   initiative as an overlay `roadmaps/<id>.md`. The argument is free-form (a ticket key, an
   intent, or a full source description; bare `/autopilot:plan` infers from the conversation) —
   *the more you say, the fewer questions it asks*. `plan` is the **only** command that asks you
   questions and always confirms before writing; it never overwrites an existing base or overlay.
   See *Example prompts* below.

2. **Fill the TODOs it leaves** in the initiative overlay — chiefly `## Queue` (your
   dependency-ordered work and the fleet's lane surfaces) and `## Reserved decisions`. See
   `docs/config-schema.md` for what each section means.

3. **Run the initiative.**
   - **Single agent:** `/autopilot:solo <id>` — works the queue one item at a time, posting one
     line per item, until a stop condition.
   - **Parallel:** `/autopilot:fleet <id>` — claims one item per lane, uses the host adapter's
     helper-agent model for each, and merges through a strictly-serial queue.
   - **Unattended to completion:** launch either on a loop. It ends itself at mission
     complete and does not reschedule.

## Example prompts

These are **illustrative examples, not required steps** — copy whichever fits. The three slash
commands are the whole interface; you work by **initiatives**, and `/autopilot:plan` starts one.

**Start an initiative — `/autopilot:plan`.** The normal way to work. `plan` sets up the project's
shared plumbing (code host, build gate, review, conventions) the first time and reuses it
afterward, then scaffolds the initiative. The argument is free-form — **the more you tell it, the
fewer questions it asks**; it infers the rest from the repo and the conversation and always
confirms before writing.

```
/autopilot:plan
```
Infer the initiative from the conversation; confirm, or ask what it can't tell.

```
/autopilot:plan TICKET-101
```
A tracker key → a tracker-backed initiative; infers which tracker (Jira / GitHub / GitLab) and confirms.

```
/autopilot:plan redesign the onboarding flow
```
No ticket → distills a slug (`roadmaps/onboarding-redesign.md`) and starts planning the initiative.

```
/autopilot:plan work items are subtasks of Jira epic TICKET-101, via acli; the epic's comments are the log
```
Same as `TICKET-101`, but the pre-stated source skips those questions.

```
/autopilot:plan my roadmap is the change specs under openspec/changes/ — one item per change folder
```
Likewise: a spec-file source (openspec-style), described up front.

Source bindings live in `examples/bindings/` (`jira.md`, `github-issues.md`, `markdown.md`).

**Run an initiative — `/autopilot:solo` / `/autopilot:fleet`.** Launch either on a loop to run
unattended to completion:

```
/autopilot:solo onboarding-redesign
/autopilot:fleet onboarding-redesign
```
The id is the initiative — a ticket key or a derived slug (bare `solo`/`fleet` run the only
initiative if there's just one). `fleet` parallelizes lanes through the strictly-serial merge queue.

## How initiatives are stored

Each initiative is an **overlay** on a shared **base**; a repo accumulates as many as it needs.

- `roadmap.config.md` (root) — the project **base**: shared plumbing (code host, verify gate,
  review ritual, conventions). No source or queue of its own.
- `roadmaps/<id>.md` — an **initiative**: its own `## Source binding` and `## Queue` (plus optional
  overrides). The effective config is base + overlay, section-level override.

`/autopilot:plan` sets up the base once and scaffolds each initiative (see *Example prompts*); the
id is a ticket key or a slug it derives from your intent, which also groups the initiative's
branches as `<type>/<roadmap-slug>/<item-slug>`. One initiative runs per `solo`/`fleet` invocation.
(Legacy: a root config that carries its own source + queue still runs as a single roadmap.) Full
rules: `docs/config-schema.md` → *The model*.

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

Those hooks do **not** live in this engine. (Hooks are a Claude Code feature. See host-specific adapter docs for equivalent mechanisms on other hosts.) A hook is a shell command bound to a real tool
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

`/autopilot:plan` is the fastest way to produce `roadmap.config.md`. To write or tune it by
hand, see **`docs/config-schema.md`** for the full schema and the verb contract. The
`examples/bindings/` directory has ready-made bindings:

- Source: `jira.md` (`acli`), `github-issues.md` (`gh`), `markdown.md` (a checklist, no
  tracker).
- Code-host: `github.md` (`gh`), `gitlab.md` (`glab`).

## License

MIT — see `LICENSE`.
