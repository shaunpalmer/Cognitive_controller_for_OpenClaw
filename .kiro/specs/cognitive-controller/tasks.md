# Implementation Tasks: Cognitive Controller

**Project:** Cognitive Controller for OpenClaw  
**Phase:** 0 (Foundations)  
**Timeline:** 2 weeks (10 working days)  
**Status:** Ready to Begin

---

## Phase 0: Foundations (Weeks 1-2)

**Goal:** Prove snapshot model works - event capture → snapshot assembly → context injection

---

### Week 1: Foundation & Event Capture

#### Task 1: Project Setup & Infrastructure

**Objective:** Initialize TypeScript project with all dependencies and tooling

- [ ] 1.1 Initialize npm project with TypeScript configuration
  - Create `package.json` with dependencies (better-sqlite3, uuid, etc.)
  - Setup `tsconfig.json` with strict mode
  - Configure build scripts (build, dev, test)
  - Setup ESLint and Prettier

- [~] 1.2 Create directory structure
  - Create `src/` with subdirectories (models, repositories, services, handlers, utils)
  - Create `tests/` with subdirectories (unit, integration)
  - Create `migrations/` for SQL schema files
  - Create `memory/rollups/` for output files

- [~] 1.3 Setup development tooling
  - Configure Jest for testing
  - Setup nodemon for development
  - Create `.gitignore` with appropriate exclusions
  - Setup GitHub Actions for CI (optional)

**Acceptance Criteria:**
- Project builds successfully with `npm run build`
- Tests run with `npm test`
- All directories created and structured correctly

---

#### Task 2: SQLite Database Setup

**Objective:** Create database schema and migration system

- [~] 2.1 Create initial schema migration
  - Write `migrations/001_initial_schema.sql` with all tables
  - Include events, threads, snapshots tables
  - Add all indexes for performance
  - Set PRAGMA settings (WAL mode, foreign keys)

- [~] 2.2 Implement database initialization
  - Create `src/database/connection.ts` for SQLite connection
  - Implement migration runner
  - Add schema version tracking (PRAGMA user_version)
  - Handle database file creation and permissions

- [~] 2.3 Write database utilities
  - Create `src/utils/retry.ts` for retry logic with exponential backoff
  - Create `src/utils/uuid.ts` for UUID generation
  - Add database backup function
  - Add health check function

**Acceptance Criteria:**
- Database initializes with correct schema
- Migrations run successfully
- WAL mode enabled
- All indexes created
- Health check returns valid status

---

#### Task 3: Data Models

**Objective:** Define TypeScript interfaces and types for all data structures

- [~] 3.1 Create event models
  - Define `TierAEvent` interface in `src/models/event.ts`
  - Add event type enums
  - Add domain enums (system, personal, professional)
  - Add validation functions

- [~] 3.2 Create snapshot models
  - Define `Snapshot` interface in `src/models/snapshot.ts`
  - Define `Handshake`, `Thread`, `AtomicFact`, `Decision` interfaces
  - Add `ToolState` and `SnapshotMeta` interfaces
  - Match schema from `Snapshot.json`

- [~] 3.3 Create thread models
  - Define `Thread` interface in `src/models/thread.ts`
  - Add `BlockerType` enum
  - Add `ThreadStatus` enum
  - Add `ConfidenceSignal` interface

- [~] 3.4 Create utility types
  - Define `Result<T, E>` type in `src/utils/result.ts`
  - Add error types (StorageError, ValidationError)
  - Add filter types (EventFilter, FactFilter, DecisionFilter)

**Acceptance Criteria:**
- All interfaces match design document
- All types compile without errors
- Validation functions work correctly
- Result type implements success/failure pattern

---

#### Task 4: Event Repository

**Objective:** Implement SQLite repository for event storage with batch writes

- [~] 4.1 Create event repository interface
  - Define `IEventRepository` in `src/repositories/event.repository.ts`
  - Add methods: appendBatch, findSince, findByRange, purgeOld
  - Use Result<T, E> return types

- [~] 4.2 Implement SQLite event repository
  - Create `SQLiteEventRepository` class
  - Implement batch insert with transactions
  - Add prepared statements for performance
  - Implement query methods with filters

- [~] 4.3 Add error handling
  - Implement retry logic for write failures
  - Add fallback to JSON file on persistent failure
  - Log all errors with context
  - Handle database lock errors

- [~] 4.4 Write unit tests
  - Test batch insert (100 events)
  - Test query with filters (since, until, type, domain)
  - Test purge old events
  - Test error handling and retry logic

**Acceptance Criteria:**
- Batch insert of 1000 events completes in <1 second
- Queries return correct filtered results
- Retry logic works on simulated failures
- All tests pass

---

#### Task 5: Event Observer with Write-Ahead Buffer

**Objective:** Implement event capture with in-memory buffer and 5-second flush

- [~] 5.1 Create observer interface
  - Define `IObserver` in `src/services/observer.service.ts`
  - Add methods: onUserMessage, onToolResult, onStateTransition
  - Add buffer management methods: flushBuffer, getBufferSize

- [~] 5.2 Implement write-ahead buffer
  - Create in-memory buffer (array of TierAEvent)
  - Set buffer size limit (1000 events)
  - Implement 5-second flush interval
  - Implement immediate flush on buffer full

- [~] 5.3 Implement event handlers
  - Transform OpenClaw events to TierAEvent format
  - Classify domain (system/personal/professional)
  - Assign monotonic event_id
  - Add to buffer (non-blocking)

- [~] 5.4 Implement flush logic
  - Batch write buffered events to repository
  - Clear buffer on successful write
  - Retry on failure (3 attempts)
  - Fallback to JSON file on persistent failure

- [~] 5.5 Write unit tests
  - Test event transformation
  - Test buffer fills and flushes
  - Test 5-second interval flush
  - Test fallback on write failure

**Acceptance Criteria:**
- Events buffered in memory
- Flush occurs every 5 seconds OR when buffer reaches 1000 events
- No events lost on write failure (fallback works)
- Event capture completes in <10ms (non-blocking)
- All tests pass

---

### Week 2: Snapshot Assembly & Context Injection

#### Task 6: Token Estimator

**Objective:** Implement token counting for snapshot budget management

- [~] 6.1 Create token estimator interface
  - Define `ITokenEstimator` in `src/utils/token-estimator.ts`
  - Add methods: estimate, validateBudget

- [~] 6.2 Implement simple token estimator
  - Use character-based approximation (1 token ≈ 4 characters)
  - Add method to estimate snapshot tokens
  - Add method to estimate individual field tokens

- [~] 6.3 Write unit tests
  - Test token estimation accuracy (±10% acceptable)
  - Test snapshot token counting
  - Test budget validation

**Acceptance Criteria:**
- Token estimates within ±10% of actual
- Snapshot token counting works for all fields
- Budget validation correctly identifies over-budget snapshots

---

#### Task 7: Reranker Service

**Objective:** Implement scoring algorithm (Risk → Recency → Relevance → Progress)

- [~] 7.1 Create reranker interface
  - Define `IReranker` in `src/services/reranker.service.ts`
  - Add methods: scoreCandidate, rankCandidates, assembleSnapshot

- [~] 7.2 Implement scoring functions
  - Implement `calculateRiskScore` (constraints = 1.0, errors = 0.9)
  - Implement `calculateRecencyScore` (exponential decay)
  - Implement `calculateRelevanceScore` (keyword matching for v1)
  - Implement `calculateProgressScore` (thread status-based)

- [~] 7.3 Implement ranking logic
  - Apply weighted scoring (Risk: 0.4, Recency: 0.3, Relevance: 0.2, Progress: 0.1)
  - Sort candidates by score (descending)
  - Return ranked candidates with score breakdown

- [~] 7.4 Write unit tests
  - Test each scoring function independently
  - Test weighted scoring calculation
  - Test ranking order (constraints first, then recent, etc.)
  - Test with various candidate types

**Acceptance Criteria:**
- Constraints always score highest (1.0 risk score)
- Recent events score higher than old events
- Active threads score higher than stalled threads
- Ranking order matches expected priority
- All tests pass

---

#### Task 8: Snapshot Assembly Service

**Objective:** Build snapshots from ranked candidates within 500 token budget

- [~] 8.1 Create snapshot assembly interface
  - Define `ISnapshotAssembly` in `src/services/snapshot-assembly.service.ts`
  - Add methods: createSnapshot, validateSnapshot, trimToFit

- [~] 8.2 Implement candidate extraction
  - Extract facts from events
  - Extract threads from events
  - Extract constraints from events
  - Extract decisions from events

- [~] 8.3 Implement snapshot assembly
  - Create handshake metadata (snapshot_id, timestamps, event range)
  - Pack fields in priority order until budget reached
  - Ensure all constraints included (never drop)
  - Add objective, threads, facts, decisions, questions, tool_state, actions

- [~] 8.4 Implement validation
  - Check required fields present
  - Check token budget (300-500 tokens)
  - Check constraints included
  - Check at least one thread has next_step

- [~] 8.5 Implement trim-to-fit logic
  - Remove open_questions first
  - Remove excess last_actions
  - Remove lowest-scored facts
  - Remove lowest-confidence threads
  - Never remove constraints

- [~] 8.6 Write unit tests
  - Test snapshot creation with various event sets
  - Test token budget enforcement
  - Test trim-to-fit logic
  - Test validation rules
  - Test constraints never dropped

**Acceptance Criteria:**
- Snapshots created within 300-500 token budget
- All constraints preserved verbatim
- Validation catches missing required fields
- Trim-to-fit reduces tokens without losing critical data
- All tests pass

---

#### Task 9: Snapshot Repository

**Objective:** Implement SQLite repository for snapshot storage

- [~] 9.1 Create snapshot repository interface
  - Define `ISnapshotRepository` in `src/repositories/snapshot.repository.ts`
  - Add methods: save, findLatest, findByThread, purgeOld

- [~] 9.2 Implement SQLite snapshot repository
  - Implement save (insert snapshot as JSON blob)
  - Implement findLatest (query by agent_id or thread_id)
  - Implement findByThread (get all snapshots for thread)
  - Implement purgeOld (keep only last N snapshots per thread)

- [~] 9.3 Add error handling
  - Handle JSON serialization errors
  - Handle database write errors
  - Implement retry logic
  - Log all errors

- [~] 9.4 Write unit tests
  - Test save and retrieve snapshot
  - Test findLatest returns most recent
  - Test purgeOld keeps correct count
  - Test error handling

**Acceptance Criteria:**
- Snapshots saved and retrieved correctly
- JSON serialization/deserialization works
- Purge keeps last N snapshots (configurable)
- All tests pass

---

#### Task 10: Plugin Entry Point & Gateway Integration

**Objective:** Create plugin initialization and hook into OpenClaw events

- [~] 10.1 Create mock Gateway interface
  - Define `IGateway` in `src/interfaces/gateway.interface.ts`
  - Add methods: on, registerMenuItem, registerHealthCheck
  - Create mock implementation for testing

- [~] 10.2 Implement plugin entry point
  - Create `src/index.ts` with initialize and shutdown functions
  - Initialize database on startup
  - Register event handlers
  - Start observer with buffer

- [~] 10.3 Implement event handler registration
  - Hook `agent.message` → observer.onUserMessage
  - Hook `agent.tool_result` → observer.onToolResult
  - Hook `agent.memory_flush` → snapshot creation (CRITICAL)
  - Hook `agent.state_transition` → observer.onStateTransition

- [~] 10.4 Implement graceful shutdown
  - Flush event buffer
  - Close database connection
  - Log shutdown complete

- [~] 10.5 Write integration tests
  - Test plugin initialization
  - Test event handler registration
  - Test event flow (message → buffer → database)
  - Test graceful shutdown

**Acceptance Criteria:**
- Plugin initializes without errors
- Event handlers registered successfully
- Events flow from Gateway → Observer → Database
- Shutdown flushes buffer and closes connections
- All tests pass

---

#### Task 11: Context Injection

**Objective:** Format snapshots for LLM and inject into context before LLM call

- [~] 11.1 Implement snapshot formatter
  - Create `formatSnapshotForLLM` function in `src/utils/snapshot-formatter.ts`
  - Format as structured markdown
  - Include all 9 fields in readable format
  - Keep formatting compact

- [~] 11.2 Implement context injection hook
  - Hook `agent.before_llm_call` event
  - Retrieve latest snapshot for agent
  - Format snapshot
  - Append to context.systemPrompt

- [~] 11.3 Implement memory flush handler
  - Hook `agent.memory_flush` event (CRITICAL)
  - Trigger snapshot creation
  - Save snapshot to database
  - Log snapshot creation

- [~] 11.4 Write unit tests
  - Test snapshot formatting
  - Test context injection
  - Test memory flush triggers snapshot
  - Test formatted output is valid markdown

**Acceptance Criteria:**
- Snapshots formatted as readable markdown
- Context injection adds snapshot to system prompt
- Memory flush triggers snapshot creation
- Formatted snapshot is <500 tokens
- All tests pass

---

#### Task 12: End-to-End Integration Test

**Objective:** Verify complete flow from event capture to context injection

- [~] 12.1 Create integration test scenario
  - Simulate user message event
  - Simulate tool result event
  - Simulate memory flush event
  - Verify snapshot created and injected

- [~] 12.2 Test complete flow
  - Event captured → buffered → persisted
  - Snapshot assembled from events
  - Snapshot validated and saved
  - Snapshot formatted and injected into context

- [~] 12.3 Test performance
  - Measure event capture time (<100ms)
  - Measure snapshot assembly time (<500ms)
  - Measure end-to-end latency (<1 second)

- [~] 12.4 Test error scenarios
  - Database write failure → fallback works
  - Snapshot assembly failure → minimal snapshot created
  - Buffer overflow → flush triggered

**Acceptance Criteria:**
- Complete flow works end-to-end
- Performance targets met
- Error scenarios handled gracefully
- No data loss in failure scenarios

---

## Phase 0 Deliverables

### Code Deliverables
- [ ] TypeScript project with all source files
- [ ] SQLite database with schema and migrations
- [ ] Event observer with write-ahead buffer
- [ ] Reranker with scoring algorithm
- [ ] Snapshot assembly service
- [ ] Plugin entry point with Gateway integration
- [ ] Context injection with snapshot formatting

### Test Deliverables
- [ ] Unit tests for all components (>80% coverage)
- [ ] Integration tests for complete flow
- [ ] Performance tests for key operations

### Documentation Deliverables
- [ ] README.md with setup instructions
- [ ] API documentation for key interfaces
- [ ] Configuration guide
- [ ] Troubleshooting guide

---

## Success Criteria for Phase 0

### Must Have ✅
- [x] Events captured and persisted to SQLite
- [x] Write-ahead buffer prevents event loss
- [x] Snapshots assembled within 500 token budget
- [x] Snapshots injected into context before LLM call
- [x] Memory flush hook triggers snapshot creation
- [x] All constraints preserved verbatim
- [x] No blocking operations in event capture

### Performance Targets
- Event capture: <100ms (including buffer write)
- Snapshot assembly: <500ms (full rebuild)
- Database query: <10ms (indexed queries)
- Buffer flush: <1 second (1000 events)

### Quality Targets
- Test coverage: >80%
- No memory leaks
- No data loss on failures
- Graceful degradation on errors

---

## Out of Scope for Phase 0

The following are explicitly NOT included in Phase 0:

- ❌ Thread management (status, confidence, blockers)
- ❌ Roll-up engine (consolidation to Markdown)
- ❌ UI components (control panel, LED, toasts)
- ❌ Health monitoring (beyond basic health check)
- ❌ Advanced reranking (embeddings, semantic search)
- ❌ Confidence scoring and decay
- ❌ Elastic resolver
- ❌ Multi-agent coordination

These will be implemented in Phase 1 and beyond.

---

## Risk Mitigation

### Risk 1: OpenClaw Plugin API Unknown
**Mitigation:** Start with mock Gateway interface, refine during implementation

### Risk 2: Performance Bottlenecks
**Mitigation:** Implement performance tests early, optimize as needed

### Risk 3: SQLite Write Contention
**Mitigation:** Use WAL mode, batch writes, retry logic

### Risk 4: Token Estimation Inaccuracy
**Mitigation:** Use conservative estimates, validate with actual LLM

---

## Next Steps After Phase 0

1. **User Testing:** Deploy to test environment, gather feedback
2. **Performance Tuning:** Optimize based on real-world usage
3. **Phase 1 Planning:** Thread management, roll-ups, UI
4. **Documentation:** Complete user guide and API docs

---

**Status:** Ready to begin implementation  
**Estimated Effort:** 2 weeks (10 working days)  
**Team Size:** 1 developer  
**Start Date:** Ready immediately
