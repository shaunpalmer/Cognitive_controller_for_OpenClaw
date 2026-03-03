# Roll-Up Policy
Cognitive Controller – OpenClaw

## Purpose

The Roll-Up process consolidates episodic memory into structured, durable knowledge.

It does **not**:
- summarize summaries
- rewrite hard constraints
- compress narrative text
- overwrite raw logs

It does:
- extract stable decisions
- promote validated facts
- record architectural changes
- preserve source traceability
- reduce episodic sprawl

Roll-Up is curation, not compression.

---

# When Roll-Up Runs

Roll-Up may be triggered by:

- Scheduled task (e.g., every 30 minutes or 1 hour)
- Idle detection (no activity for X minutes)
- Snapshot expiry threshold reached
- Manual operator trigger

Roll-Up must never block the gateway.

---

# Inputs

Roll-Up reads from:

- SQLite episodic events (Tier A events only)
- Latest valid snapshot
- Decisions marked for promotion
- Hard constraints (verbatim)
- Thread completion events

Roll-Up does NOT use previous roll-ups as primary input.

Atomic facts are the source of truth.

---

# Output Location

Roll-Up writes to:

- `memory/YYYY-MM-DD.md`  (append-only daily log)
OR
- `memory/rollups/YYYY-MM-DD-HHMM.md`

Never overwrite previous roll-ups.

Roll-Ups are immutable once written.

---

# Roll-Up Structure (Markdown Template)

Each roll-up must follow this structure:

---

# Hard Rules

1. No prose storytelling.
2. No summary-of-summary compression.
3. All facts must trace back to event IDs.
4. Hard constraints must be verbatim.
5. Decisions must retain why + how + when.
6. Roll-Up files are append-only and immutable.

---

# What Roll-Up Is NOT

Roll-Up is NOT:

- A chat transcript
- A compressed memory file
- A replacement for SQLite
- A snapshot
- A planner

Roll-Up is archival consolidation.

---

# Relationship to Snapshot

Snapshot:
- Short-term operational memory
- Injected into LLM
- Overwritten

Roll-Up:
- Durable structured record
- Human-readable
- Never overwritten
- Used for future semantic retrieval

---

# Expiry Policy

Episodic SQLite data may be purged after:

- 48 hours (default)
- After confirmed Roll-Up write
- After raw dump fallback

Roll-Up files are permanent unless manually archived.

---

# Design Philosophy

Hot memory keeps continuity.

Episodic memory keeps recoverability.

Roll-Up keeps clarity.

Semantic memory keeps wisdom.

Never mix these responsibilities.