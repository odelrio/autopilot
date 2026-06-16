---
description: Start a roadmap initiative — set up the shared base (code host, build gate, conventions) the first time, then scaffold the initiative as an overlay (its own source + queue). Autodetects the stack and confirms before writing.
---

Start a roadmap **initiative** in the current repository. Load and run the **`autopilot:init`**
skill, which autodetects what the repo reveals, confirms every choice with you, and writes from
the plugin's packaged templates. It does two things: sets up the project's shared **base**
(`roadmap.config.md` — code host, verify gate, review ritual, conventions) the first time and
reuses it after, then scaffolds the **initiative** as an overlay `roadmaps/<id>.md` (its own
source + queue). Working by initiatives is the normal way; a repo accumulates as many as it needs.

This is the one interactive autopilot command — unlike `/autopilot:solo` and `/autopilot:fleet`,
`init` may ask you questions before writing, and it always confirms the resolved initiative id
first.

The argument is **free-form — the more you say, the fewer questions it asks** (it infers the rest
from the repo and the conversation):

- `/autopilot:init` — infer the initiative from the conversation; confirm, or ask what it can't tell.
- `/autopilot:init TICKET-101` — a tracker-backed initiative; infers which tracker and confirms.
- `/autopilot:init redesign the onboarding flow` — no ticket; distills a kebab-case slug
  (`roadmaps/onboarding-redesign.md`) and starts planning the initiative.
- `/autopilot:init work items are subtasks of Jira epic TICKET-101 via acli; the epic's comments are the log`
  — same as the ticket case, but the pre-stated source saves the source questions.
- `/autopilot:init my roadmap is the change specs under openspec/changes/ — one item per change folder`
  — likewise: a spec-file source described up front.

The initiative then runs via `/autopilot:solo <id>` or `/autopilot:fleet <id>`. See
`docs/config-schema.md` for the full model. `init` never overwrites an existing base or overlay.
