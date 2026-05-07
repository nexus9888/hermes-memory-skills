# Hermes Agent Memory Skills (Memory-Agnostic)

Memory hygiene skills for [Hermes Agent](https://github.com/NousResearch/hermes-agent) that work with **any memory backend** — built-in MEMORY.md and Holographic.

These are the memory-agnostic versions of the original [hermes-memory-skills](https://github.com/nexus9888/hermes-memory-skills). They auto-detect which memory provider is active and route operations to the correct toolset.

## Why Memory-Agnostic?

The original skills assume the built-in `memory()` tool — they read/write `MEMORY.md` directly. That works great for the default backend, but Hermes Agent supports 8 external memory providers. Holographic, the most interesting one, stores facts in SQLite with HRR vector algebra and trust scoring. It has completely different tools (`fact_store`, `fact_feedback`), no character limit, and no `§` delimiter format.

Rather than making users choose between backends OR skills, these skills detect the active backend at runtime and adapt:

| Backend | Detection | Phase 2 tool | Capacity model | Bloat handling |
|---------|-----------|-------------|----------------|----------------|
| **Built-in** (default) | `memory.provider` empty/null | `memory()` | 2,200 char limit | `memory-lean-check` trimming |
| **Holographic** | `memory.provider: holographic` | `fact_store()` + `fact_feedback()` | Trust scores (0.0–1.0) | `fact_feedback(action='unhelpful')` decay |
| **Honcho, Mem0, etc.** | Detected but unsupported | Falls back to built-in `memory()` | Built-in limits apply | Built-in trimming |

## Skills

### 🌙 Agent Dreaming (Memory-Agnostic)

Three-phase background memory consolidation — same Light/Deep/REM structure as the original, but with backend detection as Phase 0 and dual routing in Phase 2.

```
Phase 0: Detect backend (built-in or holographic)
Phase 1: Light — review sessions, stage candidates
Phase 2: Deep — score candidates, promote via correct backend tools
Phase 3: REM  — extract patterns, propose structural changes
```

**What's different from the original:**
- Phase 0 backend detection from `config.yaml`
- Phase 2 routes to `memory()` or `fact_store()`/`fact_feedback()` automatically
- Capacity check adapts: char limit for built-in, trust score distribution for holographic
- New entries in holographic always include `entities` for compositional recall
- Honcho/Mem0/other backends gracefully fall back to built-in

### ✂️ Memory Lean Check (Memory-Agnostic)

Memory auditing that adapts to the backend. For built-in: the same surgical trimmer as the
original — condenses verbose entries to wiki pointers, enforces the 2,200 char limit. For
holographic: audits trust scores, decays stale facts, removes dead facts below
`min_trust_threshold`, and validates wiki pointers.

```
Built-in:   read MEMORY.md → validate pointers → condense to wiki → § integrity check
Holographic: fact_store(list) → audit trust/retrieval → decay stale → remove dead → contradictions
```

**What's different from my original:**
- Phase 0 backend detection from `config.yaml`
- Built-in path is unchanged (same §-safe trimming)
- Holographic path: trust-score buckets (healthy/stale/low-trust/suspicious)
- `fact_feedback(action='unhelpful')` gently decays stale facts (−0.10 per flag)
- `fact_store(action='contradict')` finds conflicting claims
- Quick MEMORY.md pass even when holographic is active (it still runs)
- ASCII-art report with per-bucket counts

## Installation

```bash
# Clone the repo
git clone https://github.com/nexus9888/hermes-memory-skills.git
cd hermes-memory-skills

# Copy the skills into your Hermes skills directory
cp -r agent-dreaming-agnostic ~/.hermes/skills/management/
cp -r memory-lean-check-agnostic ~/.hermes/skills/management/

# Verify they're installed
hermes skills list | grep agnostic
```

Or install directly from the repo:

```bash
hermes skills tap add https://github.com/nexus9888/hermes-memory-skills
hermes skills install agent-dreaming-agnostic
hermes skills install memory-lean-check-agnostic
```

## Cron Setup

Recommended schedule (works regardless of backend):

```bash
# Dream every 6 hours
hermes cron create \
  --schedule "0 */6 * * *" \
  --name "agent-dreaming" \
  --skill agent-dreaming-agnostic \
  "Run the memory consolidation with the active backend"

# Audit memory daily — works for both backends
hermes cron create \
  --schedule "0 3 * * *" \
  --name "memory-lean-check" \
  --skill memory-lean-check-agnostic \
  "Surgical memory audit with the active backend"
```

## Requirements

- Hermes Agent (any version with `memory` tool)
- `session_search` tool enabled
- `llm-wiki` skill installed (for wiki pointer creation in Phase 2)
- For Holographic backend: `memory.provider: holographic` in config.yaml + plugin installed

## How It Detects the Backend

The skill reads `memory.provider` from `$HERMES_HOME/config.yaml`:

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
│  Phase 3: REM (pattern extraction)                   │
│    ├─ cross-dream pattern detection                  │
│    ├─ propose structural changes                     │
│    └─ user approval gate                             │
└─────────────────────────────────────────────────────┘
                         │
                         ▼ produces entries
┌─────────────────────────────────────────────────────┐
│         memory-lean-check-agnostic                   │
│                                                      │
│  Phase 0: Backend Detection (same as dreaming)       │
│                                                      │
│  Built-in path:                                      │
│    read MEMORY.md → validate pointers → condense     │
│    to wiki → § integrity check → report              │
│                                                      │
│  Holographic path:                                   │
│    fact_store(list) → trust/retrieval audit          │
│    stale → fact_feedback(unhelpful)                  │
│    dead  → fact_store(remove)                        │
│    contradictions → flag for review                  │
│    MEMORY.md quick pass → report                     │
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
