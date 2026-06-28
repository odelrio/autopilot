# Agent Host Contract

Autopilot is a **host-agnostic roadmap engine**. It speaks in abstract verbs; the project's
`roadmap.config.md` binds those verbs to real tools. The host provides the execution
environment — file access, shell commands, and optionally skills and subagents.

This document defines the contract between Autopilot core and any agent host that runs it
(Claude Code, Codex, OpenCode, or others).

## Required Capabilities

Every agent host MUST provide these capabilities for Autopilot to function.

### 1. File system access

Read and write files in the user's repository — roadmap configs, project source code,
specs, and artifacts. Autopilot reads `roadmap.config.md`, `roadmaps/<id>.md`, `AGENTS.md`,
and the project's canonical docs; it writes code, specs, and session digests.

### 2. Shell command execution

Run foreground shell commands and capture their exit codes and output. Autopilot drives
the project's `## Verify` gate, `## Code-host binding` commands (`branch`, `push`,
`merge`, etc.), and any project-specific tooling through shell commands.

### 3. Command output capture

Capture long-running command output to files rather than relying on in-context buffering.
The engine's verify discipline (`autopilot:standards` §1) requires: `cmd > run.log 2>&1;
echo "exit=$?"`. Piped output discards exit codes; the host must support file redirection.

### 4. Project instruction access

Read the project's `AGENTS.md`, `CLAUDE.md`, or equivalent agent-guidance files.
Autopilot defers to the project's own conventions, commit style, and branch naming —
these live in the project's instruction files, not in Autopilot core.

## Optional Capabilities (Accelerators)

These capabilities are not required but significantly improve the experience.

### Skills / reusable instruction modules

A skill system lets Autopilot package its `autopilot:task`, `autopilot:standards`, and
`autopilot:plan` components as discoverable, reusable instruction bundles. Without skills,
the host can still run Autopilot by loading the relevant markdown files directly.

### Subagents

The ability to spawn helper agents in parallel enables Autopilot's `fleet` command to
work multiple roadmap items concurrently. Hosts without subagent support can still run
`fleet` in a conservative sequential mode — `fleet` degrades gracefully.

### Permission / sandbox / hook enforcement

Host-level guardrails that enforce binary safety checks — blocking `verify` on the base
branch, confirming push acceptance, sweeping for orphaned processes. These complement
Autopilot's semantic guardrails (don't skip verify, don't merge outside configured target).

### Model routing

The ability to route mechanical verification work to cheaper/faster models while keeping
judgment tasks on the default model. Autopilot's cost policy (`autopilot:standards` §8)
uses this where available; without it, all work runs on the default model.

## How Autopilot Uses the Host

### Entrypoints

Autopilot exposes these entrypoints; the host adapter decides how they are invoked:

| Entrypoint | Type | Interactive? | Description |
|-----------|------|-------------|-------------|
| `plan` | Setup | Yes | Scaffolds `roadmap.config.md` and initiative overlays. The one command that may ask questions. |
| `solo` | Runtime | No | Single-agent execution: works roadmap items one at a time until stop condition. |
| `fleet` | Runtime | No | Parallel execution: claims multiple items, spawns subagents, serial merge queue. |
| `task` | Skill | No | One roadmap item end-to-end (claim → implement → verify → review → merge → bookkeeping). Invoked by `solo` and `fleet`. |
| `standards` | Skill | No | Discipline layer: survival rules, merge protocol, crash recovery, communication contract. Loaded by everything else. |

### The Verb Contract

Autopilot core speaks only these verbs. The host never resolves them — the project's
`roadmap.config.md` does. The host only provides the environment to run those resolutions.

**Roadmap-source verbs:** `next-ready`, `list-open`, `claim`, `complete`, `park`, `note`,
`log-decision`, `derive`.

**Code-host verbs:** `branch`, `push`, `open-pr`, `comment-pr`, `merge`, `reconcile`.

## Guardrail Model

### Autopilot Core Guardrails (semantic — enforced by instruction)

- Do not work an un-queued item
- Do not claim an item twice
- Do not skip `verify`
- Do not merge outside the configured target
- Do not ignore reserved decisions
- Do not silently complete a blocked item
- Do not narrate internal roadmap mechanics as product artifacts

### Host/Project Guardrails (execution — enforced by the host)

- Command permissions (which shell commands are allowed)
- File write permissions (which paths are writable)
- Branch protection (prevent direct pushes to mainline)
- Sandboxing (network, filesystem, process isolation)
- Subagent/model policy (which models subagents use)
- Resource limits (CPU, memory, timeouts)
- Hooks/preflight/postflight checks

## Fleet on Different Hosts

### Fleet Degradation

`fleet` is conceptually available on every host, but its implementation varies:

| Host Capability | Fleet Behavior |
|----------------|---------------|
| Subagents + parallel execution | Full parallel: spawn one subagent per claimed item, merge through serial queue |
| No subagents | Sequential fallback: work items one at a time, same as `solo` but with fleet output contract |
| Subagents but no background | Spawn one at a time, wait for completion, then next |

The fleet output contract and merge discipline are identical regardless of execution model —
only throughput changes.

## What Autopilot Does NOT Require

- **No specific tracker or code host.** Bindings live in `roadmap.config.md`.
- **No build system or package manager.** The engine is pure Markdown.
- **No specific checkout or isolation mechanism.** The engine names abstract verbs; the host
  and config handle the execution environment.
- **No daemon or persistent process.** Autopilot runs to completion and stops.
- **No external orchestrator.** Autopilot is self-contained — it reads a config and drives
  work.
