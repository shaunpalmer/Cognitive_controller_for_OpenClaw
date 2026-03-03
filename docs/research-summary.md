# OpenClaw Research Summary: Key Findings for Cognitive Controller

**Date:** 2026-03-03  
**Questions Answered:** 78/100  
**Status:** Research Phase Complete, Ready for Implementation

---

## Executive Summary

We conducted comprehensive research into OpenClaw's architecture, answering 78 technical questions across 8 major areas. This research revealed critical integration points, constraints, and patterns that will fundamentally shape our Cognitive Controller implementation.

**Most Critical Findings:**

1. **Memory Flush Hook (Q47)** - OpenClaw triggers a "memory flush" turn before every compaction. This is THE perfect moment to create snapshots.

2. **No Shutdown Hook (Q50)** - OpenClaw has NO shutdown hook, meaning crashes can cause data loss. We MUST implement aggressive 5-second flush intervals.

3. **Rate Limits (Q51)** - Only 3 control-plane writes per 60 seconds. We must batch operations carefully.

4. **No Event Replay (Q14)** - Events lost during disconnection are GONE. State recovery is critical.

5. **Per-Agent Isolation (Q40)** - Complete isolation with separate workspaces, databases, and sessions. Clean architecture.

---

## Research Coverage

### Plugin Architecture & Gateway (Q1-Q14)
**Documents:** `docs/openclaw-research-findings.md`

**Key Findings:**
- Plugins are standard npm packages with package.json
- No hot reload - Gateway restart required for infrastructure changes
- Config: `~/.openclaw/openclaw.json` (JSON5 format)
- WebSocket API: `ws://127.0.0.1:18789`
- Authentication mandatory with device token
- Events NOT replayed - must implement state recovery
- Rate limit: 3 requests per 60 seconds

**Impact on Design:**
- Deploy as npm package in `~/.openclaw/skills/cognitive-controller`
- Implement state recovery for crash scenarios
- Track rate limits carefully
- Subscribe to WebSocket events for Observer component

---

### Context Injection & Memory (Q15-Q27)
**Documents:** `docs/openclaw-context-memory-findings.md`

**Key Findings:**
- Bootstrap files injected on first turn (150K char budget, 20K per file)
- Transform modules for per-turn injection at `~/.openclaw/hooks/transforms`
- **Memory flush before compaction** - our hook point!
- Compaction formula: `contextWindow - reserveTokensFloor - softThresholdTokens`
- Reserve tokens floor: 20,000 (configurable)
- Hybrid search: 0.7 vector + 0.3 BM25
- Embedding model: embeddinggemma-300m-qat-Q8_0.gguf
- Temporal decay: 30-day half-life
- 50K embedding cache in SQLite

**Impact on Design:**
- Hook into memory flush event for snapshot creation
- Write snapshots to `memory/rollups/*.canonical.md` for automatic indexing
- Use bootstrap files for initial snapshot injection
- Align with 400-token chunk size for roll-ups
- Leverage OpenClaw's memory_search for fact retrieval

---

### Session Management & Tools (Q28-Q37)
**Documents:** `docs/openclaw-session-tools-findings.md`

**Key Findings:**
- JSONL transcripts at `~/.openclaw/agents/<agentId>/sessions/`
- Full conversation history preserved
- Complete per-agent isolation
- Custom tools via skills directory
- Async/long-running support
- Structured error handling with retry
- Concurrency limits configurable

**Impact on Design:**
- Read session transcripts for snapshot context
- Store snapshots in per-agent directories
- Implement async snapshot creation
- Use structured error handling pattern

---

### Agents & Models (Q38-Q47)
**Documents:** `docs/openclaw-agents-models-findings.md`

**Key Findings:**
- Agent creation: `openclaw agents add <id>`
- Complete isolation (workspace, memory, sessions)
- Shared skills from `~/.openclaw/skills`
- Model config hot-reloads (no restart)
- Model fallback chains supported
- Token estimation via tiktoken
- Streaming responses available
- **Memory flush hook = perfect snapshot trigger**

**Impact on Design:**
- Bootstrap Cognitive Controller on agent creation
- Use isolated paths per agent
- Implement thread-safe operations for concurrent agents
- Use tiktoken for accurate token budgeting
- Adapt to model changes and fallbacks
- Hook into memory flush event (CRITICAL)

---

### Error Handling & Performance (Q48-Q56)
**Documents:** `docs/openclaw-error-performance-findings.md`

**Key Findings:**
- Error taxonomy: RPC, Network, Tool, Auth
- Automatic retry with exponential backoff (30s, 1m, 5m, 15m, 60m)
- Idempotency required for safe retries
- **No event replay** - state recovery critical
- **No shutdown hook** - aggressive persistence required
- **Rate limits: 3 writes per 60 seconds**
- 120s turn timeout (snapshot creation must be <5s)
- 1GB memory limit, 1 CPU core
- 50K embedding cache limit

**Impact on Design:**
- Implement error classification system
- Retry logic with exponential backoff
- Idempotency keys for snapshot operations
- **Aggressive 5-second flush interval**
- State recovery on startup
- Rate limit tracking and batching
- Performance monitoring with <5s target
- Memory usage monitoring

---

### Security & File System (Q57-Q64)
**Documents:** `docs/openclaw-security-filesystem-findings.md`

**Key Findings:**
- Secrets in `~/.openclaw/agents/<agentId>/agent/auth-profiles.json`
- No automatic encryption at rest
- OAuth token refresh uses file locks
- Gateway auth via token handshake
- Required files: AGENTS.md, SOUL.md, USER.md, IDENTITY.md, TOOLS.md
- Reserved: BOOTSTRAP.md, HEARTBEAT.md
- Daily logs: YYYY-MM-DD.md pattern
- Bootstrap limits: 20K per file, 150K total
- File watcher with 1.5s debounce
- File locking for critical operations

**Impact on Design:**
- Store config in auth-profiles.json
- Implement optional database encryption
- Use file locks for critical writes
- Avoid reserved filenames
- Follow YYYY-MM-DD.md pattern for daily logs
- Respect bootstrap file limits
- Implement file watcher with 1.5s debounce
- Single-instance enforcement per agent

---

### Testing & Advanced Features (Q65-Q78)
**Documents:** `docs/openclaw-testing-advanced-findings.md`

**Key Findings:**
- Limited test infrastructure (must build own)
- Synthetic provider for testing
- No hot reload for infrastructure plugins
- Skills hot reload with 250ms debounce
- Versioning: vYYYY.M.D (pre-1.0, fast-moving)
- openclaw doctor for diagnostics and repair
- Web Control UI at http://127.0.0.1:18789/
- Health checks via WebSocket, CLI, Docker
- Heartbeat: 30 min default
- Cron jobs with automatic staggering

**Impact on Design:**
- Build comprehensive test suite
- Use Synthetic provider for CI/CD
- Implement doctor command for diagnostics
- Register health checks with Gateway
- Add Control UI panel
- Implement heartbeat monitoring
- Register cron jobs for maintenance

---

## Critical Integration Points

### 1. Memory Flush Hook (HIGHEST PRIORITY)
**Source:** Q47

OpenClaw runs a silent "memory flush" turn before every compaction, explicitly prompting the agent to write durable memories. This is THE perfect moment to create snapshots.

**Implementation:**
```typescript
gateway.on('memory_flush', async (event) => {
  await memoryFlushHook.onMemoryFlush(event.agentId, event.sessionId);
});
```

**Why Critical:**
- Guaranteed execution before every compaction
- Agent explicitly prompted to preserve state
- Reliable trigger for snapshot creation
- Aligns perfectly with our design

---

### 2. Aggressive Persistence (HIGHEST PRIORITY)
**Source:** Q50, Q14

OpenClaw has NO shutdown hook and NO event replay. Crashes cause data loss, and lost events are GONE forever.

**Implementation:**
- 5-second flush interval
- File locks for critical writes
- State recovery on startup
- Backup before repairs

**Why Critical:**
- Prevents data loss on crashes
- No way to recover lost events
- Must assume crashes will happen

---

### 3. Rate Limit Management (HIGH PRIORITY)
**Source:** Q51

Control-plane write RPCs limited to 3 requests per 60 seconds per device/IP.

**Implementation:**
- Track write operations
- Batch pending operations
- Wait for retryAfterMs on limit
- Prioritize critical writes

**Why Critical:**
- Exceeding limit causes throttling
- Very restrictive (3 per minute)
- Must batch carefully

---

### 4. Per-Agent Isolation (HIGH PRIORITY)
**Source:** Q40

Complete isolation with separate workspaces, memory indexes, and session stores.

**Implementation:**
- Separate SQLite databases per agent
- Separate roll-up directories per agent
- Validate agentId on all operations
- Single-instance enforcement

**Why Critical:**
- Prevents cross-agent data leakage
- Clean architecture
- Security requirement

---

### 5. Token Budget Enforcement (HIGH PRIORITY)
**Source:** Q45

OpenClaw uses tiktoken for token counting with model-specific tokenizers.

**Implementation:**
- Use tiktoken library
- Model-specific tokenizers
- Validate before storage
- Trim iteratively if exceeded

**Why Critical:**
- Must match OpenClaw's estimation
- Ensures snapshots fit budget
- Prevents context overflow

---

## Architecture Decisions Validated by Research

### Decision 1: Structured Snapshots vs Prose
**Validated by:** Q15-Q19 (Memory flush, compaction)

OpenClaw's memory flush explicitly prompts for structured preservation. Our 9-field snapshot design aligns perfectly with this pattern.

### Decision 2: SQLite for Operational Memory
**Validated by:** Q20-Q27 (Memory indexing, performance)

OpenClaw uses SQLite for memory indexing with 50K cache. Our SQLite operational store follows the same pattern.

### Decision 3: Markdown for Archival
**Validated by:** Q15, Q61 (Bootstrap files, roll-ups)

OpenClaw indexes Markdown files in `memory/rollups/` automatically. Our canonical roll-ups will be discovered and searchable.

### Decision 4: Bounded Snapshots (1-10 per thread)
**Validated by:** Q56 (Scalability limits)

OpenClaw's 50K embedding cache and sync thresholds (100KB/50 messages) support bounded growth strategy.

### Decision 5: Reranker Scoring (Risk → Recency → Relevance → Progress)
**Validated by:** Q23 (Temporal decay), Q47 (Memory flush priorities)

OpenClaw's 30-day temporal decay and memory flush priorities align with our reranker scoring.

---

## Requirements Impact Summary

### Original Requirements (R1-R14)
All validated and enhanced with research findings.

### New Requirements Added (R15-R30)

**R15: Memory Flush Hook Integration** - Q47  
**R16: Per-Agent Isolation** - Q40  
**R17: Rate Limit Compliance** - Q51  
**R18: Token Budget Enforcement** - Q45  
**R19: Aggressive Persistence** - Q50, Q14  
**R20: File System Compliance** - Q61-Q62  
**R21: File Watching and Debouncing** - Q63  
**R22: Health Check Integration** - Q73  
**R23: Diagnostic Tool (Doctor)** - Q71  
**R24: Version Compatibility and Migration** - Q68-Q70  
**R25: Security and Privacy** - Q57-Q60  
**R26: Performance Monitoring** - Q53-Q54  
**R27: Cron Job Integration** - Q78  
**R28: Control UI Integration** - Q72  
**R29: Testing Infrastructure** - Q65-Q67  
**R30: Development Mode** - Q66-Q67

---

## Implementation Priorities

### Phase 0: Foundations

**Priority 1: Core Infrastructure**
1. Plugin initialization and Gateway connection (Q1-Q8)
2. Per-agent isolation and path management (Q40)
3. SQLite schema and storage (Q2, Q50)
4. File system compliance (Q61-Q62)

**Priority 2: Event Capture**
1. Observer component (Q3, Q28-Q31)
2. Memory flush hook integration (Q47) - CRITICAL
3. File watcher with debouncing (Q63)
4. Aggressive persistence (Q50, Q14)

**Priority 3: Safety & Reliability**
1. Rate limit tracking (Q51)
2. Error handling and retry logic (Q48-Q49)
3. State recovery (Q50)
4. File locking (Q64)

### Phase 1: Working Memory

**Priority 1: Snapshot System**
1. Snapshot assembly (Q5, Q45)
2. Token budget enforcement with tiktoken (Q45)
3. Reranker scoring (Q7, Q23)
4. Bounded cache (Q9, Q56)

**Priority 2: Thread Management**
1. Thread tracking (Q6)
2. Confidence gradient (Q10)
3. Status transitions (Q6)

**Priority 3: Roll-Ups**
1. Canonical roll-up generation (Q11)
2. Markdown vault integration (Q15, Q61)
3. Fact extraction (Q8)

**Priority 4: Observability**
1. Health checks (Q73)
2. Doctor command (Q71)
3. Control UI panel (Q72)
4. Performance monitoring (Q53, Q26)

---

## Risk Mitigation Updates

### New Risks Identified

1. **No Shutdown Hook** (Q50)
   - Risk: Data loss on crash
   - Mitigation: 5-second flush, state recovery

2. **No Event Replay** (Q14)
   - Risk: Lost events unrecoverable
   - Mitigation: Aggressive persistence, audit logging

3. **Rate Limits** (Q51)
   - Risk: Throttling blocks operations
   - Mitigation: Track limits, batch writes

4. **Pre-1.0 API** (Q68)
   - Risk: Breaking changes
   - Mitigation: Version tracking, migrations

5. **Memory Limits** (Q54)
   - Risk: 1GB limit exceeded
   - Mitigation: Monitor usage, quota management

---

## Testing Strategy

### Unit Tests
- Snapshot assembly with token budgeting
- Reranker scoring algorithms
- Fact extraction
- Thread management
- SQLite operations

### Integration Tests
- Observer event capture
- Memory flush hook
- File watcher
- Rate limit tracking
- State recovery

### E2E Tests
- Plugin initialization
- Full snapshot lifecycle
- Roll-up generation
- Cron job execution
- Doctor diagnostics

### Performance Tests
- Snapshot creation <5s
- Event capture <100ms
- Memory usage <1GB
- Database query performance

---

## Next Steps

1. **Review requirements.md** - All 30 requirements now documented
2. **Update design.md** - Incorporate research findings
3. **Create tasks.md** - Break down implementation into tasks
4. **Begin Phase 0** - Start with core infrastructure
5. **Prototype memory flush hook** - Validate critical integration point

---

## Research Documents

All research findings documented in:

1. `docs/openclaw-research-findings.md` (Q1-Q14)
2. `docs/openclaw-context-memory-findings.md` (Q15-Q27)
3. `docs/openclaw-session-tools-findings.md` (Q28-Q37)
4. `docs/openclaw-agents-models-findings.md` (Q38-Q47)
5. `docs/openclaw-error-performance-findings.md` (Q48-Q56)
6. `docs/openclaw-security-filesystem-findings.md` (Q57-Q64)
7. `docs/openclaw-testing-advanced-findings.md` (Q65-Q78)

---

## Conclusion

The research phase has been incredibly valuable. We discovered critical integration points (memory flush hook), constraints (rate limits, no shutdown hook), and patterns (per-agent isolation, file system conventions) that will fundamentally shape our implementation.

The Cognitive Controller design is now validated against OpenClaw's actual architecture. We have a clear path forward with 30 comprehensive requirements, detailed implementation examples, and a prioritized roadmap.

**We're ready to build.**

