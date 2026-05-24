# Hermes Agent Memory Skills (Memory-Agnostic)

Memory hygiene skills for [Hermes Agent](https://github.com/NousResearch/hermes-agent) that work with **any memory backend** — built-in MEMORY.md and Holographic.

These are the memory-agnostic versions of the original [hermes-memory-skills](https://github.com/nexus9888/hermes-memory-skills). They auto-detect which memory provider is active and route operations to the correct toolset.

## Why Memory-Agnostic?

The original skills assume the built-in `memory()` tool — they read/write `MEMORY.md` directly. That works great for the default backend, but Hermes Agent supports 8 external memory providers. Holographic, the most interesting one, stores facts in SQLite with HRR vector algebra and trust scoring. It has completely different tools (`fact_store`, `fact_feedback`), no character limit, and no `§` delimiter format.

Rather than making users choose between backends OR skills, these skills detect the active backend at runtime and adapt:

| Backend | Detection | Phase 2 tool | Capacity model | Bloat handling |
|---------|-----------|-------------|----------------|----------------|
| **Built-in** (default) | `memory.provider` empty/null | `memory()` | 2,200 char limit | Phase 2.5 Condensation (built into dreaming) |
| **Holographic** | `memory.provider: holographic` | `fact_store()` + `fact_feedback()` | Trust scores (0.0–1.0) | `fact_feedback(action='unhelpful')` decay |
| **Honcho, Mem0, etc.** | Detected but unsupported | Falls back to built-in `memory()` | Built-in limits apply | Built-in condensation |

## Skills

### 🌙 Agent Dreaming (Memory-Agnostic)

Three-phase background memory consolidation — same Light/Deep/REM structure as the original, but with backend detection as Phase 0, dual routing in Phase 2, and a built-in **Phase 2.5 Condensation** that replaces the deprecated `memory-lean-check` skill.

```
Phase 0:   Detect backend (built-in or holographic)
Phase 1:   Light — review sessions, stage candidates
Phase 2:   Deep — score candidates, promote via correct backend tools
Phase 2.5: Condensation — trim bloat / decay stale facts (built into dreaming)
Phase 3:   REM — extract patterns, propose structural changes
```

**What's different from the original:**
- Phase 0 backend detection from `config.yaml`
- Phase 2 routes to `memory()` or `fact_store()`/`fact_feedback()` automatically
- Capacity check adapts: char limit for built-in, trust score distribution for holographic
- New entries in holographic preserve entity context using whichever `fact_store` schema is available (`entities` parameter on older/newer plugins, or entity names in `tags`/`content` on current Hermes tool schemas)
- **Phase 2.5 Condensation** built in — no separate `memory-lean-check` skill needed
- Honcho/Mem0/other backends gracefully fall back to built-in

## Installation

```bash
# Clone the repo
git clone https://github.com/nexus9888/hermes-memory-skills.git
cd hermes-memory-skills

# Copy the agnostic dreaming skill into your Hermes skills directory
cp -r agent-dreaming-agnostic ~/.hermes/skills/management/

# Verify it's installed
hermes skills list | grep agnostic
```

Or install the single-file skill directly from the raw `SKILL.md` URL:

```bash
hermes skills install https://raw.githubusercontent.com/nexus9888/hermes-memory-skills/main/agent-dreaming-agnostic/SKILL.md
```

Note: the direct raw install only installs `SKILL.md`. Use the clone/copy method above if you also want the optional `references/holographic-backend.md` file. The generic `hermes skills tap add ... && hermes skills install agent-dreaming-agnostic` flow may not discover this repository unless/until the repo is published in a Hermes skill-index format.

## Cron Setup

Recommended schedule (works regardless of backend):

```bash
# Dream — handles both promotion AND condensation
hermes cron create \
  --name "agent-dreaming" \
  --skill agent-dreaming-agnostic \
  "0 */6 * * *" \
  "Run memory consolidation with the active backend"
```

> **No separate lean-check cron needed.** Phase 2.5 Condensation runs as part of dreaming whenever capacity thresholds are breached (built-in >60%) or every cycle (holographic).

## Requirements

- Hermes Agent (any version with `memory` tool)
- `session_search` tool enabled
- `llm-wiki` skill installed (for wiki pointer creation in Phase 2)
- For Holographic backend: `memory.provider: holographic` in config.yaml + plugin installed

## Compatibility Notes

Hermes and the Holographic plugin have used more than one `fact_store` schema. Some versions accept an explicit `entities=[...]` argument on `add`; current Hermes tool schemas may only expose `content`, `category`, and comma-separated `tags` for `add`, with `entity`/`entities` reserved for `probe`, `related`, and `reason`.

This skill therefore treats explicit `entities` as optional and schema-dependent. If `entities` is unavailable on `fact_store(action='add')`, preserve the same retrieval signal by including canonical entity names in `tags` and, when useful, in compact content text. Never issue a tool call with parameters absent from the active schema.

## How It Detects the Backend

The skill reads `memory.provider` from `$HERMES_HOME/config.yaml` and then verifies which memory tools/parameters are actually available in the running Hermes session:

```yaml
# Built-in (default — no provider set):
memory:
  memory_enabled: true

# Holographic:
memory:
  provider: holographic
```

The detection logic:

```
memory.provider == "" or null → built-in
memory.provider == "holographic" → holographic
memory.provider == "honcho" or "mem0" or ... → built-in (fallback)
```

Fallback is intentional — better to write to MEMORY.md than to nothing.

## Architecture

```
┌─────────────────────────────────────────────────────┐
│           agent-dreaming-agnostic                    │
│                                                      │
│  Phase 0: Backend Detection                          │
│    └─ read memory.provider from config.yaml          │
│                                                      │
│  Phase 1: Light (same regardless of backend)         │
│    ├─ session_search() → recent sessions             │
│    ├─ deep-dive signal sessions                      │
│    └─ stage candidates → dream artifact              │
│                                                      │
│  Phase 2: Deep (backend-routed)                      │
│    ├─ score candidates (4 dimensions)                │
│    ├─ built-in → memory() tool                       │
│    ├─ holographic → fact_store/fact_feedback         │
│    └─ post-promotion capacity check                  │
│                                                      │
│  Phase 2.5: Condensation (built in)                  │
│    ├─ built-in: merge/shorten/remove bloat           │
│    └─ holographic: trust decay + contradiction check │
│                                                      │
│  Phase 3: REM (pattern extraction)                   │
│    ├─ cross-dream pattern detection                  │
│    ├─ propose structural changes                     │
│    └─ user approval gate                             │
└─────────────────────────────────────────────────────┘
```

## License

MIT — same as the original hermes-memory-skills.

## Contributing

This is a shared workspace. If you add support for Honcho, Mem0, or another backend:

1. Update Phase 0 detection with the new provider
2. Add Phase 2 routing for the new backend's tools
3. Update the compatibility matrix in SKILL.md
4. Test with that backend active

PRs welcome.
