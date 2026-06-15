# AGENTS.md

Guidance for AI coding agents (Claude Code, and any agent that reads `AGENTS.md`) working in
this repository.

## What this is

A Claude Code plugin — a **pure-Markdown engine** with no build system, no package manager,
and no test runner. Every component is a `.md` file loaded by the agent host at runtime. There
is nothing to compile and no dependencies to install.

## Running locally

```
claude --plugin-dir .
```

Confirm it loaded: `/help` should list `/autopilot:init`, `/autopilot:solo`, and
`/autopilot:fleet`, and the `autopilot:task` / `autopilot:standards` skills should be available.

## Architecture

The engine is **project-agnostic**. It speaks abstract verbs only; a consuming project's
`roadmap.config.md` binds those verbs to real tools. Three layers:

```
Engine (this plugin)      commands/solo · commands/fleet · skills/task · skills/standards
   │  speaks abstract verbs only
Project config            roadmap.config.md  +  bindings  (examples/ has copyable templates)
   │  resolves verbs → real tools
Your repository           the roadmap actually gets executed
```

### Components and their roles

| File | Invocation | Role |
|------|-----------|------|
| `commands/init.md` + `skills/init/SKILL.md` | `/autopilot:init` | Setup layer: autodetect the stack and scaffold the consumer's `roadmap.config.md` (the one interactive command) |
| `commands/solo.md` | `/autopilot:solo` | Single-agent: works items one at a time until a stop condition |
| `commands/fleet.md` | `/autopilot:fleet` | Orchestrator: claims per lane, spawns worktree subagents, runs serial merge queue |
| `skills/task/SKILL.md` | `autopilot:task` | One roadmap item end-to-end (pick → spec gate → implement → verify → review → PR → merge → bookkeeping) |
| `skills/standards/SKILL.md` | `autopilot:standards` | Discipline layer: survival rules, merge protocol, crash recovery, communication contract |

**Key design constraint:** `skills/standards` is the fixed layer — it is never duplicated in
other components, only referenced. `skills/task` and the runtime commands load
`autopilot:standards` first and defer to it. Similarly, tool names (`gh`, `acli`, `glab`,
`make`) never appear in the **runtime** engine — they belong in the consuming project's
`roadmap.config.md`. The deliberate exception is the **setup layer** (`commands/init.md` +
`skills/init/`): `init` inspects the repo and names real tools to *produce* that config, so it —
like `examples/bindings/` — is where tool names legitimately live. It also is the one
interactive command: `solo` and `fleet` never ask the user anything.

### The verb contract

The engine emits these verbs; bindings resolve them to real commands:

- **Roadmap-source:** `next-ready`, `claim`, `complete`, `park`, `note`, `log-decision`, `derive`
- **Code-host:** `branch`, `push`, `open-pr`, `comment-pr`, `merge`

### Plugin manifest

`.claude-plugin/plugin.json` is the plugin descriptor. `.claude-plugin/marketplace.json` is the
marketplace entry. Neither contains logic.

## Writing config for a consuming project

Consumers drop a `roadmap.config.md` at their repo root. Start from
`examples/roadmap.config.example.md`. Required sections: `## Source binding`,
`## Code-host binding`, `## Verify`, `## Review ritual`, `## Conventions`,
`## Reserved decisions`, `## Queue`, `## Canonical docs`. Optional: `## Spec gate`,
`## Environment gotchas`.

Pre-built binding snippets live in `examples/bindings/` — paste and adjust:
- **Source:** `jira.md` (`acli`), `github-issues.md` (`gh`), `markdown.md` (checklist, no tracker)
- **Code-host:** `github.md` (`gh`), `gitlab.md` (`glab`)

## Key invariants to preserve when editing

- `autopilot:standards` must remain the single authoritative source for all discipline rules.
  Never restate a rule from it in another component — reference the section (e.g. "per
  `autopilot:standards` §6").
- The **runtime** engine (`commands/solo`, `commands/fleet`, `skills/task`,
  `skills/standards`) must never name a specific tool (`gh`, `git`, `make`, etc.) — those
  references belong in the consuming project's config, in `examples/bindings/`, or in the
  **setup layer** (`commands/init.md` + `skills/init/`), which by design inspects the repo and
  names tools to scaffold that config.
- `skills/task` drives one item; the fleet orchestrator drives the merge queue. This separation
  is load-bearing: fleet subagents do **not** merge — they stop after PR open and triage; the
  orchestrator's serial merge queue handles everything else.
- The user-facing walkthrough is `README.md`; the deep schema reference is
  `docs/config-schema.md`. When you change a section name or the verb contract, update
  `docs/config-schema.md` in the same change — it is the schema of record for consumer configs.
