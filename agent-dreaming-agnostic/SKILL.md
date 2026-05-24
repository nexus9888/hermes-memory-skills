---
name: agent-dreaming-agnostic
description: "Memory-agnostic background consolidation — reviews sessions, scores candidates, promotes durable insights to the active memory backend (built-in MEMORY.md or Holographic). Three-phase (Light/Deep/Condensation/REM). Run via cron or manually."
tags: [memory, consolidation, dreaming, introspection, maintenance, holographic]
triggers: ["dream", "consolidate memory", "run dreaming", "memory consolidation"]
---

# Agent Dreaming (Memory-Agnostic)

Three-phase background memory consolidation that auto-detects the active memory
backend (built-in MEMORY.md or Holographic) and routes Phase 2 promotions to the
correct tools. Condensation (formerly `memory-lean-check`) is built into the
dreaming lifecycle as Phase 2.5.

**When to run:** Scheduled cron (recommended: every 6–8 hours) or manually after
a burst of activity.

**Autonomous vs interactive:** Light and Deep phases run autonomously in both
cron and manual modes. REM phase sends a message to the user with proposed
structural actions and waits for response — it never creates wiki pages or skills
without approval.

---

## Phase 0: Backend Detection (NEW — run first)

Before any dreaming work, determine which memory backend is active. This
determines which tools you use in Phase 2.

### Steps

1. **Read the active memory provider** from `$HERMES_HOME/config.yaml`:

   ```bash
   grep -A1 "^memory:" $HERMES_HOME/config.yaml | grep "provider:" | awk '{print $2}'
   ```

   Or read the config and check `memory.provider`.

2. **Set backend mode:**

   | Config value | Backend | Tools to use in Phase 2 | Threshold check |
   |---|---|---|---|
   | `""` (empty) or `null` or absent | **built-in** | `memory(action='add'|'replace'|'remove')` | Char limit (60%/80% of 2,200) |
   | `holographic` | **holographic** | `fact_store(action='add'|'update'|'remove')` + `fact_feedback(action='helpful'|'unhelpful')` | Trust score (min 0.3 default) |
   | `honcho` | **honcho** | Not yet supported — fall back to built-in `memory()` | Built-in limits apply |
   | `mem0` | **mem0** | Not yet supported — fall back to built-in `memory()` | Built-in limits apply |
   | Other | **unknown** | Fall back to built-in `memory()` | Built-in limits apply |

3. **Log the backend** in the dream artifact: `## Backend: <mode>`

4. **For holographic users:** Also note the config:
   ```bash
   grep -A6 "hermes-memory-store:" $HERMES_HOME/config.yaml
   ```
   Record `default_trust` and `min_trust_threshold` — these inform Phase 2 scoring.

   **If grep returns nothing:** The `plugins.hermes-memory-store` section is
   missing from config.yaml. This means holographic tools (`fact_store`, `probe`,
   `reason`, `fact_feedback`) are **not registered** and will not appear in any
   toolset. Log this as: `holographic configured but plugin not wired — fallback
   to built-in`. Route all Phase 2 promotions to the built-in `memory()` tool.
   This is the single most common blocker across dreaming cycles — flag it in the
   REM message so the user can wire the plugin.

5. **Verify tool availability before Phase 2.** After determining the backend,
   confirm the required tools are actually available in your toolset by scanning
   your tool list. If `fact_store` is absent despite holographic being configured,
   fall back to built-in `memory()` and note it in the dream artifact's backend
   section: `Holographic configured but tools unavailable — fallback to built-in`.
   The Phase 2 routing table still applies, but the actual tool calls degrade to
   built-in when holographic tools are missing.

---

## Phase 1: Light (Ingest + Stage)

**Goal:** Pull raw material from recent sessions, identify candidates for promotion.

### Steps

1. **Get recent sessions.** Call `session_search()` with no arguments. This
   returns recent session titles, previews, and timestamps. Note the count — if
   zero sessions, skip dreaming entirely and report "nothing to dream about."

2. **Resolve wiki path.** Read `wiki.path` from `$HERMES_HOME/config.yaml`
   (e.g. `grep wiki.path $HERMES_HOME/config.yaml`). Fall back to
   `$HERMES_HOME/wiki` if not configured. Never use `$HOME/wiki` — that's shared
   across profiles and breaks isolation. This path is needed for checking
   existing wiki pages when evaluating pointer candidates.

3. **Read current memory.** This step is **backend-specific**:

   **Built-in:** Read `$HERMES_HOME/memories/MEMORY.md`. Count entries (split on
   `§`, count non-empty segments). Note char usage. This is your baseline.

   **Holographic:** Call `fact_store(action='list', limit=50)` and
   `fact_store(action='search', query='')` to get a broad view of what's stored.
   Count entries and note trust score distribution. Holographic has no char
   limit, so "baseline" means entry count and average trust.

4. **Filter sessions.** Skip cron sessions (`session.source == "cron"`) — they're
   fully automated with no user interaction, unless they logged errors.

5. **Deep-dive sessions with signal.** For each remaining session that looks
   substantive, call `session_search(query="<session topic keywords>")` to get a
   full summary. **Important:** FTS5 queries match message *content*, not session
   titles — if `session_search(query="<exact session title>")` returns no results,
   try broader keyword queries using terms likely to appear in the messages
   (e.g. `"hello OR test OR bot"` rather than `"Testing the bot connection"`).
   Also try `session_search(query="...")` without a `session_id` constraint —
   discovery mode searches across all sessions and is more forgiving. Look for:
   - **User corrections** — "actually it's X not Y", "don't do that", "remember this"
   - **New preferences** — tool choices, style, workflow habits
   - **Environment discoveries** — ports, paths, service names, config quirks
   - **Recurring problems** — things that broke, workarounds found
   - **Resolved ambiguities** — cases where you had to ask clarifying questions
     and got definitive answers

6. **Stage candidates.** Write a dream artifact to
   `$HERMES_HOME/dreams/YYYY-MM-DD-HHMM.md` with this structure:

   ```markdown
   # Dream Artifact — YYYY-MM-DD HH:MM

   ## Backend
   - Provider: [built-in | holographic]
   - [For built-in: entry count, char usage / 2,200 (% full)]
   - [For holographic: entry count, avg trust score, min_trust_threshold]

   ## Sessions Reviewed
   - [session_id] title (timestamp)
   - ...

   ## Candidates
   ### New Entries (genuinely new info not in current memory)
   - [candidate 1]: description + source session
   - ...

   ### Replacements (corrections to existing entries)
   - [existing entry text] → [proposed replacement] + source session
   - ...

   ### Removals (stale/broken info)
   - [entry text] — reason (contradicted by session X, expired, etc.)

   ## Patterns Noted
   - [any recurring themes across sessions]
   ```

   **Do NOT write to memory yet.** This phase only stages candidates.

---

## Phase 2: Deep (Score + Promote)

**Goal:** Evaluate staged candidates against current memory, promote winners.

**This phase routes promotions based on the backend detected in Phase 0.**

### Steps

1. **Load the dream artifact** created in Phase 1. If it has no candidates, skip
   to Phase 3.

2. **Score each candidate.** For each candidate, assess four dimensions. A
   candidate must pass ALL four to be promoted — any single failure means skip:

   - **Novelty:** Is this genuinely new, or does it overlap with an existing
     entry? (Check current memory for similar entries — for holographic, use
     `fact_store(action='search', query='<keywords>')` or
     `fact_store(action='probe', entity='<entity>')`) → FAIL if a similar entry
     already exists.
   - **Durability:** Will this still be true in 30 days? User preferences and
     environment facts score high. Task progress, TODOs, and session outcomes
     score low. → FAIL if the fact is temporary or likely to change.
   - **Specificity:** Is this precise enough to be actionable? "User prefers X"
     is good. "User might like X" is vague. → FAIL if the entry would require
     guessing to act on.
   - **Reduction:** Does promoting this let you *remove or shorten* an existing
     entry? → Not a hard fail, but candidates with reduction potential get priority.

   **For replacements:** Only assess Novelty (is the new version genuinely
   better?) and Durability (is the replacement still accurate?). Specificity
   and Reduction don't apply.

3. **Execute promotions — BACKEND-ROUTED:**

   #### Built-in Backend

   Use the `memory` tool as normal:
   - **New entries:** `memory(action='add', target='memory', content='...')` —
     use wiki pointers where a wiki page exists or should exist. Keep entries
     under 150 chars ideally.
   - **Replacements:** `memory(action='replace', target='memory',
     old_text='...', content='...')` — old_text must be a unique substring.
   - **Removals:** `memory(action='remove', target='memory',
     old_text='...')` — only if genuinely stale/contradicted.

   #### Holographic Backend

   Use `fact_store` and `fact_feedback`:
   - **New entries:** `fact_store(action='add', content='...',
     category='<user_pref|project|tool|general>', tags='...',
     entities=['entity1', 'entity2'])` — Provide entities for compositional
     recall. Categories: `user_pref` for preferences/habits, `project` for
     workspace/codebase facts, `tool` for tool quirks/workarounds, `general` for
     everything else.
   - **Replacements:** `fact_store(action='update', fact_id=<id>,
     content='updated text')` — Get the fact_id from Phase 1's `list` output.
   - **Removals:** `fact_store(action='remove', fact_id=<id>)` — only if the
     fact is genuinely stale or contradicted.

   **Entities are critical for holographic.** Unlike MEMORY.md (flat text),
   holographic uses HRR vectors bound to entities for compositional queries.
   Always provide entities when adding facts:
   - User preferences → entity: "user"
   - Project facts → entity: project name (e.g., "openroom", "ghostfolio")
   - Tool facts → entity: tool name (e.g., "docker", "pytest")
   - General/environment → entity: "environment"

4. **Log promotions to dream diary.** Append to
   `$HERMES_HOME/dreams/diary.md`:

   ```markdown
   ### YYYY-MM-DD HH:MM — Deep Phase (backend: <mode>)
   - PROMOTED: [entry text] (from: [session summary])
   - REPLACED: [old] → [new] (reason: [correction source])
   - SKIPPED: [candidate] (reason: [novelty/durability/specificity failure])
   - REMOVED: [entry] (reason: [stale/contradicted])
   ```

5. **Post-promotion check — BACKEND-SPECIFIC:**

   #### Built-in Backend

   Re-read `$HERMES_HOME/memories/MEMORY.md`. Check char usage:
   - Under 60%: healthy, no action needed
   - 60–80%: flag in diary, allow replacements but defer new additions next cycle
   - Over 80%: flag in diary as critical; run Phase 2.5 Condensation immediately
     (see below) before promoting any new entries

   #### Holographic Backend

   Call `fact_store(action='list', limit=50)` to verify new entries were stored.
   Check trust scores:
   - Entries below `min_trust_threshold` (default 0.3): flag in diary — these
     will be filtered from retrieval
   - New entries at `default_trust` (default 0.5): healthy
   - Entries that were contradicted or corrected: call
     `fact_feedback(action='unhelpful', fact_id=<id>)` to decay them

---

## Phase 2.5: Condensation (Memory Hygiene)

**Goal:** Trim bloat and decay stale entries without losing signal. Runs after Phase 2
promotions when capacity thresholds are breached, or as part of every dreaming cycle for
holographic backends.

**This replaces the deprecated `memory-lean-check` skill.** Condensation is now built
into the dreaming lifecycle.

### When to run

| Backend | Trigger | Action |
|---------|---------|--------|
| **Built-in** | MEMORY.md > 60% char usage | Run condensation before next promotion cycle |
| **Built-in** | MEMORY.md > 80% char usage | Run condensation immediately — defer all promotions |
| **Holographic** | Every dreaming cycle | Light condensation pass (trust decay review) |
| **Holographic** | 5+ facts below `min_trust_threshold` | Deep condensation (prune or update) |

### Steps (Built-in Backend)

1. **Read MEMORY.md.** Count entries and calculate char usage.

2. **Identify bloat candidates:**
   - Wiki pointers that are already covered by more recent entries
   - Entries that duplicate info (merge them)
   - Verbose entries that can be shortened to wiki pointers
   - Stale entries referencing deprecated tools/projects/workflows

3. **Execute condensation:**
   - **Merge duplicates:** Combine overlapping entries into one. Use `memory(action='replace')`
     for the merged version, then `memory(action='remove')` for the absorbed one.
   - **Shorten to pointers:** Replace verbose entries with wiki pointers where the wiki
     page already exists. Example: `"OpenWork: see wiki/infrastructure/openwork.md"`.
   - **Remove stale entries:** Only if the tool/project is confirmed dead or the info
     is contradicted by a more recent session.
   - **Rephrase for density:** Tighten wording without losing meaning.

4. **Verify:** Re-read MEMORY.md and confirm char usage dropped. Log changes to dream
   diary with `CONDENSED:` prefix.

### Steps (Holographic Backend)

1. **List all facts:** `fact_store(action='list', limit=100)`.

2. **Review trust scores:**
   - Facts below `min_trust_threshold` (default 0.3): mark for pruning
   - Facts at `default_trust` (0.5) with 0 retrieval count: candidates for decay
   - Facts with `retrieval_count > 0` but `helpful_count == 0`: ambiguous — flag for review

3. **Run contradiction check:** `fact_store(action='contradict')` — find pairs of facts
   that make conflicting claims. For each pair, use `session_search` to determine which
   is correct, then decay the wrong one with `fact_feedback(action='unhelpful')`.

4. **Execute decay/pruning:**
   - **Decay stale facts:** `fact_feedback(action='unhelpful', fact_id=<id>)` on facts
     that are outdated or never retrieved.
   - **Prune dead facts:** `fact_store(action='remove', fact_id=<id>)` only for facts
     that are genuinely wrong or contradicted.
   - **Update aging facts:** `fact_store(action='update', fact_id=<id>, content='...')`
     for facts that are still relevant but need refreshing.

5. **Log to dream diary:**
   ```
   ### YYYY-MM-DD HH:MM — Condensation (backend: holographic)
   - DECAYED: [fact_id] [content snippet] (reason: never retrieved / outdated)
   - REMOVED: [fact_id] [content snippet] (reason: contradicted by fact N)
   - UPDATED: [fact_id] old → new
   - HEALTHY: N facts at trust ≥0.5, avg retrieval: M
   ```

### Rules

- **Condensation must not increase char/trust pressure.** Every action should reduce
  the footprint or improve signal quality.
- **Never remove facts without evidence.** For built-in, verify staleness against
  recent sessions. For holographic, use `contradict` + `session_search` before
  `fact_feedback(action='unhelpful')`.
- **Wiki pointers are preferred over inline detail.** If a wiki page exists or should
  exist, point to it rather than embedding the detail in memory.
- **Condensation log entries go in the dream diary**, not a separate artifact.

---

## Phase 3: REM (Pattern Extract)

**Goal:** Look across recent dream artifacts for recurring themes that warrant
structural action.

### Steps

1. **Read recent dream artifacts.** List files in `$HERMES_HOME/dreams/`
   (excluding `diary.md`). Read the last 3–5 artifacts. Look for:
   - **Repeated corrections** — Same thing corrected 2+ times → candidate for a
     wiki entry or skill update
   - **Repeated patterns** — Same problem solved 2+ times → candidate for a new
     skill
   - **Memory gaps** — Topics that came up frequently but have no memory entry →
     candidate for promotion in next cycle
   - **Skill staleness** — Skills referenced in sessions but with outdated info →
     candidate for `skill_manage(action='patch')`

2. **For patterns that warrant action, report — don't act.** REM patterns
   involve structural changes (creating wiki pages, skills, patching skills) that
   benefit from human review. Instead of executing directly:

   - **Log to dream artifact and diary** as usual (traceability)
   - **Send a message** to the user's home channel summarizing the patterns and
     proposed actions. Format:

   ```
   🌙 Dreaming REM (backend: <mode>) — patterns found:

   • [PATTERN]: [description]
     → Proposed: [create wiki page / create skill / patch skill / queue for next cycle]

   • [PATTERN]: ...
     → Proposed: ...

   Reply with which to act on, or "skip all" to defer.
   ```

   - **Wait for user response** before executing any structural changes
   - If running as a cron job with no interactive session, send the message and
     stop — the user can trigger the actions manually or reply in chat

3. **Write REM summary** to the dream artifact (append):

   ```markdown
   ## REM Phase — Patterns
   - PATTERN: [description] → ACTION: [create wiki page / create skill / patch skill / queue for next cycle]
   - ...
   ```

4. **Update dream diary** with REM findings.

---

## Rules

- **Never fabricate session content.** If `session_search` doesn't return useful
  detail for a session, skip it. Do not infer or hallucinate what happened.
- **Never promote speculative entries.** Every promoted memory must trace to a
  specific session interaction. "I think the user might prefer X" is not
  promotion-worthy.
- **Pointer = entry.** If an entry has a wiki/skill pointer (`see wiki/...` or
  `see skill '...'`), the pointer IS the entry. Inline detail that duplicates
  what the pointer targets should be removed. Details belong at the destination,
  not in memory.
- **For holographic: entities are mandatory on add.** Never call
  `fact_store(action='add')` without `entities`. Without entities, the fact
  cannot be recalled via `probe` or `reason` — it's effectively orphaned.
- **Preserve the § delimiter (built-in only).** When reading/writing MEMORY.md
  directly, never corrupt the entry separator.
- **Keep dreams compact.** Dream artifacts should be under 200 lines. If a
  session generated 20+ candidates, pick the top 5 by durability score.
- **Respect backend limits:**
  - Built-in: Over 60% char usage → defer *new additions*, allow *replacements
    that reduce char count*. Over 80% → defer everything, run Phase 2.5 Condensation first.
  - Holographic: No char limit, but flag entries below `min_trust_threshold`.
    Contradicted entries should get `fact_feedback(action='unhelpful')`.
- **Dream diary is append-only.** Never overwrite previous diary entries. New
  entries go at the bottom (chronological order).
- **Session IDs in diary.** Always note which session(s) informed each promotion
  for traceability.

---

## Verification

After a dreaming run:
1. Backend was correctly detected and logged
2. Phase 2 promotions used the correct backend tools
3. Post-promotion check passed (char limit for built-in, trust scores for
   holographic)
4. Dream artifact exists with all three phases documented
5. Dream diary has a new entry
6. No entries were promoted without a source session reference
7. For holographic: all new entries have entities

---

## Compatibility Matrix

| Feature | Built-in (MEMORY.md) | Holographic |
|---|---|---|
| Phase 0 detection | ✅ | ✅ |
| Phase 1 session review | ✅ | ✅ |
| Phase 2: add | `memory(action='add', target='memory')` | `fact_store(action='add', content='...', entities=[...])` |
| Phase 2: replace | `memory(action='replace', old_text='...')` | `fact_store(action='update', fact_id=N)` |
| Phase 2: remove | `memory(action='remove', old_text='...')` | `fact_store(action='remove', fact_id=N)` |
| Phase 2: list/verify | Read MEMORY.md | `fact_store(action='list')` |
| Capacity check | Char count / 2,200 | Trust score distribution |
| Bloat remediation | Phase 2.5 Condensation (built into dreaming) | `fact_feedback(action='unhelpful')` + trust decay |
| Phase 3 REM | ✅ | ✅ |
| Honcho fallback | ✅ (falls back to built-in) | N/A |
| Mem0 fallback | ✅ (falls back to built-in) | N/A |

---

## Pitfalls

- **Detection failure is silent.** If `memory.provider` is set to a value this
  skill doesn't recognize, it falls back to built-in. That's intentional — better
  to write to MEMORY.md than to nothing. But if the user expected holographic
  behavior and didn't get it, they won't know unless you report the backend in
  the REM message.
- **Holographic entities are not optional.** Unlike built-in memory where you
  can write any string, holographic facts without entities are invisible to
  `probe` and `reason`. Always provide entities.
- **Don't assume trust scores replace char limits.** Holographic has no capacity
  limit, but low-trust facts still pollute retrieval. Use `fact_feedback` to
  decay bad entries rather than hoarding indefinitely.
- **Wiki pointer format differs per backend:**
  - Built-in: `see wiki/path/page.md` inline in the entry text
  - Holographic: still include the pointer in `content`, but also add the wiki
    page as an entity for compositional queries
- **Dream diary compatibility.** Both backends share the same diary format and
  dream artifact directory. Entries are tagged with `(backend: <mode>)` for
  traceability.
- **Dream diary patching requires extra context.** Diary entries accumulate
  repetitive closing phrases (e.g. `→ No structural action needed.`, `→ Proposed:`)
  that appear in every cycle. When using `patch` to append, include the full
  pattern name and preceding lines in `old_string` to guarantee a unique match.
  A single-sentence `old_string` like `→ No structural action needed.` will hit
  multiple matches and fail — anchor with the pattern's numbered header.
