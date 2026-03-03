# Requirements Document: Cognitive Controller Plugin

**Project:** Cognitive Controller for OpenClaw  
**Phase:** 0 (Foundations) & 1 (Working Memory)  
**Status:** Draft  
**Last Updated:** 2026-03-03

---

## Introduction

The Cognitive Controller is an event-driven cognitive layer plugin for OpenClaw that addresses context management challenges in long-running agent sessions. As sessions accumulate skills, steering files, historical data, and notes, context windows fill and the model's working space for reasoning shrinks. When context exceeds 50% capacity, agents exhibit degraded performance and "reactive zombie" behavior.

The Cognitive Controller maintains bounded, high-signal working memory through structured snapshots, SQLite-based operational memory, and intelligent reranking. This requirements document focuses on Phase 0 (Foundations) and Phase 1 (Working Memory), establishing the core infrastructure and memory management capabilities.

---

## Glossary

- **Observer**: Component that captures messages, tool results, and state transitions from OpenClaw
- **Reranker**: Component that scores and selects Top-K threads to build resume blocks
- **Snapshot**: Structured memory artifact (~500 tokens) containing current objective, threads, facts, constraints, and decisions
- **Resume_Block**: Compact context envelope injected into LLM calls, built from snapshot data
- **Thread**: A unit of work with status, confidence, and next steps
- **Tier_A_Event**: High-signal event (tool results, user messages, state transitions) captured by Observer
- **Canonical_Roll_Up**: Deterministic consolidation of events into structured facts, decisions, and constraints
- **Editorial_Roll_Up**: Optional LLM-enhanced readability layer that cites canonical items
- **Heartbeat**: Event-driven health signal triggered by real activity
- **Elastic_Resolver**: Component that handles fail-fast policies, tactic swaps, and model escalation
- **Confidence_Gradient**: Softmax score (0-1) indicating thread health and progress
- **Hard_Constraint**: Verbatim rule that must never be paraphrased or summarized
- **Atomic_Fact**: Timestamped, source-linked, verifiable event record
- **OpenClaw**: AI agent platform that the Cognitive Controller integrates with
- **SQLite_Store**: Append-only database for operational memory (events and snapshots)
- **Markdown_Vault**: Human-readable archival storage for curated roll-ups

---

## Requirements

### Requirement 1: Plugin Infrastructure

**User Story:** As an OpenClaw operator, I want the Cognitive Controller to run as a standard TypeScript plugin, so that I can deploy it without modifying the OpenClaw core.

#### Acceptance Criteria

1. THE Cognitive_Controller SHALL be implemented as a TypeScript plugin compatible with OpenClaw's plugin architecture
2. THE Cognitive_Controller SHALL initialize without requiring modifications to OpenClaw core code
3. THE Cognitive_Controller SHALL support Windows-safe deployment paths
4. WHEN the plugin fails to initialize, THE Cognitive_Controller SHALL log the error and fail gracefully without crashing OpenClaw
5. THE Cognitive_Controller SHALL expose a health check endpoint that returns plugin status

---

### Requirement 2: SQLite Schema and Storage

**User Story:** As a system architect, I want operational memory stored in SQLite with append-only semantics, so that I can maintain high-fidelity recovery points without data loss.

#### Acceptance Criteria

1. THE SQLite_Store SHALL create a database schema with tables for events, snapshots, and threads
2. THE SQLite_Store SHALL implement append-only semantics for event logs
3. THE SQLite_Store SHALL store snapshot data as JSON blobs with metadata (snapshot_id, thread_id, created_at, expires_at, event_range_from, event_range_to)
4. WHEN a write operation fails, THE SQLite_Store SHALL retry up to 3 times before logging an error
5. THE SQLite_Store SHALL support schema migrations for future version upgrades
6. THE SQLite_Store SHALL enforce foreign key constraints between threads and snapshots
7. WHEN the database file is corrupted, THE SQLite_Store SHALL create a backup and initialize a new database

---

### Requirement 3: Event Capture and Observer

**User Story:** As an agent operator, I want all high-signal events captured automatically, so that I can reconstruct agent state at any point in time.

#### Acceptance Criteria

1. WHEN a user message is received, THE Observer SHALL capture it as a Tier_A_Event with timestamp and source metadata
2. WHEN a tool execution completes, THE Observer SHALL capture the tool name, parameters, result, and execution time
3. WHEN a state transition occurs, THE Observer SHALL capture the previous state, new state, and transition reason
4. THE Observer SHALL assign monotonically increasing event IDs to all captured events
5. THE Observer SHALL write events to SQLite_Store within 100ms of capture
6. WHEN event capture fails, THE Observer SHALL buffer up to 1000 events in memory and retry writes
7. THE Observer SHALL tag events with domain classification (system, personal, professional)

---

### Requirement 4: Heartbeat Mechanism

**User Story:** As an operator, I want event-driven heartbeat signals, so that I can monitor system health without synthetic polling overhead.

#### Acceptance Criteria

1. WHEN a Tier_A_Event is captured, THE Observer SHALL emit a heartbeat signal
2. THE Heartbeat SHALL include timestamp, event count since last heartbeat, and active thread count
3. THE Heartbeat SHALL NOT use synthetic timers or polling mechanisms
4. WHEN no events occur for 5 minutes, THE Observer SHALL emit a stale heartbeat signal
5. THE Heartbeat SHALL be accessible via a status endpoint for external monitoring

---

### Requirement 5: Snapshot Structure and Schema

**User Story:** As an AI agent, I want structured snapshots with bounded token size, so that I can resume work with all critical context without exceeding token budgets.

#### Acceptance Criteria

1. THE Snapshot SHALL contain exactly 9 fields: objective_now, active_threads, recent_facts, hard_constraints, decisions, open_questions, tool_state, last_actions, handshake
2. THE Snapshot SHALL consume between 300 and 500 tokens total
3. THE Snapshot SHALL store active_threads as an array with maximum 7 entries
4. THE Snapshot SHALL store recent_facts as an array with maximum 20 entries
5. THE Snapshot SHALL store hard_constraints verbatim without paraphrasing
6. THE Snapshot SHALL include handshake metadata with created_at, expires_at, and source_event_ids
7. WHEN a snapshot exceeds 500 tokens, THE Reranker SHALL remove lowest-priority items until the budget is met
8. THE Snapshot SHALL validate against a JSON schema before storage

---

### Requirement 6: Thread Management

**User Story:** As an agent, I want threads to track work items with status and confidence, so that I can prioritize and resume work effectively.

#### Acceptance Criteria

1. THE Thread SHALL contain fields: thread_id, title, next_step, confidence, status, blocked_by
2. THE Thread.status SHALL be one of: active, stalled, done
3. THE Thread.confidence SHALL be a float between 0.0 and 1.0
4. THE Thread.blocked_by SHALL be an array containing zero or more of: external_dependency, missing_credential, user_input, rate_limit, unknown
5. WHEN a thread is created, THE Thread SHALL default to status=active and confidence=0.5
6. WHEN a thread has no activity for 30 minutes, THE Thread.status SHALL transition to stalled
7. THE Cognitive_Controller SHALL maintain a maximum of 7 active threads simultaneously

---

### Requirement 7: Reranker and Resume Block Assembly

**User Story:** As an agent, I want the reranker to select the most important context, so that I receive high-signal information within token budgets.

#### Acceptance Criteria

1. THE Reranker SHALL score snapshot candidates using priority order: Risk → Recency → Task_Relevance → Progress_Signal
2. THE Reranker SHALL boost constraints and errors (risk scoring)
3. THE Reranker SHALL prioritize newer events over older events (recency scoring)
4. THE Reranker SHALL rank items related to current objective higher (relevance scoring)
5. THE Reranker SHALL rank active/moving threads higher than stalled threads (progress scoring)
6. THE Reranker SHALL assemble snapshot fields until 500 token budget is reached
7. WHEN the token budget is exceeded, THE Reranker SHALL remove lowest-scored items
8. THE Reranker SHALL validate that all hard_constraints are included before finalizing snapshot

---

### Requirement 8: Atomic Facts and Verbatim Constraints

**User Story:** As a system architect, I want facts stored atomically and constraints preserved verbatim, so that I prevent "JPEG artifacts" and memory drift.

#### Acceptance Criteria

1. THE Atomic_Fact SHALL contain fields: fact_id, timestamp, source_type, source_id, content, tags
2. THE Atomic_Fact.source_type SHALL be one of: tool, user, system, db
3. THE Atomic_Fact.content SHALL be verifiable against source events
4. THE Hard_Constraint SHALL be stored exactly as stated without paraphrasing
5. WHEN a constraint is added to a snapshot, THE Reranker SHALL copy it verbatim
6. THE Cognitive_Controller SHALL NOT summarize or improve constraint wording
7. THE Atomic_Fact SHALL include source_id linking back to the originating event

---

### Requirement 9: Bounded Snapshot Cache

**User Story:** As an operator, I want snapshot storage bounded to prevent unbounded growth, so that the system remains performant over time.

#### Acceptance Criteria

1. THE SQLite_Store SHALL maintain between 1 and 10 snapshots per thread
2. WHEN an 11th snapshot is created for a thread, THE SQLite_Store SHALL purge the oldest snapshot
3. THE SQLite_Store SHALL retain snapshots younger than 7 days regardless of count
4. WHEN a snapshot is purged, THE SQLite_Store SHALL log the purge event with snapshot_id and reason
5. THE SQLite_Store SHALL provide a cleanup endpoint for manual snapshot purging
6. THE SQLite_Store SHALL expose metrics for snapshot count per thread

---

### Requirement 10: Confidence Gradient Scoring

**User Story:** As an elastic resolver, I want confidence scores for threads, so that I can make escalation decisions based on thread health.

#### Acceptance Criteria

1. THE Confidence_Gradient SHALL be a float between 0.0 and 1.0
2. WHEN a thread shows consistent progress, THE Confidence_Gradient SHALL increase
3. WHEN a thread is stalled or shows conflicting signals, THE Confidence_Gradient SHALL decrease
4. THE Confidence_Gradient SHALL consider: progress signals, error frequency, time since last activity, blocker presence
5. WHEN confidence drops below 0.3, THE Thread SHALL be marked for escalation review
6. THE Confidence_Gradient SHALL update after each Tier_A_Event related to the thread
7. THE Confidence_Gradient SHALL decay by 0.05 per hour of inactivity (max decay to 0.2)

---

### Requirement 11: Roll-Up Policy (Canonical Only - Phase 1)

**User Story:** As an operator, I want episodic memory consolidated into durable roll-ups, so that I can maintain long-term knowledge without unbounded SQLite growth.

#### Acceptance Criteria

1. THE Roll_Up SHALL consolidate Tier_A_Events into structured facts, decisions, and constraints
2. THE Roll_Up SHALL be triggered by: scheduled task (every 30 minutes), idle detection (no activity for 10 minutes), or manual trigger
3. THE Roll_Up SHALL read from SQLite episodic events and latest snapshot
4. THE Roll_Up SHALL write to Markdown files at `memory/rollups/YYYY-MM-DD-HHMM.canonical.md`
5. THE Roll_Up SHALL NOT summarize or paraphrase hard constraints
6. THE Roll_Up SHALL include event IDs for all facts and decisions (source traceability)
7. THE Roll_Up SHALL be append-only and immutable once written
8. WHEN Roll_Up completes, THE SQLite_Store MAY purge events older than 48 hours

---

### Requirement 12: UI Indicators and Toasts

**User Story:** As an operator, I want visual indicators of system health, so that I can monitor the Cognitive Controller without checking logs.

#### Acceptance Criteria

1. THE Cognitive_Controller SHALL display a 3-state LED indicator: Green (healthy/progressing), Amber (stalled/needs attention), Red (failure/escalation)
2. WHEN a snapshot is created, THE Cognitive_Controller SHALL display a toast notification with thread count and snapshot size
3. WHEN a thread becomes stalled, THE Cognitive_Controller SHALL display an amber toast with thread title
4. WHEN confidence drops below 0.3, THE Cognitive_Controller SHALL display a red toast with escalation recommendation
5. THE LED indicator SHALL update within 1 second of state changes
6. THE Cognitive_Controller SHALL provide a settings toggle to disable toast notifications

---

### Requirement 13: Windows-Safe Deployment

**User Story:** As a Windows user, I want the plugin to handle Windows-specific path and file system constraints, so that deployment succeeds on my platform.

#### Acceptance Criteria

1. THE Cognitive_Controller SHALL use path.join() for all file path construction
2. THE Cognitive_Controller SHALL handle Windows path separators (backslash) correctly
3. THE Cognitive_Controller SHALL avoid reserved Windows filenames (CON, PRN, AUX, NUL)
4. THE Cognitive_Controller SHALL handle long path names (>260 characters) gracefully
5. WHEN file operations fail due to Windows permissions, THE Cognitive_Controller SHALL log the error with actionable guidance
6. THE Cognitive_Controller SHALL support deployment to paths containing spaces

---

### Requirement 14: Error Handling and Graceful Degradation

**User Story:** As an OpenClaw operator, I want the plugin to fail gracefully, so that OpenClaw continues functioning even if the Cognitive Controller encounters errors.

#### Acceptance Criteria

1. WHEN SQLite writes fail, THE Cognitive_Controller SHALL buffer events in memory and retry
2. WHEN the buffer exceeds 1000 events, THE Cognitive_Controller SHALL write to a fallback JSON file
3. WHEN snapshot assembly fails, THE Cognitive_Controller SHALL log the error and return the last valid snapshot
4. WHEN the reranker encounters invalid data, THE Cognitive_Controller SHALL skip the invalid item and continue
5. THE Cognitive_Controller SHALL NOT throw unhandled exceptions that crash OpenClaw
6. WHEN the plugin is disabled, OpenClaw SHALL continue operating normally without Cognitive Controller features

---

## Non-Functional Requirements

### Performance

1. Event capture SHALL complete within 100ms
2. Snapshot assembly SHALL complete within 500ms
3. SQLite queries SHALL use indexes for event_id, thread_id, and timestamp
4. The plugin SHALL consume less than 100MB of memory under normal operation

### Reliability

1. The plugin SHALL achieve 99.9% uptime during OpenClaw sessions
2. SQLite writes SHALL be ACID-compliant
3. The plugin SHALL recover from crashes without data loss (events buffered in memory may be lost)

### Maintainability

1. The codebase SHALL include TypeScript type definitions for all public interfaces
2. The codebase SHALL include unit tests with >80% code coverage
3. The codebase SHALL include integration tests for Observer, Reranker, and SQLite_Store
4. The codebase SHALL follow OpenClaw plugin development guidelines

### Security

1. The plugin SHALL NOT expose sensitive data in logs or toasts
2. The plugin SHALL validate all inputs from OpenClaw hooks
3. The plugin SHALL sanitize file paths to prevent directory traversal attacks

---

---

### Requirement 15: Memory Flush Hook Integration

**User Story:** As an agent, I want snapshots created automatically during OpenClaw's memory flush event, so that critical context is preserved before compaction.

**Research Finding:** Q47 - OpenClaw triggers a "memory flush" turn before every compaction, explicitly prompting the agent to write durable memories. This is THE perfect hook point for snapshot creation.

#### Acceptance Criteria

1. THE Cognitive_Controller SHALL subscribe to OpenClaw's memory flush event
2. WHEN memory flush is triggered, THE Cognitive_Controller SHALL create a snapshot within 5 seconds
3. THE Snapshot SHALL capture current session transcript, memory files (MEMORY.md, USER.md), and existing snapshots
4. THE Snapshot SHALL be written to both SQLite and roll-up directory (`memory/rollups/*.canonical.md`)
5. THE Memory flush hook SHALL have priority over other snapshot triggers
6. WHEN memory flush snapshot creation fails, THE Cognitive_Controller SHALL log error but NOT block OpenClaw's compaction
7. THE Cognitive_Controller SHALL track memory flush events in audit log

---

### Requirement 16: Per-Agent Isolation

**User Story:** As an OpenClaw operator with multiple agents, I want complete isolation between agent memories, so that agents cannot access each other's data.

**Research Finding:** Q40 - OpenClaw provides complete per-agent isolation with separate workspaces, memory indexes, and session stores.

#### Acceptance Criteria

1. THE Cognitive_Controller SHALL maintain separate SQLite databases per agent at `~/.openclaw/agents/<agentId>/.cognitive-controller/memory.db`
2. THE Cognitive_Controller SHALL maintain separate roll-up directories per agent at `<workspace>/memory/rollups/`
3. THE Cognitive_Controller SHALL verify agent isolation on initialization
4. WHEN accessing data, THE Cognitive_Controller SHALL validate agentId matches current context
5. THE Cognitive_Controller SHALL NOT allow cross-agent queries or data access
6. THE Cognitive_Controller SHALL use agent-specific paths for all file operations
7. THE Cognitive_Controller SHALL enforce single-instance per agent using file locks

---

### Requirement 17: Rate Limit Compliance

**User Story:** As a plugin developer, I want to respect OpenClaw's rate limits, so that I don't trigger throttling or errors.

**Research Finding:** Q51 - OpenClaw control-plane write RPCs are limited to 3 requests per 60 seconds per device/IP.

#### Acceptance Criteria

1. THE Cognitive_Controller SHALL track write operations to OpenClaw control plane
2. THE Cognitive_Controller SHALL NOT exceed 3 write operations per 60 seconds
3. WHEN rate limit is approached, THE Cognitive_Controller SHALL batch pending operations
4. WHEN rate limit is exceeded, THE Cognitive_Controller SHALL wait for retryAfterMs before retrying
5. THE Cognitive_Controller SHALL log rate limit events for monitoring
6. THE Cognitive_Controller SHALL prioritize critical writes (snapshots) over non-critical writes (metrics)

---

### Requirement 18: Token Budget Enforcement

**User Story:** As an agent, I want accurate token counting matching OpenClaw's estimation, so that snapshots fit within budget.

**Research Finding:** Q45 - OpenClaw uses tiktoken for token counting with model-specific tokenizers (e.g., cl100k_base for GPT-4).

#### Acceptance Criteria

1. THE Cognitive_Controller SHALL use tiktoken library for token estimation
2. THE Cognitive_Controller SHALL use model-specific tokenizers matching current agent model
3. WHEN creating snapshots, THE Cognitive_Controller SHALL validate token count before storage
4. WHEN snapshot exceeds budget, THE Cognitive_Controller SHALL trim lowest-priority items iteratively
5. THE Cognitive_Controller SHALL log actual vs target token counts for monitoring
6. THE Cognitive_Controller SHALL adapt token budget when agent model changes (Q43 - hot reload)
7. THE Cognitive_Controller SHALL handle model fallback by recalculating budgets (Q44)

---

### Requirement 19: Aggressive Persistence

**User Story:** As an operator, I want data persisted aggressively, so that crashes don't cause data loss.

**Research Finding:** Q50 - OpenClaw has NO shutdown hook, meaning crashes can cause data loss. Q14 - No event replay means lost events are GONE.

#### Acceptance Criteria

1. THE Cognitive_Controller SHALL flush pending writes every 5 seconds
2. THE Cognitive_Controller SHALL use file locks for all critical write operations
3. WHEN buffer exceeds 10 pending snapshots, THE Cognitive_Controller SHALL flush immediately
4. THE Cognitive_Controller SHALL implement state recovery on startup to detect incomplete writes
5. WHEN database corruption is detected, THE Cognitive_Controller SHALL create backup and reinitialize
6. THE Cognitive_Controller SHALL log all flush operations for audit trail
7. THE Cognitive_Controller SHALL prioritize data durability over performance

---

### Requirement 20: File System Compliance

**User Story:** As an OpenClaw user, I want the plugin to follow OpenClaw's file system conventions, so that integration is seamless.

**Research Finding:** Q61 - OpenClaw has specific workspace structure with required files (AGENTS.md, SOUL.md, USER.md, IDENTITY.md, TOOLS.md) and reserved filenames (BOOTSTRAP.md, HEARTBEAT.md).

#### Acceptance Criteria

1. THE Cognitive_Controller SHALL NOT use reserved filenames: BOOTSTRAP.md, HEARTBEAT.md
2. THE Cognitive_Controller SHALL create files in `.cognitive-controller/` directory to avoid conflicts
3. THE Cognitive_Controller SHALL follow YYYY-MM-DD.md pattern for daily logs
4. THE Cognitive_Controller SHALL respect bootstrap file limits: 20K per file, 150K total (Q62)
5. THE Cognitive_Controller SHALL write roll-ups to `memory/rollups/*.canonical.md` for automatic indexing
6. THE Cognitive_Controller SHALL initialize workspace structure on first run
7. THE Cognitive_Controller SHALL validate file sizes before writing to prevent exceeding limits

---

### Requirement 21: File Watching and Debouncing

**User Story:** As a developer, I want file changes detected automatically, so that snapshots stay synchronized with workspace state.

**Research Finding:** Q63 - OpenClaw uses real-time file watcher with 1.5s debounce for memory files.

#### Acceptance Criteria

1. THE Cognitive_Controller SHALL watch MEMORY.md and memory/ directory for changes
2. THE Cognitive_Controller SHALL debounce file changes with 1.5s delay matching OpenClaw
3. WHEN memory files change, THE Cognitive_Controller SHALL extract new facts
4. WHEN fact count exceeds 50 since last snapshot, THE Cognitive_Controller SHALL trigger snapshot creation
5. WHEN time since last snapshot exceeds 1 hour, THE Cognitive_Controller SHALL trigger snapshot creation
6. THE Cognitive_Controller SHALL log file change events for debugging
7. THE Cognitive_Controller SHALL handle rapid file writes without excessive processing

---

### Requirement 22: Health Check Integration

**User Story:** As an operator, I want health status exposed via OpenClaw's health system, so that I can monitor all components in one place.

**Research Finding:** Q73 - OpenClaw provides health command over WebSocket and CLI (openclaw health).

#### Acceptance Criteria

1. THE Cognitive_Controller SHALL register health check with OpenClaw Gateway
2. THE Health check SHALL return status: healthy, degraded, or unhealthy
3. THE Health check SHALL include metrics: snapshot count, fact count, pending writes
4. THE Health check SHALL include per-agent health status
5. THE Health check SHALL detect database corruption and report as unhealthy
6. THE Health check SHALL detect stale snapshots (>1 hour old) and report as degraded
7. THE Health check SHALL respond within 1 second

---

### Requirement 23: Diagnostic Tool (Doctor Command)

**User Story:** As an operator, I want a diagnostic tool to check and repair common issues, so that I can troubleshoot problems quickly.

**Research Finding:** Q71 - OpenClaw provides `openclaw doctor` for diagnostics and auto-repair.

#### Acceptance Criteria

1. THE Cognitive_Controller SHALL provide a doctor command for diagnostics
2. THE Doctor SHALL check: configuration validity, database integrity, file structure, permissions, connectivity, health
3. THE Doctor SHALL auto-fix issues when possible: missing directories, corrupted database, invalid configuration
4. THE Doctor SHALL report issues that cannot be auto-fixed with actionable guidance
5. THE Doctor SHALL create backups before attempting repairs
6. THE Doctor SHALL log all diagnostic runs and repairs
7. THE Doctor SHALL provide summary report: X/Y agents healthy, Z issues fixed

---

### Requirement 24: Version Compatibility and Migration

**User Story:** As an operator, I want automatic migrations when upgrading, so that I don't lose data or configuration.

**Research Finding:** Q68-Q70 - OpenClaw uses vYYYY.M.D versioning, is pre-1.0 (fast-moving), and auto-runs doctor migrations on startup.

#### Acceptance Criteria

1. THE Cognitive_Controller SHALL track its version in `version.txt` per agent
2. THE Cognitive_Controller SHALL check OpenClaw version compatibility on startup
3. THE Cognitive_Controller SHALL require OpenClaw >= v2026.1.1
4. WHEN version mismatch detected, THE Cognitive_Controller SHALL run migrations automatically
5. THE Cognitive_Controller SHALL log all migrations applied with descriptions
6. THE Cognitive_Controller SHALL backup data before running migrations
7. THE Cognitive_Controller SHALL fail gracefully if migration fails, preserving original data

---

### Requirement 25: Security and Privacy

**User Story:** As an operator, I want sensitive data protected, so that secrets and PII are not exposed.

**Research Finding:** Q57-Q60 - OpenClaw stores secrets in auth-profiles.json (no encryption), recommends encrypting backups, has no built-in PII redaction.

#### Acceptance Criteria

1. THE Cognitive_Controller SHALL support optional database encryption via environment variable
2. THE Cognitive_Controller SHALL store configuration in per-agent auth-profiles.json
3. THE Cognitive_Controller SHALL NOT log sensitive data (API keys, tokens, PII)
4. THE Cognitive_Controller SHALL provide PII detection and redaction for facts
5. THE Cognitive_Controller SHALL support GDPR compliance: data export, deletion, anonymization
6. THE Cognitive_Controller SHALL use file permissions (0600) for sensitive files
7. THE Cognitive_Controller SHALL validate webhook authentication tokens

---

### Requirement 26: Performance Monitoring

**User Story:** As a developer, I want performance metrics tracked, so that I can identify bottlenecks and optimize.

**Research Finding:** Q53 - OpenClaw has 120s turn timeout, 4s memory search timeout, targets ~400 token chunks.

#### Acceptance Criteria

1. THE Cognitive_Controller SHALL track operation timings: snapshot creation, fact extraction, database writes
2. THE Cognitive_Controller SHALL target snapshot creation < 5 seconds (well under 120s turn timeout)
3. THE Cognitive_Controller SHALL log slow operations (>5s) as warnings
4. THE Cognitive_Controller SHALL calculate percentiles: p50, p95, p99 for all operations
5. THE Cognitive_Controller SHALL expose performance metrics via health endpoint
6. THE Cognitive_Controller SHALL monitor memory usage and warn if approaching 1GB limit (Q54)
7. THE Cognitive_Controller SHALL log performance metrics every 10 minutes in development mode

---

### Requirement 27: Cron Job Integration

**User Story:** As an operator, I want scheduled tasks for maintenance, so that cleanup and roll-ups happen automatically.

**Research Finding:** Q78 - OpenClaw supports cron jobs with 5/6-field expressions, automatic staggering for top-of-hour jobs.

#### Acceptance Criteria

1. THE Cognitive_Controller SHALL register cron job for hourly snapshot creation (0 * * * *)
2. THE Cognitive_Controller SHALL register cron job for daily roll-up (0 2 * * *)
3. THE Cognitive_Controller SHALL register cron job for weekly cleanup (0 3 * * 0)
4. THE Cron jobs SHALL run in isolated sessions to avoid blocking main agent
5. THE Cron jobs SHALL log execution start, duration, and result
6. WHEN cron job fails, THE Cognitive_Controller SHALL log error but NOT crash
7. THE Cognitive_Controller SHALL respect OpenClaw's automatic staggering for load management

---

### Requirement 28: Control UI Integration

**User Story:** As an operator, I want a visual dashboard in OpenClaw's Control UI, so that I can monitor Cognitive Controller without CLI.

**Research Finding:** Q72 - OpenClaw serves Control UI at http://127.0.0.1:18789/ with agent management, session viewer, configuration.

#### Acceptance Criteria

1. THE Cognitive_Controller SHALL register UI panel at /cognitive-controller
2. THE UI panel SHALL display per-agent health status with color coding
3. THE UI panel SHALL display latest snapshot summary: ID, created time, objective, thread count
4. THE UI panel SHALL display recent facts (last 10) per agent
5. THE UI panel SHALL auto-refresh every 10 seconds
6. THE UI panel SHALL provide links to detailed views: snapshots, facts, roll-ups
7. THE UI panel SHALL display performance metrics: snapshot creation time, fact count, database size

---

### Requirement 29: Testing Infrastructure

**User Story:** As a developer, I want comprehensive tests, so that I can refactor confidently without breaking functionality.

**Research Finding:** Q65-Q67 - OpenClaw has limited test infrastructure, supports Synthetic provider for testing, no hot reload for infrastructure plugins.

#### Acceptance Criteria

1. THE Cognitive_Controller SHALL include unit tests with >80% code coverage
2. THE Cognitive_Controller SHALL include integration tests for: Observer, Reranker, SQLite_Store, Snapshot Assembly
3. THE Cognitive_Controller SHALL use Synthetic provider for LLM-free testing
4. THE Cognitive_Controller SHALL include E2E tests for: plugin initialization, event capture, snapshot creation, roll-up generation
5. THE Tests SHALL run in CI/CD pipeline on every commit
6. THE Tests SHALL use isolated test agents to prevent data contamination
7. THE Tests SHALL clean up test data after execution

---

### Requirement 30: Development Mode

**User Story:** As a developer, I want enhanced debugging in development mode, so that I can troubleshoot issues quickly.

**Research Finding:** Q66-Q67 - OpenClaw supports verbose logging via --verbose flag, logs viewable via `openclaw logs --follow`.

#### Acceptance Criteria

1. WHEN NODE_ENV=development, THE Cognitive_Controller SHALL enable verbose logging
2. THE Development mode SHALL log all snapshot creation steps with context
3. THE Development mode SHALL log all fact extraction with confidence scores
4. THE Development mode SHALL log performance metrics every 10 seconds
5. THE Development mode SHALL watch source files and warn when restart needed
6. THE Development mode SHALL emit events to Control UI for real-time monitoring
7. THE Development mode SHALL validate all operations against performance targets

---

## Out of Scope (Future Phases)

The following are explicitly out of scope for Phase 0 and Phase 1:

- Editorial roll-ups (LLM-enhanced readability layer)
- Elastic resolver (fail-fast policies, tactic swaps, model escalation)
- Obsidian vault integration
- Semantic memory search beyond OpenClaw's built-in memory_search
- Multi-agent coordination via Agent Send (Phase 4+)
- External API integrations
- QMD backend integration (experimental feature)
- Custom embedding models (use OpenClaw's default)

---

## Success Criteria

Phase 0 and Phase 1 are considered successful when:

1. The plugin initializes and runs on Windows without errors
2. Events are captured and stored in SQLite with <1% data loss
3. Snapshots are generated within 500 token budget with all constraints preserved
4. Threads are tracked with confidence scores that update based on activity
5. Roll-ups are generated and written to Markdown without data loss
6. UI indicators reflect system health accurately
7. OpenClaw continues functioning normally if the plugin fails
8. **Memory flush hook captures snapshots before every compaction (Q47)**
9. **Per-agent isolation verified with no cross-agent data access (Q40)**
10. **Rate limits respected: <3 writes per 60 seconds (Q51)**
11. **Token estimation matches OpenClaw using tiktoken (Q45)**
12. **Aggressive persistence: 5s flush interval, no data loss on crash (Q50)**
13. **File system compliance: follows OpenClaw conventions (Q61-Q62)**
14. **Health checks integrated with OpenClaw health system (Q73)**
15. **Doctor command diagnoses and repairs common issues (Q71)**
16. **Automatic migrations on version upgrades (Q68-Q70)**
17. **Performance targets met: snapshot creation <5s (Q53)**
18. **Cron jobs registered and executing on schedule (Q78)**
19. **Control UI panel displays agent status and metrics (Q72)**
20. **Test suite passes with >80% coverage (Q65)**

---

## Acceptance Testing

### Test Scenario 1: Plugin Initialization
1. Install plugin in OpenClaw
2. Start OpenClaw
3. Verify plugin initializes without errors
4. Verify SQLite database is created
5. Verify health check endpoint returns status

### Test Scenario 2: Event Capture
1. Send user message to OpenClaw
2. Execute a tool
3. Verify both events are captured in SQLite
4. Verify event IDs are monotonically increasing
5. Verify timestamps are accurate

### Test Scenario 3: Snapshot Assembly
1. Generate 50 Tier_A_Events
2. Trigger snapshot creation
3. Verify snapshot contains 9 fields
4. Verify token count is between 300-500
5. Verify all hard constraints are present verbatim

### Test Scenario 4: Thread Management
1. Create 3 threads
2. Mark 1 thread as stalled
3. Complete 1 thread
4. Verify thread states are correct
5. Verify confidence scores update appropriately

### Test Scenario 5: Roll-Up Generation
1. Generate 100 Tier_A_Events over 30 minutes
2. Trigger roll-up
3. Verify Markdown file is created
4. Verify facts include event IDs
5. Verify constraints are verbatim
6. Verify file is immutable (cannot be overwritten)

### Test Scenario 6: Bounded Cache
1. Create 12 snapshots for a single thread
2. Verify only 10 snapshots are retained
3. Verify oldest snapshots are purged first
4. Verify purge events are logged

### Test Scenario 7: Graceful Degradation
1. Corrupt SQLite database file
2. Restart plugin
3. Verify plugin creates backup and new database
4. Verify OpenClaw continues functioning
5. Verify error is logged with actionable guidance

---

## Dependencies

- OpenClaw v2026.1.1+ (pre-1.0, fast-moving API)
- SQLite3 (via better-sqlite3 TypeScript library)
- TypeScript 4.5+
- Node.js 16+
- tiktoken (for accurate token estimation matching OpenClaw)
- proper-lockfile (for file locking during critical writes)
- chokidar (for file watching with debouncing)

---

## Risks and Mitigations

| Risk | Impact | Likelihood | Mitigation | Research Source |
|------|--------|------------|------------|-----------------|
| OpenClaw plugin API changes | High | High | Version pin OpenClaw >= v2026.1.1, maintain compatibility layer, track breaking changes | Q68-Q70 |
| SQLite corruption | High | Low | Automatic backup and recovery, fallback to JSON, aggressive 5s flush | Q50 |
| Token budget estimation errors | Medium | Low | Use tiktoken matching OpenClaw, validate before storage, adapt to model changes | Q45 |
| Windows path handling issues | Medium | Medium | Extensive Windows testing, use path.join(), avoid reserved names | Q61 |
| Performance degradation with large event logs | Medium | High | Implement event purging after 48h, index optimization, bounded cache (1-10 snapshots) | Q9, Q56 |
| Reranker scoring produces poor snapshots | High | Medium | Tune scoring weights after Phase 0, add manual override, validate constraints present | Q7 |
| Rate limit throttling | High | Medium | Track writes, batch operations, respect 3 per 60s limit | Q51 |
| No shutdown hook causes data loss | High | Medium | Aggressive 5s flush, file locks, state recovery on startup | Q50, Q14 |
| Memory flush hook not triggered | High | Low | Fallback to time-based snapshots (hourly), file change triggers | Q47 |
| Database exceeds 1GB memory limit | Medium | Medium | Monitor usage, implement quota management, cleanup old data | Q54, Q56 |
| Cross-agent data leakage | High | Low | Enforce per-agent isolation, validate agentId on all operations, use file locks | Q40 |
| PII exposure in logs/snapshots | Medium | Medium | Implement PII detection/redaction, sanitize logs, GDPR compliance | Q60 |

---

**Document Owner:** Project Team  
**Review Status:** Draft  
**Next Review:** After Phase 0 implementation begins
