# Claude Code Adapter

Claude Code is the **native host** for Autopilot. It was the first host, and the current
`commands/` and `skills/` directories at the repo root are Claude Code's packaging format.
Every other host adapter wraps the same core instructions in that host's format.

See `../agent-host-contract.md` for the capabilities contract this adapter fulfills.

## Installation

### From the marketplace

```
/plugin marketplace add odelrio/autopilot
/plugin install autopilot@autopilot
```

Verify: `/help` should list `/autopilot:plan`, `/autopilot:solo`, `/autopilot:fleet`, and
the skills `autopilot:task` / `autopilot:standards`.

### Local development

```bash
claude --plugin-dir .
```

Same verification — `/help` should show the commands and skills.

## How Claude Code Hosts Autopilot

### Commands → Slash Commands

Claude Code discovers slash commands from `commands/*.md`. Each file's YAML frontmatter
provides the description shown in `/help`.

| File | Slash Command | Role |
|------|-------------|------|
| `commands/plan.md` | `/autopilot:plan` | Interactive setup — scaffold `roadmap.config.md` and initiative overlays |
| `commands/solo.md` | `/autopilot:solo` | Single-agent roadmap execution |
| `commands/fleet.md` | `/autopilot:fleet` | Parallel fleet orchestrator with serial merge queue |

A command file is a prompt template: it loads the relevant skills and delegates to them.
For example, `commands/solo.md` loads `autopilot:standards` and `autopilot:task` and sets
the solo execution context.

### Skills → Skill System

Claude Code discovers skills from `skills/*/SKILL.md`. Each skill's YAML frontmatter
(`name:`, `description:`) drives discovery. Skills are namespaced by the plugin —
`autopilot:task`, `autopilot:standards`, `autopilot:plan`.

| Directory | Skill Name | Role |
|-----------|-----------|------|
| `skills/task/SKILL.md` | `autopilot:task` | One roadmap item end-to-end |
| `skills/standards/SKILL.md` | `autopilot:standards` | Discipline layer (survival rules, merge protocol, crash recovery, communication contract) |
| `skills/plan/SKILL.md` | `autopilot:plan` | Setup layer (stack detection, config scaffolding) |

### Plugin Manifest

`.claude-plugin/plugin.json` is the plugin descriptor. It provides metadata (name, version,
author, repository) but no logic. Claude Code reads it for plugin discovery and installation.

## Capabilities Overview

Claude Code fulfills the full agent host contract:

| Capability | Status | Notes |
|-----------|--------|-------|
| File system access | ✅ Native | Full read/write within the workspace |
| Shell commands | ✅ Native | Bash tool with timeout, background support |
| Command output capture | ✅ Native | File redirection (`>`, `2>&1`) works |
| Project instructions | ✅ Native | Reads `CLAUDE.md` and `AGENTS.md` automatically |
| Skills | ✅ Native | `skills/*/SKILL.md` with YAML frontmatter and namespace |
| Subagents | ✅ Native | Task tool supports subagent spawning (`task(description=..., prompt=...)`) |
| Permissions | ✅ Native | Permission system with allow/deny/ask modes |
| Hooks | ✅ Native | PreToolUse, PostToolUse, Stop, SubagentStop hooks |
| Model routing | ✅ Native | Opus/Sonnet/Haiku for cost policy (§8) |

## Guardrails

Claude Code can enforce mechanical safety checks through **hooks**. These live in the
**consumer's** `.claude/settings.json`, not in the Autopilot plugin — hooks name real tools
(`git`, `gh`), which belong in the project layer, not the engine.

Recommended hooks (all from `autopilot:standards`):

- **PreToolUse** before `verify`: block if current branch is the base branch (§5)
- **PostToolUse** after `push`: assert remote tip equals local tip (§1)
- **Stop / SubagentStop**: sweep for orphaned processes (§2)

```jsonc
// .claude/settings.json — in the CONSUMER's project
{
  "hooks": {
    "PreToolUse": [{
      "matcher": "Bash",
      "hooks": [{
        "type": "command",
        "command": ".claude/hooks/guard-gate.sh"
      }]
    }]
  }
}
```

## Claude-Specific Files

These files are specific to the Claude Code adapter and are not used by Codex or OpenCode:

| Path | Purpose |
|------|---------|
| `.claude-plugin/plugin.json` | Plugin manifest |
| `.claude-plugin/marketplace.json` | Marketplace entry |
| `commands/plan.md` | `/autopilot:plan` slash command |
| `commands/solo.md` | `/autopilot:solo` slash command |
| `commands/fleet.md` | `/autopilot:fleet` slash command |
| `skills/plan/SKILL.md` | `autopilot:plan` skill |
| `skills/task/SKILL.md` | `autopilot:task` skill |
| `skills/standards/SKILL.md` | `autopilot:standards` skill |

Other adapters (Codex, OpenCode) wrap the same core logic in their host's format under
`adapters/<host>/`.
