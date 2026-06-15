---
description: Scaffold this project's roadmap.config.md — autodetect the stack (code host, build gate, base branch), confirm, then write the config (and a starter ROADMAP.md when the source is a Markdown checklist).
---

Set up autopilot for the current repository. Load and run the **`autopilot:init`** skill, which
autodetects what the repo already reveals, confirms every choice with you, and writes
`roadmap.config.md` from the plugin's packaged templates.

This is the one interactive autopilot command — unlike `/autopilot:solo` and `/autopilot:fleet`,
`init` is allowed to ask you questions before it writes anything.

If `roadmap.config.md` already exists, it reports that and stops rather than overwriting your
config.
