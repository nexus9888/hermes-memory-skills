---
name: memory-lean-check-agnostic
description: "Memory-agnostic surgical trimmer — validates, condenses, and audits memory across backends. For built-in: trims MEMORY.md to wiki pointers. For holographic: audits trust scores, decays stale facts, validates pointers."
tags: [memory, maintenance, trimming, holographic]
triggers: ["lean check", "trim memory", "memory audit", "clean memory"]
---

# Memory Lean Check (Memory-Agnostic)

Keeps memory lean across backends. Auto-detects the active memory provider and
applies the right audit strategy. Partner to `agent-dreaming-agnostic`.

**When to run:** After dreaming when memory needs auditing, or as periodic maintenance.
For built-in users: run when memory is above 60% of the 2,200 char limit.
For holographic users: run weekly to catch trust-score decay and stale facts.

---

## Phase 0: Backend Detection

Same as `agent-dreaming-agnostic`. Detect before acting:

1. **Read the active memory provider** from `$HERMES_HOME/config.yaml`:

   ```bash
   grep -A1 "^memory:" $HERMES_HOME/config.yaml | grep "provider:" | awk '{print $2}'
   ```

2. **Set backend mode:**

   | Config value | Backend | Strategy |
   |---|---|---|
   | `""` (empty), `null`, or absent | **built-in** | Trim MEMORY.md, condense to wiki pointers, enforce char limit |
   | `holographic` | **holographic** | Audit trust scores, decay stale facts, validate pointers, quick MEMORY.md pass |
   | Other | **unknown** | Fall back to built-in strategy |

3. **For holographic:** Read plugin config for thresholds:
   ```bash
   grep -A6 "hermes-memory-store:" $HERMES_HOME/config.yaml
   ```
   Note `min_trust_threshold` and `default_trust` — these are your audit baselines.

---

## Built-in Backend Steps

Identical to the original `memory-lean-check` skill:

1. Read `$HERMES_HOME/memories/MEMORY.md`
2. Count current entries: split on `§` and count non-empty segments. **Remember this count.**
3. Determine the wiki path: read `wiki.path` from `$HERMES_HOME/config.yaml`. Fall back to
   `$HERMES_HOME/wiki` if not configured. Never use `$HOME/wiki`. Read `$WIKI/index.md`
   to know what wiki pages exist.
4. **Validate existing pointers first.** For each entry containing `see wiki/`:
   - Extract the wiki path after `see wiki/` (up to next space, `.md`, or `(`)
   - Check if the file exists at `$WIKI/<path>`
   - If the file is **missing**: flag it but do NOT delete — the page may have moved
   - If the file **exists**: leave the entry alone
5. For each remaining entry (non-pointer):
   - **Detailed info** (ports, IPs, config, multi-sentence) → condense to wiki pointer
   - **Short fact, preference, or correction** → leave alone
6. When condensing:
   - If wiki page exists → `Topic: see wiki/path/page.md (brief description).`
   - If no wiki page → load `llm-wiki` skill, create page, then write pointer
   - If detail is trivial/temporary → remove entirely
7. Do NOT condense: already-pointer entries, user preferences, short env facts, convention note
8. Save only if something changed
9. **Post-write integrity check (CRITICAL):**
   - Re-read the file from disk
   - Split on `§` — confirm entry count matches expected count (start − removals)
   - If entry count is wrong, file is corrupted — rewrite from in-memory state
   - Report: final entry count, char usage, any broken pointers
10. Target: under 30% of 2,200 char limit (pointer entries are already compact)

---

## Holographic Backend Steps

Holographic has no character limit, but retrieval quality degrades as noise accumulates.
This audit catches the degradation before it becomes a problem.

### Step 1: Inventory

Call `fact_store(action='list', limit=100)`. For each fact, record:
- `fact_id`, `content`, `category`, `trust_score`, `retrieval_count`, `helpful_count`

Compute summary:
```
Total facts: N
Trust distribution: 0.0-0.3: X, 0.3-0.5: Y, 0.5-0.7: Z, 0.7-1.0: W
Never retrieved (retrieval_count=0): M
Below min_trust_threshold (0.3): K
Avg trust: X.XX
```

### Step 2: Validate Wiki Pointers

For each fact whose `content` contains `see wiki/`:
- Extract the wiki path (same pattern as built-in: up to next space, `.md`, or `(`)
- Check if file exists at `$WIKI/<path>`
- Missing → flag (do not remove — may have moved)
- Exists → leave alone, mark as healthy

Resolve wiki path: read `wiki.path` from `$HERMES_HOME/config.yaml`, fall back to `$HERMES_HOME/wiki`.

### Step 3: Flag Categories

Categorize every fact into one of four buckets:

| Bucket | Criteria | Action |
|--------|----------|--------|
| **Healthy** | `trust_score >= 0.5` AND `retrieval_count > 0` | Leave alone |
| **Stale** | `retrieval_count == 0` AND age > 14 days (from `created_at`) | `fact_feedback(action='unhelpful', fact_id=N)` — decay by 0.10 |
| **Low-trust** | `trust_score < min_trust_threshold` (default 0.3) | Flag for removal — these are invisible to retrieval, dead weight |
| **Suspicious** | `trust_score < 0.5` but `retrieval_count > 0` | Flag for review — used but distrusted. May be outdated but still referenced |

### Step 4: Find Contradictions

Call `fact_store(action='contradict')` to find pairs of facts making conflicting claims.
For each pair:
- Read both facts to understand the conflict
- If one is clearly newer/more accurate → `fact_feedback(action='unhelpful')` on the stale one
- If unclear which is correct → flag both for user review

### Step 5: Execute

For each actionable flag:
- **Stale facts:** `fact_feedback(action='unhelpful', fact_id=N)` — this drops trust by 0.10. A fact at 0.5 goes to 0.4. Two consecutive stale flags drop it below the typical 0.3 threshold at which point it's invisible.
- **Low-trust facts:** Call `fact_store(action='remove', fact_id=N)` — these are below threshold, invisible to retrieval, just wasting space. Only remove if the fact is genuinely obsolete (not a recent addition that hasn't had time to be retrieved).
- **Suspicious facts:** Do NOT auto-remove. Report to user.
- **Contradictions:** Apply feedback on the stale one. Flag unclear pairs for user.

### Step 6: Quick MEMORY.md Pass

MEMORY.md still runs alongside holographic. Do a lightweight check:
- Read `$HERMES_HOME/memories/MEMORY.md`
- Count entries and char usage (for reference — no hard target)
- Ensure entries are wiki pointers where possible (`see wiki/...` or `see fact_store for ...`)
- If memory is above 60%, condense verbose entries to pointers (same rules as built-in Steps 4-9)

### Step 7: Report

```
╔══════════════════════════════════════╗
║   Memory Lean Check (holographic)    ║
╠══════════════════════════════════════╣
║ Total facts:        47               ║
║ Healthy:            34 (72%)         ║
║ Stale (decayed):     3 → trust -0.10 ║
║ Low-trust (removed): 2               ║
║ Suspicious (flagged): 5              ║
║ Contradictions:      1 pair          ║
║ Wiki pointers OK:    8/8             ║
║ MEMORY.md:          420/2,200 (19%)  ║
╚══════════════════════════════════════╝
```

---

## Rules

- **Never delete user preferences or corrections** — in either backend
- **Never add new entries** — this skill only trims/audits existing ones
- **Built-in: § is the entry delimiter, NOT decoration.** Preserve it exactly.
- **Built-in: after ANY write, verify entry count.** If wrong, rewrite from in-memory state.
- **Holographic: never remove facts with `retrieval_count > 0`** — even if low trust. If something is being used, the user needs to decide.
- **Holographic: stale decay is gentle.** A single `unhelpful` feedback drops trust by 0.10. A fact needs 3 consecutive stale flags to drop from 0.5 to 0.2. This prevents one bad audit from nuking useful facts.
- **Holographic: don't chase zero.** The goal isn't zero stale facts — it's keeping retrieval SNR healthy. If 80%+ of facts are healthy, that's good enough.
- **Wiki pointers are sacred.** In either backend, `see wiki/` entries should only be touched if the target page no longer exists.
- **Report what you did and didn't do.** Flagged-but-not-acted items (suspicious facts, unclear contradictions) must appear in the report so the user can act.

---

## Pitfalls

- **Holographic staleness detection needs timestamps.** The `list` action returns `created_at` — use it to filter facts older than 14 days when checking staleness. A fact created 2 hours ago with `retrieval_count=0` is not stale, just new.
- **Holographic: `contradict` may return false positives.** Two facts about different things that happen to share keywords can trigger. Always read both before flagging.
- **Built-in: don't over-trim user preferences.** The temptation is to compress them into terse bullets, but they're "correction memory" — the reasoning matters. Trim filler, keep the why.
- **Backend confusion.** If `memory.provider: holographic` is set but the plugin isn't loaded (tools not available), `fact_store` calls will fail. Verify plugin availability first: `hermes plugins` → check "hermes-memory-store" is active.
- **Double-counting entries.** When both MEMORY.md and holographic are active, the same knowledge may exist in both. That's fine — MEMORY.md is for always-on context injection, holographic is for on-demand retrieval. Don't try to deduplicate across backends.
