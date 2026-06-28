---
description: Plan a roadmap initiative — set up the shared base (code host, build gate, conventions) the first time, then scaffold the initiative as its own roadmap file (its own source + queue), choosing whether it runs on mainline or its own integration branch. Autodetects the stack and confirms before writing.
---

Start a roadmap **initiative** in the current repository. Load and run the **`autopilot-plan`**
skill, which autodetects what the repo reveals, confirms every choice with you, and writes from
the plugin's packaged templates.

This is the one interactive autopilot command — unlike `$autopilot-solo` and `$autopilot-fleet`,
`plan` may ask you questions before writing, and it always confirms the resolved initiative id
first.

The argument is **free-form — the more you say, the fewer questions it asks**:

- `/autopilot-plan` — infer the initiative from the conversation; confirm, or ask what it can't tell.
- `/autopilot-plan TICKET-101` — a tracker-backed initiative; infers which tracker and confirms.
- `/autopilot-plan redesign the onboarding flow` — no ticket; distills a kebab-case slug.

The initiative then runs via `/autopilot-solo <id>` or `/autopilot-fleet <id>`.
