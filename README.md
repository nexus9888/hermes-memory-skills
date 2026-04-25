# Hermes Agent Memory Skills

A pair of complementary skills for [Hermes Agent](https://github.com/NousResearch/hermes-agent) that automate memory hygiene — keeping your agent's `MEMORY.md` lean, accurate, and well-organized.

## Skills

### 🧠 Agent Dreaming
**Three-phase background memory consolidation** modeled on OpenClaw's dreaming metaphor.

- **Light Phase** — Ingests recent session transcripts, identifies candidates for promotion to memory (user corrections, preferences, environment discoveries)
- **Deep Phase** — Scores candidates on Novelty, Durability, Specificity, and Reduction; promotes winners to `MEMORY.md`
- **REM Phase** — Extracts recurring patterns across dream artifacts; proposes structural actions (new wiki pages, skills) and waits for user approval

Best run as a cron job every 6–8 hours, or manually after a burst of activity.

### ✂️ Memory Lean Check
**Surgical memory trimmer** that validates and condenses `MEMORY.md`.

- Validates existing wiki pointers (flags broken links, preserves working ones)
- Condenses verbose entries into wiki pointers where detail already exists
- Removes stale/temporary entries (task progress, TODOs, session outcomes)
- Post-write integrity check ensures no data corruption

Best run after dreaming when memory is near capacity, or as a periodic maintenance task.

## How They Work Together

```
Agent Dreaming (produces) → Memory Lean Check (trims/verifies)
```

Dreaming *adds* better entries by extracting insights from sessions. Lean check *removes* bloat by condensing verbose entries into wiki pointers. Together they keep memory under 30% of the 2,200 character limit while preserving all the signal.

## Installation

Copy the skill directories into your Hermes skills folder:

```bash
cp -r memory-lean-check ~/.hermes/skills/management/
cp -r agent-dreaming ~/.hermes/skills/management/
```

Or install via Hermes CLI if available:
```
/skill install memory-lean-check
/skill install agent-dreaming
```

## Cron Setup

Recommended schedule:

```
# Dream every 6 hours
hermes cron create --schedule "0 */6 * * *" --prompt "Run the agent-dreaming skill"

# Lean check daily
hermes cron create --schedule "0 3 * * *" --prompt "Run the memory-lean-check skill"
```

## Requirements

- Hermes Agent with memory system enabled
- `llm-wiki` skill installed (lean check uses it to create wiki pages when condensing)
- Session history available via `session_search()`

## License

MIT
