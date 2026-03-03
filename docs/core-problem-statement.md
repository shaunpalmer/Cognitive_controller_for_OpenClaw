# Core Problem Statement: Memory System Architecture

**Date:** 2026-03-03  
**Status:** Problem Definition  
**Priority:** Critical

---

## The Fundamental Problem

### OpenClaw's Current Memory Model

**What it does:**
- Stores memory as Markdown files
- LLM reads entire memory files into context
- Works well at small scale

**Why it breaks:**
```
Session Start: 2,000 tokens (context)
+ Skills loaded: 5,000 tokens
+ Memory files: 10,000 tokens
+ Conversation: 20,000 tokens
─────────────────────────────────
Total: 37,000 tokens

Context window: 128,000 tokens ✓ (still fits)

After 2 hours:
+ More skills: 8,000 tokens
+ Memory growth: 25,000 tokens
+ Conversation: 60,000 tokens
─────────────────────────────────
Total: 93,000 tokens

Context window: 128,000 tokens ⚠️ (70% full)

After 4 hours:
+ Even more skills: 12,000 tokens
+ Memory bloat: 45,000 tokens
+ Conversation: 90,000 tokens
─────────────────────────────────
Total: 147,000 tokens

Context window: 128,000 tokens ❌ (OVERFLOW)
```

**The cascade failure:**
1. Memory grows unbounded (Markdown accumulates)
2. Skills multiply (each adds to context)
3. Context window fills
4. Compaction kicks in (summarizes/trims)
5. Critical details lost (paraphrasing introduces drift)
6. Agent "forgets" constraints, decisions, active work
7. **Agent amnesia** - can't remember what it was doing

---

## Why Markdown Memory Fails at Scale

### Problem 1: Unbounded Growth

```
memory/
├── MEMORY.md              (5,000 tokens)
├── 2026-03-01.md         (3,000 tokens)
├── 2026-03-02.md         (4,000 tokens)
├── 2026-03-03.md         (6,000 tokens)
├── api-design-notes.md   (2,000 tokens)
├── database-schema.md    (3,000 tokens)
└── decisions.md          (2,000 tokens)
─────────────────────────────────────────
Total: 25,000 tokens (and growing)
```

**Every file loaded = context consumed**

### Problem 2: No Prioritization

All memory treated equally:
- Critical constraint from 2 hours ago
- Random note from 3 days ago
- Active thread with next step
- Completed task from last week

**LLM sees everything or nothing** (no middle ground)

### Problem 3: Skills Multiply the Problem

```
skills/
├── api-client.ts         (1,500 tokens)
├── database-query.ts     (2,000 tokens)
├── file-operations.ts    (1,200 tokens)
├── git-operations.ts     (1,800 tokens)
├── test-runner.ts        (1,500 tokens)
├── linter.ts             (1,000 tokens)
└── deployment.ts         (2,000 tokens)
─────────────────────────────────────────
Total: 11,000 tokens

With 20 skills: 30,000+ tokens
With 50 skills: 75,000+ tokens
```

**Skills + Memory + Conversation = Context overflow**

### Problem 4: Compaction Loses Information

When context fills, OpenClaw compacts:
```
Original constraint:
"Never use async/await in the database layer. 
Always use callbacks for connection pooling 
to avoid memory leaks in long-running processes."

After compaction:
"Use callbacks in database layer."

After 2nd compaction:
"Database uses callbacks."

After 3rd compaction:
[constraint removed - deemed low priority]
```

**Critical details lost through summarization**

---

## The Solution: State-Based Memory with Snapshots

### Core Concept: Bounded Working Memory

Instead of loading ALL memory, load a **bounded snapshot**:

```
┌─────────────────────────────────────────┐
│         Snapshot (500 tokens)           │
├─────────────────────────────────────────┤
│ Objective: "Implement user auth API"   │
│                                         │
│ Active Threads (3):                     │
│  - API endpoint design (next: review)  │
│  - Database schema (blocked: approval) │
│  - Test coverage (in progress)         │
│                                         │
│ Hard Constraints (5):                   │
│  - Never store passwords in plaintext  │
│  - Always use JWT for sessions         │
│  - Rate limit: 100 req/min per IP      │
│  - Log all auth failures               │
│  - 2FA required for admin accounts     │
│                                         │
│ Recent Facts (10):                      │
│  - User table has 'email' column       │
│  - JWT secret in .env file             │
│  - Redis available for rate limiting   │
│  ...                                    │
│                                         │
│ Last Actions (3):                       │
│  - Created /api/auth/login endpoint    │
│  - Added bcrypt for password hashing   │
│  - Wrote unit tests for login logic    │
└─────────────────────────────────────────┘

Total: 487 tokens (fits easily)
```

**Benefits:**
- ✅ Fixed size (300-500 tokens)
- ✅ Always fits in context
- ✅ Prioritized content (risk → recency → relevance)
- ✅ Constraints never paraphrased (verbatim)
- ✅ Clear next steps (no confusion)

### State Transitions: Clean In/Out

```
State A (Snapshot 1):
├─ Objective: Design API
├─ Threads: [API design, Schema review]
└─ Facts: [Endpoint requirements, Auth flow]

    ↓ Work happens (events captured)

State B (Snapshot 2):
├─ Objective: Implement API
├─ Threads: [Implementation, Testing]
└─ Facts: [Code structure, Test cases]

    ↓ Work happens (events captured)

State C (Snapshot 3):
├─ Objective: Deploy API
├─ Threads: [Deployment, Monitoring]
└─ Facts: [Deploy config, Health checks]
```

**Agent moves between states cleanly:**
- No context bloat
- No information loss
- No drift from compaction
- Clear continuity

---

## Database Flow Architecture

### Current Flow (Broken)

```
User Message
    ↓
OpenClaw Gateway
    ↓
Load ALL memory files (unbounded)
    ↓
Load ALL skills (unbounded)
    ↓
Add conversation history (unbounded)
    ↓
Send to LLM (context overflow!)
    ↓
Compaction (information loss!)
    ↓
Agent confused (amnesia!)
```

### Fixed Flow (Snapshot-Based)

```
User Message
    ↓
OpenClaw Gateway
    ↓
[Cognitive Controller intercepts]
    ↓
Load LATEST SNAPSHOT (500 tokens, bounded)
    ↓
Load RELEVANT skills only (filtered by thread context)
    ↓
Add conversation (recent turns only)
    ↓
Send to LLM (context under control)
    ↓
Capture events (tool results, decisions, facts)
    ↓
Update snapshot (rerank, trim to budget)
    ↓
Agent maintains continuity (no amnesia!)
```

### Data Flow Detail

```
┌─────────────────────────────────────────────────────────────┐
│                    Event Stream (Append-Only)                │
│  [msg] [tool] [decision] [fact] [msg] [tool] [fact] ...    │
└────────────────────┬────────────────────────────────────────┘
                     │
                     ▼
         ┌───────────────────────┐
         │   Event Observer      │
         │   (Capture & Tag)     │
         └───────────┬───────────┘
                     │
                     ▼
         ┌───────────────────────┐
         │   SQLite Store        │
         │   (Operational Memory)│
         └───────────┬───────────┘
                     │
        ┌────────────┼────────────┐
        │            │            │
        ▼            ▼            ▼
   ┌────────┐  ┌────────┐  ┌────────┐
   │Reranker│  │Snapshot│  │Roll-Up │
   │ (Score)│  │Assembly│  │ Engine │
   └────┬───┘  └────┬───┘  └────┬───┘
        │           │           │
        └───────────┼───────────┘
                    ▼
         ┌──────────────────────┐
         │  Bounded Snapshot    │
         │  (500 tokens)        │
         └──────────┬───────────┘
                    │
                    ▼
         ┌──────────────────────┐
         │  Inject into Context │
         │  (Before LLM call)   │
         └──────────────────────┘
```

---

## What We're Actually Solving

### For the System (OpenClaw)

**Problem:** Context window management is reactive (compaction after overflow)

**Solution:** Proactive bounded memory (snapshots prevent overflow)

**Benefit:** 
- Longer sessions without degradation
- No emergency compaction
- Predictable token usage

### For the Agent (LLM)

**Problem:** Memory bloat causes confusion and amnesia

**Solution:** Clean state transitions with bounded snapshots

**Benefit:**
- Always knows current objective
- Never loses constraints
- Clear next steps
- No drift from compaction

### For the User (Developer)

**Problem:** Agent forgets context, repeats work, loses decisions

**Solution:** Structured memory with audit trail

**Benefit:**
- Agent maintains continuity
- Decisions preserved
- Work doesn't get lost
- Transparent state (can inspect snapshots)

---

## Skills Integration (Future Problem)

### Current Skills Model

```
skills/
├── skill-1.ts  (loaded into context)
├── skill-2.ts  (loaded into context)
├── skill-3.ts  (loaded into context)
└── ...

With 50 skills: 50,000+ tokens in context
```

**Problem:** Every skill loaded = context consumed

### Snapshot-Aware Skills Model

```
Snapshot says:
  Objective: "Implement user authentication"
  Active threads: [API design, Database schema]

Cognitive Controller filters skills:
  ✓ Load: api-client.ts (relevant to API design)
  ✓ Load: database-query.ts (relevant to schema)
  ✓ Load: auth-helpers.ts (relevant to auth)
  ✗ Skip: deployment.ts (not relevant yet)
  ✗ Skip: monitoring.ts (not relevant yet)
  ✗ Skip: analytics.ts (not relevant yet)

Context saved: 30,000 tokens
```

**Solution:** Load only skills relevant to current snapshot

**Benefit:**
- Massive context savings
- Faster LLM inference
- More room for conversation

---

## The Database Problem

### What We're Fixing

**Current:**
- Markdown files (unstructured, unbounded)
- No queryable structure
- No prioritization
- No state management

**Fixed:**
- SQLite (structured, queryable)
- Event sourcing (append-only, audit trail)
- Reranking (priority-based)
- Snapshot state machine (bounded, versioned)

### Database Schema (Operational Memory)

```sql
-- Events: Append-only log
CREATE TABLE events (
  event_id INTEGER PRIMARY KEY AUTOINCREMENT,
  timestamp TEXT NOT NULL,
  event_type TEXT NOT NULL,
  content TEXT NOT NULL,
  metadata TEXT
);

-- Snapshots: Bounded state checkpoints
CREATE TABLE snapshots (
  snapshot_id TEXT PRIMARY KEY,
  thread_id TEXT,
  created_at TEXT NOT NULL,
  data TEXT NOT NULL,  -- JSON blob (500 tokens)
  token_count INTEGER
);

-- Threads: Work item tracking
CREATE TABLE threads (
  thread_id TEXT PRIMARY KEY,
  title TEXT NOT NULL,
  next_step TEXT,
  confidence REAL,
  status TEXT,  -- active, stalled, done
  blocked_by TEXT
);
```

**Query capabilities:**
```sql
-- Get latest snapshot
SELECT * FROM snapshots ORDER BY created_at DESC LIMIT 1;

-- Get active threads
SELECT * FROM threads WHERE status = 'active';

-- Get recent events
SELECT * FROM events WHERE timestamp > datetime('now', '-1 hour');

-- Get stalled threads
SELECT * FROM threads WHERE status = 'stalled';
```

**Markdown can't do this.**

---

## Success Criteria

### System Level

✅ Sessions run 10x longer without degradation  
✅ Context usage predictable and bounded  
✅ No emergency compaction needed  
✅ Skills loaded selectively (not all at once)

### Agent Level

✅ Agent maintains continuity across sessions  
✅ Constraints never lost or paraphrased  
✅ Clear next steps always available  
✅ No "what was I doing?" confusion

### User Level

✅ Transparent state (can inspect snapshots)  
✅ Audit trail (can replay events)  
✅ Rollback capability (restore previous snapshot)  
✅ Confidence signals (know when agent is stuck)

---

## Implementation Priority

### Phase 0: Foundation (Weeks 1-2)

**Goal:** Prove snapshot model works

1. Event capture (Observer)
2. SQLite store (Events, Snapshots, Threads)
3. Basic reranker (Risk → Recency)
4. Snapshot assembly (500 token budget)
5. Context injection (before LLM call)

**Success:** Agent uses snapshots instead of full memory

### Phase 1: Working Memory (Weeks 3-4)

**Goal:** Full snapshot lifecycle

1. Thread management (status, confidence, blockers)
2. Advanced reranker (Risk → Recency → Relevance → Progress)
3. Roll-up engine (consolidate to Markdown)
4. UI panel (view threads, snapshots, health)
5. Health monitoring (LED, confidence decay)

**Success:** Agent maintains state across long sessions

### Phase 2: Elastic Resolution (Weeks 5-6)

**Goal:** Intelligent failure handling

1. Confidence-based escalation
2. Tactic swapping on repeated failures
3. Model fallback integration
4. Automatic retry with different approaches

**Success:** Agent recovers from stuck states

---

## The Core Insight

**The problem isn't memory storage.**  
**The problem is memory LOADING.**

Markdown is fine for archival.  
But loading ALL memory into context is broken.

**Snapshots solve this:**
- Bounded (always fits)
- Prioritized (high-signal only)
- Stateful (clean transitions)
- Queryable (structured access)

**This is what we're building.**

---

**Next Steps:**

1. Finalize snapshot schema (Snapshot.json) ✓
2. Design event capture flow (Observer → SQLite)
3. Design reranker algorithm (scoring + assembly)
4. Design context injection point (before LLM call)
5. Create tasks.md with implementation breakdown
