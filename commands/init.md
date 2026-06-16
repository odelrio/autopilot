---
description: Scaffold this project's roadmap.config.md — autodetect the stack (code host, build gate, base branch), confirm, then write the config (and a starter ROADMAP.md when the source is a Markdown checklist).
---

Set up autopilot for the current repository. Load and run the **`autopilot:init`** skill, which
autodetects what the repo already reveals, confirms every choice with you, and writes
`roadmap.config.md` from the plugin's packaged templates.

This is the one interactive autopilot command — unlike `/autopilot:solo` and `/autopilot:fleet`,
`init` is allowed to ask you questions before it writes anything.

Two ways to run it:

- **`/autopilot:init`** — first-time setup: autodetect and scaffold the project base
  `roadmap.config.md`. If a base already exists, it reports that and stops rather than
  overwriting your config.
- **`/autopilot:init <ID>`** (e.g. `/autopilot:init TICKET-101`) — with a base already present,
  scaffold a new **epic overlay** `roadmaps/<ID>.md` (its own source binding and queue, reusing
  the base's code host, verify gate, and conventions). The epic then runs via
  `/autopilot:solo <ID>` or `/autopilot:fleet <ID>`. See `docs/config-schema.md` →
  *One roadmap or several*. It won't overwrite an existing overlay.
