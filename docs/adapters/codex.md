# Codex Adapter

Codex (OpenAI's coding agent) hosts Autopilot through its **skill system**. Codex does not
have a native "slash command" concept — Autopilot's entrypoints are packaged as discoverable
skills that the agent loads on demand.

See `../agent-host-contract.md` for the capabilities contract this adapter fulfills.

## Installation

### Repo-local

```bash
mkdir -p .agents/skills .agents/autopilot
cp -R /path/to/autopilot/adapters/codex/skills/* .agents/skills/
cp -R /path/to/autopilot/examples .agents/autopilot/examples
```

Skills installed repo-locally are available when Codex runs in that repository. The copied
`examples/` directory is required by `autopilot-plan` when it scaffolds configs and roadmap
files.

### Global

```bash
mkdir -p ~/.agents/skills ~/.agents/autopilot
cp -R /path/to/autopilot/adapters/codex/skills/* ~/.agents/skills/
cp -R /path/to/autopilot/examples ~/.agents/autopilot/examples
```

Globally installed skills are available in every repository. Keep the support examples updated
with the skills when upgrading the adapter.

### Verifying installation

Run `/skills` in Codex to list available skills. Autopilot skills appear with the
`autopilot-*` prefix.

## Skill Structure

The Codex adapter packages Autopilot as these skills under `adapters/codex/skills/`:

| Directory | Skill | Role |
|-----------|-------|------|
| `autopilot-plan/SKILL.md` | `autopilot-plan` | Scaffold a roadmap initiative (interactive setup) |
| `autopilot-solo/SKILL.md` | `autopilot-solo` | Single-agent roadmap execution |
| `autopilot-fleet/SKILL.md` | `autopilot-fleet` | Parallel roadmap execution with serial merge queue |
| `autopilot-task/SKILL.md` | `autopilot-task` | One roadmap item end-to-end |
| `autopilot-standards/SKILL.md` | `autopilot-standards` | Discipline layer (survival rules, merge protocol, crash recovery) |

Each skill's YAML frontmatter (`name:`, `description:`) drives Codex's skill discovery.
The skill body preserves the same core instructions as the Claude Code originals, with
invocation wording adapted for Codex — Codex invokes skills via `$autopilot-*` syntax
rather than `/autopilot:*` slash commands.

`autopilot-plan` reads its scaffolding templates from the copied support examples at
`.agents/autopilot/examples/` or `~/.agents/autopilot/examples/`.

## Usage

### Discovery

```
/skills
```

Lists all available skills. Autopilot skills appear as `autopilot-plan`,
`autopilot-solo`, `autopilot-fleet`, `autopilot-task`, `autopilot-standards`.

### Invocation

Explicit skill mention triggers loading:

```
$autopilot-solo <roadmap-id>
$autopilot-fleet <roadmap-id>
$autopilot-task <item-id>
$autopilot-plan <initiative description>
```

Or describe the task and Codex will load the matching skill:

```
> Run the onboarding-redesign roadmap solo
```

### Project Integration

Codex reads `AGENTS.md` for project-level instructions — conventions, branch naming, commit
style. Autopilot skills must not duplicate what `AGENTS.md` or `roadmap.config.md` already
define. The engine's role is roadmap execution; project rules stay project-owned.

## Capabilities Overview

Codex fulfills the required host contract and provides some optional accelerators:

| Capability | Status | Notes |
|-----------|--------|-------|
| File system access | ✅ Native | Full read/write |
| Shell commands | ✅ Native | Foreground execution with output |
| Command output capture | ✅ Native | File redirection supported |
| Project instructions | ✅ Native | Reads `AGENTS.md` |
| Skills | ✅ Native | Discovers from `.agents/skills/` (repo-local) and `~/.agents/skills/` (global) |
| Subagents | ⚠️ Limited | May have limited or no subagent support; `fleet` degrades to sequential fallback |
| Permissions | ✅ Native | Codex permission model controls tool access |
| Model routing | ✅ Native | Model selection available |

### Fleet Behavior on Codex

If Codex supports background subagents: full parallel `fleet` with one subagent per claimed
item, merge through serial queue.

If Codex does not support subagents: `fleet` runs in sequential mode — works items one at a
time, same semantics as `solo` but with fleet output contract and merge discipline.

## What's Different from Claude Code

| Concept | Claude Code | Codex |
|---------|------------|-------|
| Entrypoints | Slash commands (`/autopilot:plan`) | Skills (`$autopilot-plan`) |
| Packaging | `commands/*.md` + `skills/*/SKILL.md` | `skills/*/SKILL.md` only |
| Plugin manifest | `.claude-plugin/plugin.json` | Not used (Codex discovers skills directly) |
| Skill invocation | Skills loaded by commands or explicitly | Skills loaded by `$name` syntax or context matching |
| Namespace | Plugin-prefixed (`autopilot:task`) | Filename-prefixed (`autopilot-task`) |
| Config directory | `.claude/` | `.agents/` |

The `commands/` directory and `.claude-plugin/` are Claude-specific and are not used by Codex.

## Guardrails

Codex permissions and sandboxing are host-level concerns. Recommended:

- Restrict shell command access to the project's verified toolchain
- Prevent direct writes to protected branches
- Configure resource limits for long-running verify gates

Autopilot's semantic guardrails (don't skip verify, don't merge outside configured target,
don't claim an item twice) are enforced by instruction — the Codex adapter skills carry the
same `autopilot:standards` discipline layer.
