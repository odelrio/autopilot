# Getting started

This walks you from zero to an autonomous run on your own project.

## 1. Install the plugin

During development, load it directly:

```
claude --plugin-dir /path/to/autopilot
```

To distribute it to a team, publish it as a marketplace and install:

```
/plugin marketplace add <url-or-path>
/plugin install autopilot@autopilot
```

Confirm it loaded: `/help` should list `/autopilot:solo` and `/autopilot:fleet`, and the
`autopilot:task` / `autopilot:standards` skills should be available.

## 2. Write your `roadmap.config.md`

Copy the template to your repository root:

```
cp /path/to/autopilot/examples/roadmap.config.example.md ./roadmap.config.md
```

Then fill in each section (the full schema is in `docs/config-schema.md`):

1. **`## Source binding`** — pick a binding from `examples/bindings/` (`jira.md`,
   `github-issues.md`, or `markdown.md`) and paste its verb table; adjust keys and
   transitions.
2. **`## Code-host binding`** — pick `github.md` or `gitlab.md`; paste its verb table and
   write your conflict protocol.
3. **`## Verify`** — the exact gate command that must pass before review and merge.
4. **`## Review ritual`** — which reviewers run, in what order, and which are mandatory for
   which surfaces.
5. **`## Conventions`** — branch/commit/PR naming and house rules.
6. **`## Reserved decisions`** — what only a human may decide (or leave empty).
7. **`## Spec gate`** *(optional)* — delete it if you have no spec requirement.
8. **`## Queue`** — your dependency-ordered work and the fleet's lane surfaces.
9. **`## Canonical docs`** — the documents that stand in for a stakeholder on ambiguity.
10. **`## Environment gotchas`** *(optional)* — setup traps to try first.

## 3. Do a dry run

Before an unattended session, sanity-check selection with the lightest binding. Put a
`ROADMAP.md` checklist in a scratch directory (see `examples/bindings/markdown.md`) and ask
the agent to resolve `next-ready` — it should pick the first unchecked item whose
dependencies are all checked, touching no tracker or code host.

## 4. Run

- **Single agent:** `/autopilot:solo`. It works the queue one item at a time, posting one
  line per item, until a stop condition.
- **Parallel:** `/autopilot:fleet`. It claims one item per lane, spawns a worktree subagent
  for each, and merges through a strictly-serial queue.
- **Unattended to completion:** launch either on a loop. The command ends itself at mission
  complete and does not reschedule.

## 5. Understand the stop conditions

A run is not a daemon. It stops at **mission complete** (only externally-blocked or reserved
work remains), at a **session boundary** (context degraded — it writes a resume digest and
picks up next time from the roadmap source), or on a **safety** trip (repeated failures). See
`autopilot:standards` §9.

## Where the rules live

The fixed discipline — survival rules, the merge protocol, crash recovery, the communication
contract — is in the `autopilot:standards` skill, not in your config. Your config only
supplies project specifics. You can override a specific standard by saying so explicitly in
the config, but the defaults are what keep long runs alive — change them deliberately.

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
