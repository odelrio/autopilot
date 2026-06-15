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

- **Roadmap-source:** `next-ready`, `claim`, `complete`, `park`, `note`, `log-decision`,
  `derive`.
- **Code-host:** `branch`, `push`, `open-pr`, `comment-pr`, `merge`.

## What's inside

| Component               | Invocation            | Role                                                       |
| ----------------------- | --------------------- | ---------------------------------------------------------- |
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

For local development (run from the repo root), load it directly without installing:

```
claude --plugin-dir .
```

Then drop a `roadmap.config.md` at your repository root (start from
`examples/roadmap.config.example.md`) and run `/autopilot:solo` — or `/autopilot:fleet` for
parallel execution. To run the doable roadmap to completion unattended, launch the command on
a loop; it ends itself at mission complete.

## Writing your config

See **`docs/getting-started.md`** for the walkthrough and **`docs/config-schema.md`** for the
full schema. The `examples/bindings/` directory has ready-made bindings:

- Source: `jira.md` (`acli`), `github-issues.md` (`gh`), `markdown.md` (a checklist, no
  tracker).
- Code-host: `github.md` (`gh`), `gitlab.md` (`glab`).

## License

MIT — see `LICENSE`.
