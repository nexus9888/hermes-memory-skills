# Holographic Memory Backend Reference

Reference for the Holographic memory provider tools, schema, and configuration.
Used by `agent-dreaming-agnostic` Phase 2 routing.

## Tools

### `fact_store` — Primary fact storage and retrieval

9 actions, one tool:

| Action | Purpose | Required params | Returns |
|--------|---------|----------------|---------|
| `add` | Store a new fact | `content`, `entities` | `{fact_id, trust}` |
| `search` | Keyword lookup | `query` | Matching facts with scores |
| `probe` | All facts about an entity | `entity` | Entity's fact bundle |
| `related` | What connects to an entity | `entity` | Structural adjacency |
| `reason` | Compositional: facts connecting MULTIPLE entities | `entities` | Intersection results |
| `contradict` | Find conflicting facts | `query` or `entities` | Contradiction pairs |
| `update` | Modify a fact | `fact_id`, `content` | Updated fact |
| `remove` | Delete a fact | `fact_id` | Confirmation |
| `list` | List all facts | none | Full fact list |

Optional params: `category` (user_pref/project/tool/general), `tags` (comma-separated), `min_trust` (filter threshold, default 0.3), `limit` (max results, default 10).

### `fact_feedback` — Trust score training

| Action | Purpose | Required params | Effect |
|--------|---------|----------------|--------|
| `helpful` | Mark fact as accurate | `fact_id` | Increases trust score |
| `unhelpful` | Mark fact as outdated/wrong | `fact_id` | Decreases trust score |

Facts below `min_trust_threshold` are filtered from retrieval but remain in storage.
Trust scores range from 0.0 (untrusted) to 1.0 (fully trusted).

## Configuration

In `$HERMES_HOME/config.yaml`:

```yaml
memory:
  provider: holographic

plugins:
  hermes-memory-store:
    db_path: $HERMES_HOME/memory_store.db   # SQLite path
    auto_extract: false                      # Auto-extract facts from conversations
    default_trust: 0.5                       # Trust score for new facts (0.0–1.0)
    min_trust_threshold: 0.3                 # Minimum trust for retrieval
    temporal_decay_half_life: 0              # Time-based decay in days (0 = disabled)
    hrr_dim: 1024                            # HRR vector dimensionality
```

## HRR (Holographic Reduced Representations)

Holographic memory uses complex-valued vectors and HRR algebra:

- **Encoding:** Words → complex unit vectors (random phase angles)
- **Binding:** Circular convolution — pairs key-value associations
- **Bundling:** Vector addition — superposes multiple bindings
- **Unbinding:** Circular correlation with key → approximate value recovery
- **Similarity:** Cosine similarity between vectors

Key properties:
- **Compositional:** `reason(action='reason', entities=['user', 'project'])` finds
  facts that relate to BOTH entities, not just either one.
- **Sub-millisecond retrieval:** HRR algebra is O(n) with n = stored facts.
- **Noisy recall:** As stored facts increase, SNR decreases. The trust scoring
  system is designed to counteract this by decaying low-quality facts.

## Entity Strategy

Entities are the backbone of holographic retrieval. Every fact should be
associated with at least one entity:

| Fact type | Entity | Example |
|-----------|--------|---------|
| User preference | `user` | "Prefers concise responses" |
| Project fact | project name | "Ghostfolio runs on port 3333" |
| Tool quirk | tool name | "Docker needs --network=host for Tailscale" |
| Environment | `environment` | "Server runs Ubuntu 22.04" |
| Cross-cutting | Multiple | `['user', 'openroom']` for "Josh is the maintainer of OpenRoom" |

**Rule of thumb:** If you'd want to recall this fact when probing an entity, that
entity should be in the fact's entity list.

## Comparison with Built-in MEMORY.md

| Feature | Built-in (MEMORY.md) | Holographic |
|---------|---------------------|-------------|
| Storage | Flat markdown file | SQLite database |
| Capacity | 2,200 chars hard limit | Unlimited (practical: thousands of facts) |
| Retrieval | Injected into system prompt (always visible) | On-demand via tool calls |
| Compositional queries | No (grep at best) | Yes (HRR reason/probe/related) |
| Trust/decay | Manual (remove stale entries) | Automatic (trust scoring + feedback) |
| Entry format | Free text, § delimited | Structured: content + category + entities + tags |
| Bloat handling | Inline trimming (built-in) or trust decay (holographic) | `fact_feedback(action='unhelpful')` decay |
| Offline/air-gapped | Yes (local file) | Yes (local SQLite) |
| Multi-profile safe | Via `$HERMES_HOME` | Via `$HERMES_HOME` |
