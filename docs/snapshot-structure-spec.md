# Snapshot Structure Specification

**Phase:** Implementation Design  
**Status:** Core structure defined  
**Token Budget:** 300–500 tokens  
**Target Schema Version:** 1.0

---

## 1. The 7–9 Field Model

A snapshot should be **small, consistent, predictable**. Designed to give an agent everything it needs to "wake up" and continue in the same direction.

### Core Fields

| Field | Type | Size | Purpose |
|-------|------|------|---------|
| `objective_now` | string | ~20 tokens | What we're doing right now (1 sentence) |
| `active_threads[]` | array | ~100 tokens | Up to 7 concurrent work items |
| `recent_facts[]` | array | ~150 tokens | 10–20 atomic, timestamped facts |
| `hard_constraints[]` | array | ~80 tokens | Verbatim "DO NOT" rules (never paraphrased) |
| `decisions[]` | array | ~100 tokens | Key choices made + rationale |
| `open_questions[]` | array | ~50 tokens | What needs clarification |
| `tool_state` | object | ~30 tokens | Tool availability + health |
| `last_actions[]` | array | ~50 tokens | Last 3–5 actions (continuity) |
| `handshake` | object | ~20 tokens | Metadata: created_at, expires_at, source_event_ids |

**Total:** ~500 tokens (fits comfortably in 300–500 budget with room for expansion)

---

## 2. Active Threads Schema Detail

Each thread is minimal but sufficient:

```json
{
  "thread_id": "str",
  "title": "Brief description",
  "next_step": "Immediate action required",
  "confidence": 0.0–1.0,
  "status": "active|stalled|done",
  "blocked_by": ["external_dependency|missing_credential|user_input|rate_limit|unknown"]
}
```

**Rationale:**
- `confidence`: SoftMax gradient (0–1) — rises with consistent evidence, drops with stalled/conflicting signals
- `blocked_by`: Explicit diagnosis of why a thread is stalled (actionable for automation)
- Up to 7 threads forces prioritization; more than 7 and you're not tracking top-K

---

## 3. The Sprawl Killer: Atomic Facts + Verbatim Constraints

### Problem: Summary-of-Summary Drift

Other systems warn of "JPEG artifacts" — when you summarize a summary, LLMs hallucinate and smooth away important details. A constraint like "Never modify production DB without approval" becomes "Avoid production changes" becomes "Don't touch prod" becomes something the model misinterprets.

### Solution: Three Rules

**Rule 1: Atomic Facts**
- Small, timestamped, source-linked
- Verifiable (tool output, user statement, observable event)
- No interpretation or paraphrasing
- Example: `"Tool X returned error 429 at 14:32 UTC (rate limit)"`

**Rule 2: Verbatim Constraints**
- Copied *exactly* as stated
- Never paraphrased or "improved"
- Example: `"HARD: Do not escalate without explicit user approval — security boundary"`

**Rule 3: Single Narrative Line**
- Only `objective_now` provides prose flow
- Everything else is structured

### Why It Works

Stops hallucinated "memory smoothing." Facts and constraints remain unchanged, so when the agent wakes up, it has the actual truth, not a telephone-game distortion.

---

## 4. How the Re-ranker Assembles a Snapshot

### Inputs

1. Last X minutes/hours of **Tier A events** (high-signal activity)
2. Current task plan / thread list
3. Tool results + errors
4. Last snapshot (as prior context)

### Assembly Steps

#### Step 1: Extract Candidates
Pull facts, constraints, and thread signals from the event stream.

#### Step 2: Score Each Candidate

Score by (in priority order):
- **Risk** — constraints and errors get boosted (safety first)
- **Recency** — newer events beat older ones
- **Task relevance** — does this relate to the current objective?
- **Progress signal** — moving/active threads ranked higher than stalled ones

#### Step 3: Assemble Until Token Budget

Pack fields in order of importance until you hit ~500 tokens:
1. Handshake (metadata, ~20 tokens)
2. Objective (1 line, ~20 tokens)
3. Constraints (5–10 lines, highest score, ~80 tokens)
4. Active threads (5–7, ~100 tokens)
5. Recent facts (10–15, ~150 tokens)
6. Decisions (2–3 key, ~100 tokens)
7. Questions + tool_state + actions (remaining budget, ~50 tokens)

#### Step 4: Validate

Checklist before returning snapshot:
- [ ] All hard constraints present?
- [ ] Objective clearly stated?
- [ ] At least 1 next step defined?
- [ ] No critical "DO NOT" rules missing?

### Confidence Gradient Integration

Your **softmax confidence gradient** drives the re-ranker's escalation policy:

- **Confidence ↑** — consistent evidence, moving threads, progress signals
- **Confidence ↓** — stalled work, conflicting signals, unresolved blockers
- **Effect:** Drives escalation decision (balanced execution vs. push-hard), *without changing model capabilities*

Example:
```
Thread A: confidence 0.95 (moving, clear path) → let it run
Thread B: confidence 0.4 (stalled, blocked by user input) → escalate or pause
```

---

## 5. Storage + Lifecycle

### SQLite Store

**Table: `snapshots`**

```
- snapshot_id (UUID, PK)
- thread_id (FK)
- created_at (timestamp)
- expires_at (timestamp)
- event_range_from (event_id)
- event_range_to (event_id)
- snapshot_data (JSON blob)
```

### Retention Policy

**Keep:**
- Last N snapshots (always retained)
- Snapshots younger than 7–14 days

**Purge:**
- Older snapshots beyond age threshold
- Maintain 1–10 snapshots per thread (bounded)

### Markdown Flow (Critical Separation)

**SQLite snapshots:**
- Raw, high-fidelity recovery points
- Bounded (1–10 per thread)
- Never summarized

**Markdown vault:**
- Only curated **roll-ups** (daily/weekly summaries)
- Decisions + stable knowledge
- Decisions + lessons learned → ADR files
- Never raw snapshots

**Why the separation:**
- Snapshots are operational (for re-waking the agent)
- Roll-ups are for human review + long-term learning
- Keeps vault clean, snapshots focused

---

## 6. Token Budget Reality

### The Math

Your 300–500 token budget is **achievable and safe** with structured fields:

**Sample allocation (500 tokens):**
- Objective + status: ~20 tokens
- Threads (5–7 rows): ~100 tokens
- Constraints (5–10 lines): ~80 tokens
- Recent facts (10–15 bullets): ~150 tokens
- Decisions (2–3 key): ~100 tokens
- Questions + tool_state + actions: ~50 tokens

**Total:** ~500 tokens, fully structured, zero prose.

### Why Structured Beats Prose

| Aspect | Prose Summary | Structured Snapshot |
|--------|---------------|-------------------|
| **Generation** | Easy | Needs assembly logic |
| **Drift risk** | High (models re-interpret) | Low (atomic fields) |
| **Constraint preservation** | Poor (gets paraphrased) | Perfect (verbatim) |
| **Auditability** | Hard (implicit) | Easy (explicit) |
| **Token efficiency** | Fair (verbose) | Excellent (compact) |
| **Safety** | Risky (hallucination) | Safe (deterministic) |

---

## 7. Design Decision: Structured vs. Prose

### Option 1: Prose Summary Snapshot ❌

**Pros:**
- Simple to generate (just summarize the events)
- Reads naturally to humans

**Cons:**
- High drift risk (models re-interpret)
- Missing constraints (summarized away)
- "JPEG artifacts" — summary-of-summary degradation
- Hard to audit

### Option 2: Structured Snapshot (Atomic Fields) ✅

**Pros:**
- Stable across inference runs (no re-interpretation)
- Compact & efficient (fits token budget)
- Auditable (explicit source/reasoning)
- Safe (constraints preserved verbatim)
- Drives automation (crisp boolean signals + revisit_if)

**Cons:**
- Assembly logic required (reranker complexity)
- Slightly more schema definition up front

**Recommendation:** **Option 2 — Structured Snapshot**

The assembling complexity is worth it. You prevent the "JPEG artifacts" problem entirely and create a system that doesn't hallucinate its own context.

---

## 8. Implementation Checkpoints

- [ ] **Q1** — Cap active_threads at **5 or 7**? (default: 7)
- [ ] **Q2** — Verify facts: **strictly verifiable or include inferred context**? (recommendation: stricty verifiable to prevent drift)
- [ ] **Q3** — Snapshot expiry: **48h, 7d, or 14d**? (depends on session frequency)
- [ ] **Reranker scoring weights** — tune the risk/recency/relevance/progress weights after Phase 0
- [ ] **Token budget tuning** — monitor real snapshots after first week; adjust allocations if needed

---

## Cross-References

- [Snapshot.json](../Snapshot.json) — Actual schema instance (locked)
- [snapshot-schema-planning.md](snapshot-schema-planning.md) — Design rationale & open questions
- [architecture.md](architecture.md) — High-level Control Tower design
- [storage-schema.md](storage-schema.md) — SQLite schema details (when created)

---

**Owner:** You  
**Last Updated:** 2026-02-28  
**Status:** Specification Ready for Implementation
