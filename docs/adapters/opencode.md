# OpenCode Adapter

OpenCode hosts Autopilot through its **skill system** with optional **command wrappers**.
Skills provide the reusable instruction bundles; command wrappers provide convenient slash
command shortcuts.

See `../agent-host-contract.md` for the capabilities contract this adapter fulfills.

## Installation

### Repo-local

```bash
mkdir -p .opencode/skills .opencode/commands .opencode/autopilot
cp -R /path/to/autopilot/adapters/opencode/skills/* .opencode/skills/
cp -R /path/to/autopilot/adapters/opencode/commands/* .opencode/commands/
cp -R /path/to/autopilot/examples .opencode/autopilot/examples
```

Skills and commands installed repo-locally are available when OpenCode runs in that
repository. The copied `examples/` directory is required by `autopilot-plan` when it scaffolds
configs and roadmap files.

### Global

```bash
mkdir -p ~/.config/opencode/skills ~/.config/opencode/commands ~/.config/opencode/autopilot
cp -R /path/to/autopilot/adapters/opencode/skills/* ~/.config/opencode/skills/
cp -R /path/to/autopilot/adapters/opencode/commands/* ~/.config/opencode/commands/
cp -R /path/to/autopilot/examples ~/.config/opencode/autopilot/examples
```

Globally installed skills and commands are available in every repository. Keep the support
examples updated with the skills and commands when upgrading the adapter.

## Skill Structure

The OpenCode adapter packages Autopilot as these skills under
`adapters/opencode/skills/`:

| Directory | Skill | Role |
|-----------|-------|------|
| `autopilot-plan/SKILL.md` | `autopilot-plan` | Scaffold a roadmap initiative (interactive setup) |
| `autopilot-solo/SKILL.md` | `autopilot-solo` | Single-agent roadmap execution |
| `autopilot-fleet/SKILL.md` | `autopilot-fleet` | Parallel roadmap execution with serial merge queue |
| `autopilot-task/SKILL.md` | `autopilot-task` | One roadmap item end-to-end |
| `autopilot-standards/SKILL.md` | `autopilot-standards` | Discipline layer (survival rules, merge protocol, crash recovery) |

Each skill's YAML frontmatter (`name:`, `description:`) drives OpenCode's skill discovery.

`autopilot-plan` reads its scaffolding templates from the copied support examples at
`.opencode/autopilot/examples/` or `~/.config/opencode/autopilot/examples/`.

## Optional Command Wrappers

Under `adapters/opencode/commands/`, optional command wrappers provide slash-command
shortcuts. Each is a thin `.md` file that loads the corresponding skill.

| File | Slash Command | Loads |
|------|-------------|-------|
| `autopilot-plan.md` | `/autopilot-plan` | `autopilot-plan` skill |
| `autopilot-solo.md` | `/autopilot-solo` | `autopilot-solo` skill |
| `autopilot-fleet.md` | `/autopilot-fleet` | `autopilot-fleet` skill |
| `autopilot-task.md` | `/autopilot-task` | `autopilot-task` skill |

Commands are optional — OpenCode can invoke Autopilot through skills directly. Commands
provide convenience for users who prefer slash-command syntax.

The wrappers are included in `adapters/opencode/commands/` and can be omitted if the user
prefers direct skill invocation only.

## Usage

### Via Skills

OpenCode discovers skills from `.opencode/skills/` (repo-local) or
`~/.config/opencode/skills/` (global). Describe the task and OpenCode loads the matching
skill.

### Via Commands

If command wrappers are installed, slash commands are available:

```
/autopilot-plan <initiative>
/autopilot-solo <roadmap-id>
/autopilot-fleet <roadmap-id>
/autopilot-task <item-id>
```

### Project Integration

OpenCode reads `AGENTS.md` for project-level instructions. Autopilot skills must not
duplicate what `AGENTS.md` or `roadmap.config.md` already define.

## Capabilities Overview

OpenCode fulfills the full agent host contract:

| Capability | Status | Notes |
|-----------|--------|-------|
| File system access | ✅ Native | Full read/write |
| Shell commands | ✅ Native | Foreground execution with output |
| Command output capture | ✅ Native | File redirection supported |
| Project instructions | ✅ Native | Reads `AGENTS.md` |
| Skills | ✅ Native | Discovers from `.opencode/skills/` (repo) and `~/.config/opencode/skills/` (global) |
| Commands | ✅ Native | Discovers from `.opencode/commands/` (repo) and `~/.config/opencode/commands/` (global) |
| Subagents | ✅ Native | `task()` tool supports subagent spawning — enables full parallel `fleet` |
| Permissions | ✅ Native | Permission model for tool access |
| Model routing | ✅ Native | Model selection for cost policy |

## What's Different from Claude Code

| Concept | Claude Code | OpenCode |
|---------|------------|----------|
| Skills | Namespaced by plugin (`autopilot:task`) | Filename-prefixed (`autopilot-task`) |
| Commands | `commands/*.md` with plugin prefix (`/autopilot:plan`) | `.opencode/commands/*.md` with kebab prefix (`/autopilot-plan`) |
| Plugin manifest | `.claude-plugin/plugin.json` | Not used (skills/commands are discovered directly) |
| Config directory | `.claude/` | `.opencode/` |
| Global config | N/A (plugins are project or marketplace) | `~/.config/opencode/` |

Skills and commands are separate concepts in OpenCode, whereas Claude Code has commands
that load skills. The OpenCode adapter provides both.

## Guardrails

OpenCode permissions, sandboxing, and hooks are host-level concerns. Recommended:

- Restrict shell command access to the project's verified toolchain
- Prevent direct writes to protected branches
- Configure subagent permissions to match the fleet cost policy

### OpenCode-Specific Delegation

If the OpenCode host uses a planner/orchestrator agent, the OpenCode adapter may include
host-specific instructions that require implementation delegation (e.g., "DELEGATE ALL
implementation via task(category=...)"). These delegation rules are **adapter-local** —
they live in the OpenCode adapter's skill files only and do not affect Autopilot core or
other adapters. The core `autopilot:task` and `autopilot:standards` remain host-agnostic.
