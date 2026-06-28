---
description: Autonomous single-agent roadmap execution — works items end-to-end via autopilot-task, one at a time, without blocking, until a stop condition.
---

You are executing a project's roadmap autonomously. Invoked as `/autopilot-solo` or
`/autopilot-solo <ID>`. Load **`autopilot-standards`** for the discipline layer and drive
each item through **`autopilot-task`**. Work the queue one item at a time until a stop
condition. Never ask the user anything; never wait for anyone.

Resolve the roadmap: `<ID>` given → `roadmaps/<ID>.md`; no `<ID>` → single overlay or
legacy root config. Run the drift check, enforce the output contract, and end at mission
complete, session boundary, or safety stop.
