---
name: agent-dreaming
description: "DEPRECATED — use agent-dreaming-agnostic instead. Background memory consolidation for built-in MEMORY.md only."
tags: [memory, consolidation, dreaming, introspection, maintenance, deprecated]
triggers: ["dream", "consolidate memory", "run dreaming", "memory consolidation"]
---

# Agent Dreaming (DEPRECATED)

> **⚠️ This skill is deprecated.** Use [`agent-dreaming-agnostic`](../agent-dreaming-agnostic/SKILL.md) instead.
> It auto-detects your memory backend (built-in or Holographic) and includes Phase 2.5 Condensation.
>
> This version only works with the built-in MEMORY.md backend and relies on the
> now-removed `memory-lean-check` skill for condensation. The agnostic version handles
> both backends and has condensation built in.

---

See [`agent-dreaming-agnostic/SKILL.md`](../agent-dreaming-agnostic/SKILL.md) for the current version.

**Migration:** Replace any cron job or skill reference from `agent-dreaming` to
`agent-dreaming-agnostic`. The interface is identical — same phases, same triggers,
just with backend auto-detection added.
