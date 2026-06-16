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
- **`/autopilot:init <id-or-intent>`** — with a base already present, scaffold a new **epic
  overlay** (its own source binding and queue, reusing the base's code host, verify gate, and
  conventions). The argument is the roadmap id, and it need **not** be a ticket:
  - a **tracker key** (`/autopilot:init TICKET-101`) is used verbatim → `roadmaps/TICKET-101.md`;
  - **free-text intent** (`/autopilot:init redesign the onboarding flow`) — or, with no argument,
    intent already clear from the conversation — is distilled into a short kebab-case slug
    (`onboarding-redesign`) → `roadmaps/onboarding-redesign.md`.

  Either way `init` **proposes the resolved id and waits for your confirmation** before writing.
  The epic then runs via `/autopilot:solo <id>` or `/autopilot:fleet <id>`. See
  `docs/config-schema.md` → *One roadmap or several*. It won't overwrite an existing overlay.
