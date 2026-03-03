# Snapshot Schema Planning & Design Notes

**Phase:** Intentional Design (pre-implementation)  
**Date:** 2026-02-28  
**Status:** Work in Progress — Locking down structure before code

---

## Architecture: Snapshot as Re-ranker Output

This section captures the core design thinking behind Snapshot.json — the evolved conclusions from early planning.

### The 7–9 Field Model (Tight, Predictable)

Original concept: Think of snapshot as exactly **7–9 fields**. Small, consistent, predictable.

**The Snapshot Payload:**
- `objective_now` — 1 sentence (what we're doing right now)
- `active_threads[]` — up to 7 items (title, next_step, confidence, status)
- `recent_facts[]` — 10–20 atomic bullets (timestamped, source-linked)
- `hard_constraints[]` — verbatim "DO NOT" / boundaries (never summarized)
- `decisions[]` — key choices made + why (short)
- `open_questions[]` — what's unclear / needs user input
- `tool_state` — what tools are available/healthy
- `last_actions[]` — last 3–5 actions (for continuity)
- `handshake` — created_at, expires_at, source_event_ids

**Why this works:** This is enough to "wake up" an agent and keep heading the same direction without drift.

**Note:** The actual Snapshot.json evolved to include additional metadata fields for traceability (source, ts, etc.), but the core 9-field model remains the structure.

---

### The Sprawl Killer: Atomic Facts + Verbatim Constraints

**Problem:** "Summary-of-summary drift" and "JPEG artifacts" — when snapshots become pure prose summaries, models hallucinate and lose detail.

**Solution:** Three strict rules:

1. **Store atomic facts** — small, timestamped, source-linked, verifiable
2. **Carry constraints verbatim** — never compress or paraphrase "DO NOT" rules
3. **Allow one narrative line** — only `objective_now` for flow

**Why it matters:** This structure prevents hallucinated "memory smoothing" and keeps the system auditable. A constraint like "Never touch production DB without approval" stays exactly that — it's not reworded to "Avoid production" which an LLM might interpret differently later.

---

### How the Re-ranker Assembles a Snapshot

**Inputs to the re-ranker:**
- Last X minutes/hours of Tier A events (high-signal activity)
- Current task plan / thread list
- Tool results + errors
- Last snapshot (as prior context)

**Selection & Scoring Steps:**

1. **Extract candidates** — pull facts, constraints, and thread signals from events
2. **Score each by:**
   - Recency (newer beats older)
   - Task relevance (does this relate to current objective?)
   - Risk boost (constraints and errors get priority)
   - Progress signal (moving threads ranked higher than stalled ones)
3. **Assemble** until token budget is hit (~500 tokens)
4. **Validate:**
   - All hard constraints present? ✓
   - Objective clearly stated? ✓
   - At least one next step defined? ✓
   - No "DO NOT" rules missing? ✓

**Connection to confidence gradient:** Your earlier "softmax confidence gradient" insight fits perfectly here:
- Confidence rises when evidence is consistent
- Confidence drops when work is stalled or signals conflict
- This drives escalation policy (balanced vs. aggressive execution) *without changing model capabilities*

---

### Storage + Lifecycle (Bounded, Not Sprawling)

**Storage layer:**
- Keep snapshots in SQLite (`snapshots` table)
- Maintain 1–10 snapshots per thread
- Each snapshot tracks: `created_at`, `expires_at`, `event_range` (from_event_id → to_event_id)

**Purge policy:**
- Always keep last N snapshots
- Expire older snapshots by age (e.g., 7–14 days)

**Markdown flow (critical separation):**
- SQLite = raw snapshots (high fidelity, bounded)
- Markdown = *only* curated roll-ups (daily/weekly summaries of decisions + stable knowledge)
- **Never write snapshots directly to Markdown** (prevents sprawl, keeps vault clean)

---

### Token Budget Reality: 300–500 Tokens

**Myth:** Snapshots need to be prose summaries to fit token budgets.

**Reality:** Structured snapshots compress better than prose *and* drift less.

**Example allocation (500 tokens):**
- `objective_now` + `status` — ~20 tokens
- `active_threads` (5–7 rows) — ~100 tokens
- `hard_constraints` (5–10 lines) — ~80 tokens
- `recent_facts` (10–15 bullets) — ~150 tokens
- `decisions` (2–3 key choices) — ~100 tokens
- `open_questions` + `tool_state` — ~50 tokens

**Total:** ~500 tokens, fully structured, zero prose hallucination.

**Comparison of approaches:**

| Aspect | Prose Summary | Structured Snapshot |
|--------|---------------|-------------------|
| Easy to generate? | Yes | Needs assembly logic |
| Prone to drift? | High (models re-interpret) | Low (fields are atomic) |
| Constraint preservation? | Poor (gets paraphrased) | Excellent (verbatim) |
| Auditable? | Hard (implicit reasoning) | Easy (explicit fields) |
| Token efficiency | Fair | Excellent |

**Conclusion:** Structured snapshots are the right choice for Cognitive Controller.

---

### 1. Decisions — 4W+H + Revisit Conditions

**Schema:**
```json
{
  "decision_id": "dec_001",
  "what": "Short description of what was decided.",
  "why": "Rationale, constraints, tradeoffs.",
  "how": "Implementation approach / specific technical steps.",
  "when": "v1, after telemetry, cron marker, or date — when this applies.",
  "who": "system|user|operator — who made/owns this.",
  "inactive": false,
  "revisit_if": [
    "Condition 1 for re-evaluation — metric > threshold",
    "Condition 2 — observable event",
    "Condition 3 — time-based trigger"
  ],
  "source": { "type": "user|system|tool", "id": "evt_222" },
  "ts": "2026-02-28T00:00:00Z"
}
```

**Why this structure:**
- Prevents re-interpretation and drift (models can't re-argue a locked decision)
- `revisit_if` is the crisp trigger for re-evaluation (not vibes, not prose)
- `inactive` flag allows intentional pauses without deletion
- `when` captures phase, date, or cron-based applicability
- Full traceability via source + timestamp

**Key principle:** *Decisions are not summaries; they are atomic, immutable instructions with explicit conditions for change.*

---

### 2. Active Threads — Visibility into Stalled Work

**Added field:**
```json
"blocked_by": ["external_dependency|missing_credential|user_input|rate_limit|unknown"]
```

**Why:** Stalled threads now have a diagnosis. Instead of guessing why a thread is stuck, the system knows *what's holding it up*. Enables automation (e.g., escalate if blocked by `user_input` for > 2 hours).

---

### 3. Open Questions — Critical + Actionable

**Schema:**
```json
{
  "question_id": "q_001",
  "text": "What's missing / needs user input?",
  "priority": "high|medium|low",
  "critical": false,
  "blocker": false,
  "source": { "type": "user|system|tool", "id": "evt_123" },
  "ts": "2026-02-28T00:00:00Z"
}
```

**Fields:**
- `priority` — Relative importance (context for humans/models)
- `critical` — Escalation trigger (alert/notify *now*)
- `blocker` — Blocks a decision or thread (unblock immediately)

**Why:** Automation can act on boolean signals without re-reading prose. Clean signal-to-noise.

---

## Open Questions & Governance Gaps ❓

### Q1: When Format Clarity

Current spec says `when` can be:
- A date (ISO 8601?)
- A phase string (e.g., "after telemetry", "v1 default")
- A cron marker

**Need to define:**
- Exact date format (e.g., `2026-02-28T00:00:00Z` or `2026-02-28` or Unix timestamp?)
- List of standard phase strings (e.g., "v1", "after_telemetry_week_1", "post_qa")?
- Cron format (standard `0 0 * * *`?) or something simpler?

**Options:**
1. Documented enum of known phases (strictest)
2. Free string with parsing hints in code (most flexible, but risky for drift)
3. Hybrid: define core phases, allow free strings with validation warning

---

### Q2: Decision Archival & Snapshot Bloat

**Rule stated:** Max 3–7 decisions per snapshot (most recent + most load-bearing).

**Unclear:**
- When do we move a decision to an ADR (Architecture Decision Record) file?
- What triggers archival? (Time? Completion? Manual marking?)
- Should snapshot include a `archived_decision_refs` field linking to ADRs?
- Should archived decisions be fully removed or marked `inactive: true` first?

**Proposed workflow:**
```
Active Decision (in snapshot) 
  → Revisit condition met OR time decay
  → Mark inactive: true (grace period visible)
  → Move to docs/adr/ADR-NNN.md after 1 week
  → Remove from snapshot, add lightweight ref
```

---

### Q3: Governance Metadata

Should Snapshot.json include explicit limits in its schema, or is this just implied?

**Option A — Add to `meta`:**
```json
"governance": {
  "max_decisions": 7,
  "decision_when_format": "iso8601 | phase_string | cron",
  "max_threads": 20,
  "snapshot_token_budget": 500
}
```

**Option B — Keep in separate governance doc** (e.g., `docs/snapshot-governance.md`)

**Recommendation:** Option B (separate doc) keeps Snapshot.json clean, but add a `_governance_ref` in meta pointing to it.

---

### Q4: Hard Constraints — Verbatim Carry-Forward

From the Note.md audit: Resume blocks hallucinate. Need hard constraints that *never* get summarized.

**Current schema has `hard_constraints`:**
```json
{
  "text": "VERBATIM STRING - never summarize",
  "source": { "type": "user|system|tool", "id": "evt_001" }
}
```

**Question:** Should hard constraints also have a `revisit_if` or expiry logic? (E.g., "Never touch production DB" is always true; "Don't email until legal review" expires when reviewed.)

---

### Q5: Facts Metadata

**Current schema:**
```json
{
  "fact_id": "fct_001",
  "text": "Atomic fact sentence.",
  "type": "event|decision|state|result|warning",
  "confidence": 1.0,
  "source": { "type": "tool|user|system|db|inference", "id": "evt_123" },
  "ts": "2026-02-28T00:00:00Z",
  "tags": ["personal", "professional", "system", "auth", "memory"]
}
```

**Question:** Should facts also include:
- `ttl` (time-to-live before archival)?
- `invalidate_if` (like decisions have `revisit_if`)?
- `depends_on` (fact ID or decision ID this depends on)?

**Rationale:** Prevents stale facts from influencing decisions.

---

## Key Design Principles

1. **Structured over Prose**
   - Models can't re-interpret what they don't understand
   - Booleans and structured fields drive automation

2. **Crisp Triggers, Not Vibes**
   - `revisit_if`, `critical`, `blocker` are explicit automation hooks
   - No ambiguity about when to escalate or re-evaluate

3. **Bounded, Not Sprawling**
   - Max 3–7 decisions, ~500 token budget, 1–10 snapshots per thread
   - Overflow → ADR files or raw archives, not Snapshot bloat

4. **Traceability & Trust**
   - Every piece has `source` and `ts` (who said this + when)
   - Operator can audit the chain of reasoning

5. **Immutability with Grace**
   - Locked decisions can't be silently overwritten
   - Archival is explicit and reversible (at least during grace period)

---

## Implementation Questions (Critical for v1)

These are decisions that unlock the code.

### Q1: Active Threads Cap — 5 or 7?

Do you want `active_threads` capped at **5** (tighter focus) or **7** (more context) for v1?

- 5 = lean, forces prioritization, smaller token footprint
- 7 = more room for parallel work, matches your earlier "Top-K ≤ 8"

### Q2: recent_facts Scope — Verifiable Only or Include Inference?

Should `recent_facts` contain:
- **Option A:** Strictly verifiable events (tool outputs, user statements, timestamps)?
- **Option B:** Mix of verifiable + inferred context (e.g., "User implied they want X based on Y")?

Implications:
- Option A: Smaller, more auditable, less hallucination risk
- Option B: More contextual, but harder to trace if the inference was wrong

### Q3: Snapshot Expiry Window — 48h, 7d, or 14d?

When does a snapshot get archived/purged?
- 48 hours = keep only very recent, lean storage
- 7 days = one week of recovery points
- 14 days = two weeks, more redundancy

Depends on: session frequency, re-planning cycles, storage constraints.

---

## Notes on Snapshot as Re-ranker Payload

**Key insight:** Snapshot is not the raw log. It's the *assembled resume block* that gets injected into the next agent call.

**Data flow:**
1. Raw events → SQLite event store (append-only, high fidelity)
2. Re-ranker → selects & scores candidates from the event stream
3. Snapshot → structured, atomic, bounded payload (~500 tokens)
4. Markdown → only curated roll-ups (decisions + stable knowledge), not snapshots

**Prevention against drift:**
- Structured fields (no prose summarization)
- Atomic facts (verifiable, timestamped, immutable)
- Verbatim constraints (never compressed)
- Explicit `revisit_if` conditions (crisp automation hooks)

This prevents "summary-of-summary" hallucinations and keeps the system auditable.

---

- [ ] **Decide on `when` format** (enum vs. free string)
- [ ] **Define archival workflow** (when + how to move decisions to ADRs)
- [ ] **Governance doc** (max decisions, max threads, token budgets)
- [ ] **Facts expiry logic** (TTL, invalidate_if, dependencies)
- [ ] **Test with real example** (write a sample Snapshot.json with realistic data)
- [ ] **Pin down rollup/consolidation rules** (how decisions age, when facts consolidate)

---

## References

- **`Snapshot.json`** — Current schema (in repo root)
- **`docs/Note.md`** — Architectural audit & brutal assessment
- **`docs/architecture.md`** — High-level Control Tower design
- **ADR Pattern:** https://adr.github.io/ (for decision archival)

---

**Owner:** You (operator)  
**Last Updated:** 2026-02-28  
**Consensus Level:** High (core structure locked; governance details pending)
