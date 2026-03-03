# Phase 0 Readiness Assessment

**Date:** 2026-03-03  
**Purpose:** Determine if we have enough information to start Phase 0 implementation  
**Status:** Ready to Begin

---

## What We Have (Complete Foundation)

### ✅ 1. Core Problem Definition

**Document:** `docs/core-problem-statement.md`

**What's clear:**
- Markdown memory breaks at scale (context overflow)
- Skills multiply the problem (30k-75k tokens)
- Compaction causes information loss (agent amnesia)
- Solution: Bounded snapshots (500 tokens, state-based)

**Readiness:** ✅ COMPLETE

---

### ✅ 2. Architectural Approach (Established in Early Docs)

**Documents:** `docs/Note.md`, `docs/decisions-record.md`, `docs/Roll-Up.md`

**What's established:**

#### Memory Tiering (3-Layer Model)
```
Tier 1: Hot (SQLite In-Memory)
  - High fidelity, every event
  - Retention: 1 session
  
Tier 2: Episodic (SQLite Disk)
  - Summary-only, grouped events
  - Retention: 30 days
  
Tier 3: Semantic (Markdown)
  - Canonical knowledge only
  - Retention: Forever
```

#### Gateway Tap Pattern
```
OpenClaw Gateway
    ↓
Primary Loop (OpenClaw execution)
    ↓
Secondary Stream (Non-blocking capture)
    ↓
Event Store (SQLite)
    ↓
Snapshot Assembly
```

#### Key Architectural Decisions
1. **Structured snapshots** over prose summaries (prevents drift)
2. **Atomic facts** + verbatim constraints (no paraphrasing)
3. **SQLite for operational**, Markdown for archival
4. **Canonical vs Editorial** roll-ups (two-stage pipeline)
5. **Bounded cache** (1-10 snapshots per thread)
6. **500 token budget** with structured allocation
7. **Risk → Recency → Relevance → Progress** scoring
8. **Event-driven heartbeat** (no synthetic timers)
9. **TypeScript plugin** (no core modifications)

**Readiness:** ✅ COMPLETE

---

### ✅ 3. Data Models

**Documents:** `Snapshot.json`, `docs/snapshot-schema-analysis.md`, `docs/Note.md`

**What's defined:**

#### Snapshot Schema (9 Fields)
```json
{
  "handshake": {
    "snapshot_id": "uuid",
    "created_at": "ISO8601",
    "expires_at": "ISO8601",
    "source_event_ids": [1234, 5678],
    "trigger": "memory_flush",
    "schema_version": "1.0"
  },
  "objective_now": "string (~20 tokens)",
  "active_threads": [
    {
      "thread_id": "uuid",
      "title": "string",
      "next_step": "string",
      "confidence": 0.85,
      "confidence_last_updated": "ISO8601",
      "status": "active|stalled|done",
      "blocked_by": ["user_input"]
    }
  ],
  "recent_facts": [
    {
      "fact_id": "uuid",
      "timestamp": "ISO8601",
      "source_type": "tool|user|system|db",
      "source_id": "event_123",
      "content": "string",
      "tags": ["api", "auth"],
      "confidence": 1.0
    }
  ],
  "hard_constraints": [
    {
      "constraint_id": "uuid",
      "content": "string (verbatim)",
      "added_at": "ISO8601",
      "immutable": true
    }
  ],
  "decisions": [
    {
      "decision_id": "uuid",
      "what": "string",
      "why": "string",
      "who": "user|system|operator",
      "when": "ISO8601",
      "how": "string",
      "revisit_if": ["condition"]
    }
  ],
  "open_questions": ["string"],
  "tool_state": {
    "available": ["tool1", "tool2"],
    "unavailable": ["tool3"],
    "health": "healthy|degraded|failed"
  },
  "last_actions": [
    {
      "action_id": "uuid",
      "action": "string",
      "outcome": "success|failure|partial"
    }
  ],
  "meta": {
    "agent_id": "string",
    "model": "string",
    "token_count_actual": 487,
    "token_estimation_model": "tiktoken"
  }
}
```

#### SQLite Schema
```sql
-- Events (Append-Only)
CREATE TABLE events (
  event_id INTEGER PRIMARY KEY AUTOINCREMENT,
  timestamp TEXT NOT NULL,
  event_type TEXT NOT NULL,
  category TEXT NOT NULL, -- system|personal|professional
  thread_id TEXT,
  payload JSON,
  importance_score REAL
);

-- Snapshots (Bounded 1-10 per thread)
CREATE TABLE snapshots (
  snapshot_id TEXT PRIMARY KEY,
  thread_id TEXT,
  created_at TEXT NOT NULL,
  expires_at TEXT NOT NULL,
  data TEXT NOT NULL, -- JSON blob
  token_count INTEGER
);

-- Threads (Work Item Tracking)
CREATE TABLE threads (
  thread_id TEXT PRIMARY KEY,
  title TEXT NOT NULL,
  next_step TEXT,
  confidence REAL DEFAULT 0.5,
  status TEXT DEFAULT 'active',
  blocked_by TEXT, -- JSON array
  created_at TEXT NOT NULL,
  updated_at TEXT NOT NULL,
  last_activity_at TEXT NOT NULL
);
```

**Readiness:** ✅ COMPLETE

---

### ✅ 4. Integration Points

**Documents:** `docs/openclaw-research-findings.md`, `docs/openclaw-agents-models-findings.md`, `docs/architecture-patterns-analysis.md`

**What's known:**

#### OpenClaw Extension Points
1. **Event Hooks** - `gateway.on(eventType, handler)`
   - `agent.message` - User messages
   - `agent.tool_result` - Tool execution results
   - `agent.memory_flush` - Before compaction (CRITICAL)
   - `agent.state_transition` - State changes

2. **UI Extension** - `gateway.registerMenuItem(config)`
   - Add to left sidebar menu
   - Control panel with tabs
   - Shows threads, snapshots, roll-ups, settings

3. **Health Checks** - `gateway.registerHealthCheck(name, fn)`
   - Expose LED status
   - Thread count, confidence scores

4. **File System** - Standard paths
   - `memory/rollups/*.md` - Our roll-up files
   - `cognitive-controller.db` - SQLite database

#### Critical Integration Findings (from Research)
- **Q47: Memory Flush Hook** - THE perfect moment for snapshots
- **Q50: No Shutdown Hook** - Must use aggressive 5-second flush intervals
- **Q51: Rate Limits** - Only 3 writes per 60 seconds (batch operations)
- **Q40: Per-Agent Isolation** - Complete isolation validates per-agent architecture
- **Q14: No Event Replay** - Lost events are GONE (state recovery critical)

**Readiness:** ✅ COMPLETE

---

### ✅ 5. Requirements

**Document:** `.kiro/specs/cognitive-controller/requirements.md`

**What's defined:**
- 30 comprehensive requirements (R1-R30)
- Covers all aspects: snapshots, threads, persistence, roll-ups, health, security
- Includes research sources (Q numbers)
- Clear acceptance criteria

**Readiness:** ✅ COMPLETE

---

### ✅ 6. Design Document

**Document:** `.kiro/specs/cognitive-controller/design.md`

**What's defined:**
- Component designs (Observer, Reranker, Snapshot Assembly, SQLite Store, Roll-Up Engine, Thread Manager)
- Data models with TypeScript interfaces
- SQLite schema with indexes
- Integration with OpenClaw (plugin lifecycle, hook registration, context injection)
- Error handling strategies
- Performance considerations
- Testing strategy
- Deployment approach

**Status:** Comprehensive but needs architecture section update

**Readiness:** ✅ 95% COMPLETE (needs architecture alignment)

---

## What We Need to Clarify

### ⚠️ 1. Exact OpenClaw Plugin API

**What we know:**
- Plugins are npm packages
- Entry point: `export async function initialize(gateway: Gateway)`
- Event subscription: `gateway.on(eventType, handler)`
- UI registration: `gateway.registerMenuItem(config)`

**What we need:**
- Exact TypeScript interfaces for Gateway API
- Event payload schemas
- UI component framework (React? Vue? Custom?)
- Health check registration API

**Impact:** Medium - Can start with assumptions and adjust

**Mitigation:** Start with mock Gateway interface, refine during implementation

---

### ⚠️ 2. Write-Ahead Buffer Implementation

**From Note.md:**
> "You need a Write-Ahead Buffer. Don't write every heartbeat to SQLite immediately. Collect them in memory and flush them in chunks every 5–10 seconds."

**What we need:**
- Buffer size (1000 events?)
- Flush triggers (time-based? size-based? both?)
- Failure handling (what if flush fails?)

**Impact:** High - Critical for performance

**Decision needed:** 
- Buffer size: 1000 events
- Flush interval: 5 seconds OR buffer full
- On failure: Retry 3x, then fallback to JSON file

---

### ⚠️ 3. Cross-Pollination Index (Personal vs Professional)

**From Note.md:**
> "You need a Cross-Pollination Index. Instead of hard-segregated folders, use a Tag-based retrieval system where 'Professional' is just a high-weight filter, not a wall."

**What we need:**
- Tag-based routing instead of hard category split?
- How to handle cross-domain events?

**Impact:** Medium - Affects event classification

**Decision needed:**
- Phase 0: Simple category (system|personal|professional)
- Phase 1+: Add tags for cross-pollination

---

### ⚠️ 4. Emergency TTL (Time To Live)

**From Note.md:**
> "You need an Emergency TTL. If an event is older than 48 hours and hasn't been consolidated, it should be auto-archived to a 'Raw Dump' file."

**What we need:**
- TTL threshold (48 hours?)
- Auto-archive location
- Cleanup job schedule

**Impact:** Medium - Prevents unbounded growth

**Decision needed:**
- TTL: 48 hours
- Archive to: `memory/archive/YYYY-MM-DD-raw.jsonl`
- Cleanup: Daily cron job

---

## Phase 0 Scope (Weeks 1-2)

### Goal: Prove Snapshot Model Works

**What we're building:**

#### 1. Event Capture (Observer)
```typescript
class EventObserver {
  private buffer: TierAEvent[] = [];
  private flushInterval: NodeJS.Timer;
  
  constructor(private eventRepo: EventRepository) {
    // Flush every 5 seconds
    this.flushInterval = setInterval(() => this.flush(), 5000);
  }
  
  async onMessage(event: MessageEvent): Promise<void> {
    const tierAEvent = this.transform(event);
    this.buffer.push(tierAEvent);
    
    // Flush if buffer full
    if (this.buffer.length >= 1000) {
      await this.flush();
    }
  }
  
  private async flush(): Promise<void> {
    if (this.buffer.length === 0) return;
    
    const events = [...this.buffer];
    this.buffer = [];
    
    try {
      await this.eventRepo.appendBatch(events);
    } catch (error) {
      // Fallback to JSON file
      await this.fallbackWrite(events);
    }
  }
}
```

#### 2. SQLite Store
```typescript
class SQLiteEventRepository {
  async appendBatch(events: TierAEvent[]): Promise<void> {
    const stmt = this.db.prepare(`
      INSERT INTO events (timestamp, event_type, category, thread_id, payload)
      VALUES (?, ?, ?, ?, ?)
    `);
    
    const transaction = this.db.transaction((events) => {
      for (const event of events) {
        stmt.run(
          event.timestamp,
          event.event_type,
          event.category,
          event.thread_id,
          JSON.stringify(event.payload)
        );
      }
    });
    
    transaction(events);
  }
}
```

#### 3. Basic Reranker
```typescript
class Reranker {
  rank(candidates: Candidate[], context: Context): RankedCandidate[] {
    return candidates.map(c => ({
      ...c,
      score: this.calculateScore(c, context)
    })).sort((a, b) => b.score - a.score);
  }
  
  private calculateScore(c: Candidate, ctx: Context): number {
    const risk = c.type === 'constraint' ? 1.0 : 0.0;
    const recency = Math.exp(-getAgeMinutes(c.timestamp) / 60);
    const relevance = this.calculateRelevance(c, ctx);
    
    return (risk * 0.4) + (recency * 0.3) + (relevance * 0.3);
  }
}
```

#### 4. Snapshot Assembly
```typescript
class SnapshotService {
  async createSnapshot(agentId: string): Promise<Snapshot> {
    // 1. Get recent events
    const events = await this.eventRepo.findSince(lastSnapshotTime);
    
    // 2. Extract candidates
    const candidates = this.extractCandidates(events);
    
    // 3. Rank
    const ranked = this.reranker.rank(candidates, context);
    
    // 4. Assemble
    const snapshot = this.assemble(ranked);
    
    // 5. Validate token budget
    if (this.estimateTokens(snapshot) > 500) {
      return this.trimToFit(snapshot, 500);
    }
    
    return snapshot;
  }
}
```

#### 5. Context Injection
```typescript
export async function initialize(gateway: Gateway): Promise<void> {
  // Hook before LLM call
  gateway.on('agent.before_llm_call', async (context) => {
    const snapshot = await snapshotService.getLatest(context.agentId);
    const formatted = formatSnapshotForLLM(snapshot);
    context.systemPrompt += `\n\n${formatted}`;
  });
  
  // Hook memory flush (critical!)
  gateway.on('agent.memory_flush', async (event) => {
    await snapshotService.createSnapshot(event.agentId);
  });
}
```

---

## Readiness Verdict

### ✅ READY TO START PHASE 0

**What we have:**
- ✅ Clear problem definition
- ✅ Established architectural approach
- ✅ Complete data models
- ✅ Known integration points
- ✅ Comprehensive requirements
- ✅ Detailed design document
- ✅ Research findings (78/100 questions answered)

**What we'll discover during implementation:**
- Exact OpenClaw plugin API details
- Performance tuning (buffer sizes, flush intervals)
- Edge cases in event handling
- UI component integration specifics

**Risk level:** LOW

**Confidence:** HIGH

---

## Phase 0 Implementation Plan

### Week 1: Foundation

**Day 1-2: Project Setup**
- Initialize TypeScript project
- Setup SQLite with better-sqlite3
- Create database schema
- Write migration scripts

**Day 3-4: Event Capture**
- Implement EventObserver with write-ahead buffer
- Implement SQLiteEventRepository with batch writes
- Add fallback JSON writer
- Unit tests

**Day 5: Integration Stub**
- Create mock Gateway interface
- Implement plugin entry point
- Test event capture flow

### Week 2: Snapshot Assembly

**Day 6-7: Reranker**
- Implement scoring algorithm (Risk → Recency → Relevance)
- Implement candidate extraction
- Unit tests

**Day 8-9: Snapshot Service**
- Implement snapshot assembly
- Implement token estimation
- Implement trim-to-fit logic
- Unit tests

**Day 10: Context Injection**
- Implement snapshot formatting for LLM
- Hook into memory flush event
- Integration tests

---

## Success Criteria for Phase 0

### Must Have
✅ Events captured and persisted to SQLite  
✅ Snapshots assembled within 500 token budget  
✅ Snapshots injected into context before LLM call  
✅ Memory flush hook triggers snapshot creation  
✅ Write-ahead buffer prevents event loss  

### Nice to Have
- UI panel showing latest snapshot
- Health check endpoint
- Metrics collection

### Out of Scope (Phase 1)
- Thread management
- Confidence scoring
- Roll-up engine
- Advanced reranking

---

## Next Action

**Create tasks.md** with detailed implementation breakdown for Phase 0.

**Estimated effort:** 2 weeks (10 working days)

**Team size:** 1 developer

**Start date:** Ready to begin immediately

---

**Assessment:** ✅ WE HAVE ENOUGH TO START PHASE 0
