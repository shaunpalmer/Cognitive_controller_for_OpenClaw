# Design Document: Cognitive Controller Plugin

**Project:** Cognitive Controller for OpenClaw  
**Phase:** 0 (Foundations) & 1 (Working Memory)  
**Status:** Draft  
**Last Updated:** 2026-03-03

---

## Executive Summary

The Cognitive Controller is a TypeScript plugin for OpenClaw that provides structured, bounded memory management for long-running agent sessions. It addresses context window bloat through event capture, intelligent reranking, and structured snapshots that maintain high-signal working memory within a 300-500 token budget.

This design document covers the architecture, component interactions, data models, and implementation approach for Phase 0 (Foundations) and Phase 1 (Working Memory).

---

## Architecture

### Architectural Pattern: Hexagonal Architecture + Event-Driven Loop

The Cognitive Controller follows a **Hexagonal Architecture** (Ports & Adapters) combined with an **Event-Driven Loop** for horizontal flow. This provides clean separation of concerns with dependency inversion.

**Horizontal (Event Loop):** Circular execution responding to OpenClaw events  
**Vertical (Layers):** Clean separation of concerns with dependency inversion

```
┌─────────────────────────────────────────────────────────────┐
│                    EVENT-DRIVEN LOOP                         │
│  OpenClaw Events → Process → Snapshot → Persist → Respond   │
└─────────────────────────────────────────────────────────────┘
                              ↕
┌─────────────────────────────────────────────────────────────┐
│                    VERTICAL LAYERS                           │
│  Presentation → Application → Domain → Infrastructure        │
└─────────────────────────────────────────────────────────────┘
```

### Core Architectural Principles

1. **Domain at the Center** - Pure business logic, no dependencies
2. **Ports Define Contracts** - Interfaces for all external interactions
3. **Adapters Implement Ports** - OpenClaw, SQLite, File System adapters
4. **Event Loop Orchestrates** - Horizontal flow through vertical layers
5. **Dependency Inversion** - All dependencies point inward

### Layer Structure

#### Layer 1: Domain (Core)

**Purpose:** Pure business logic, no external dependencies

**Components:**
- `Snapshot` - Value object (immutable)
- `Thread` - Entity with identity and lifecycle
- `AtomicFact` - Value object
- `HardConstraint` - Value object (immutable)
- `ConfidenceGradient` - Value object with decay logic
- `TokenBudget` - Value object with validation

**Patterns:**
- Value Objects (immutable)
- Entities (with identity)
- Domain Services (pure functions)

**Example:**
```typescript
// domain/snapshot.ts
export class Snapshot {
  private constructor(
    public readonly handshake: Handshake,
    public readonly objectiveNow: string,
    public readonly activeThreads: Thread[],
    public readonly recentFacts: AtomicFact[],
    public readonly hardConstraints: HardConstraint[],
    public readonly decisions: Decision[],
    public readonly openQuestions: Question[],
    public readonly toolState: ToolState,
    public readonly lastActions: Action[],
    public readonly meta: SnapshotMeta
  ) {
    Object.freeze(this); // Immutable
  }

  static create(data: SnapshotData): Result<Snapshot, ValidationError> {
    // Validation logic
    if (!data.objectiveNow) {
      return Result.fail(new ValidationError('objective_now required'));
    }
    
    return Result.ok(new Snapshot(/* ... */));
  }

  // Domain logic
  exceedsTokenBudget(budget: TokenBudget): boolean {
    return this.meta.tokenCountActual > budget.value;
  }

  trimToFitBudget(budget: TokenBudget): Snapshot {
    // Pure function - returns new snapshot
    let trimmed = this;
    
    while (trimmed.exceedsTokenBudget(budget)) {
      trimmed = trimmed.removeLowestPriorityItem();
    }
    
    return trimmed;
  }
}
```

#### Layer 2: Application (Use Cases)

**Purpose:** Orchestrate domain logic, define ports

**Components:**
- `SnapshotAssemblyService` - Assembles snapshots from events
- `RerankerService` - Scores and prioritizes snapshot items
- `RollUpService` - Consolidates events into roll-ups
- `ConfidenceCalculator` - Updates thread confidence
- `EventProcessor` - Processes incoming events

**Patterns:**
- Service Layer
- Use Case pattern
- Port interfaces (contracts)

**Ports (Interfaces):**
```typescript
// application/ports/storage.port.ts
export interface ISnapshotRepository {
  save(snapshot: Snapshot): Promise<Result<void, StorageError>>;
  findLatest(agentId: string): Promise<Result<Snapshot | null, StorageError>>;
  findByThread(threadId: string): Promise<Result<Snapshot[], StorageError>>;
  deleteOldest(threadId: string): Promise<Result<void, StorageError>>;
}

export interface IEventRepository {
  append(event: TierAEvent): Promise<Result<void, StorageError>>;
  findByRange(from: string, to: string): Promise<Result<TierAEvent[], StorageError>>;
  findSince(timestamp: Date): Promise<Result<TierAEvent[], StorageError>>;
}

// application/ports/openclaw.port.ts
export interface IOpenClawGateway {
  subscribeToMemoryFlush(handler: MemoryFlushHandler): void;
  subscribeToEvents(handler: EventHandler): void;
  getSessionTranscript(sessionId: string): Promise<Transcript>;
  getModelConfig(agentId: string): Promise<ModelConfig>;
  registerHealthCheck(name: string, check: HealthCheck): void;
}

// application/ports/filesystem.port.ts
export interface IFileSystem {
  writeRollUp(agentId: string, content: string): Promise<Result<void, FileError>>;
  watchMemoryFiles(agentId: string, handler: FileChangeHandler): void;
  readMemoryFile(agentId: string, filename: string): Promise<Result<string, FileError>>;
}

// application/ports/tokenizer.port.ts
export interface ITokenEstimator {
  estimate(text: string, model: string): number;
  validateBudget(snapshot: Snapshot, budget: TokenBudget): ValidationResult;
}
```

**Use Case Example:**
```typescript
// application/use-cases/create-snapshot.use-case.ts
export class CreateSnapshotUseCase {
  constructor(
    private readonly snapshotRepo: ISnapshotRepository,
    private readonly eventRepo: IEventRepository,
    private readonly tokenEstimator: ITokenEstimator,
    private readonly reranker: RerankerService
  ) {}

  async execute(request: CreateSnapshotRequest): Promise<Result<Snapshot, Error>> {
    // 1. Gather events
    const eventsResult = await this.eventRepo.findSince(request.since);
    if (eventsResult.isFailure) return Result.fail(eventsResult.error);
    
    // 2. Extract facts, threads, constraints
    const facts = this.extractFacts(eventsResult.value);
    const threads = this.extractThreads(eventsResult.value);
    const constraints = this.extractConstraints(eventsResult.value);
    
    // 3. Rerank and prioritize
    const ranked = this.reranker.rank({
      facts,
      threads,
      constraints,
      objective: request.objective
    });
    
    // 4. Assemble snapshot
    const snapshotResult = Snapshot.create({
      objectiveNow: request.objective,
      activeThreads: ranked.threads,
      recentFacts: ranked.facts,
      hardConstraints: ranked.constraints,
      // ... other fields
    });
    
    if (snapshotResult.isFailure) return snapshotResult;
    
    // 5. Validate token budget
    const snapshot = snapshotResult.value;
    const budget = new TokenBudget(500);
    
    if (snapshot.exceedsTokenBudget(budget)) {
      const trimmed = snapshot.trimToFitBudget(budget);
      await this.snapshotRepo.save(trimmed);
      return Result.ok(trimmed);
    }
    
    // 6. Save
    await this.snapshotRepo.save(snapshot);
    return Result.ok(snapshot);
  }
}
```

#### Layer 3: Infrastructure (Adapters)

**Purpose:** Implement ports, handle external systems

**Adapters:**
- `SQLiteSnapshotRepository` - Implements ISnapshotRepository
- `SQLiteEventRepository` - Implements IEventRepository
- `OpenClawGatewayAdapter` - Implements IOpenClawGateway
- `NodeFileSystemAdapter` - Implements IFileSystem
- `TiktokenEstimator` - Implements ITokenEstimator

**Patterns:**
- Adapter pattern
- Repository pattern
- Gateway pattern

**Example:**
```typescript
// infrastructure/adapters/sqlite-snapshot.repository.ts
export class SQLiteSnapshotRepository implements ISnapshotRepository {
  constructor(private readonly db: Database) {}

  async save(snapshot: Snapshot): Promise<Result<void, StorageError>> {
    try {
      const stmt = this.db.prepare(`
        INSERT INTO snapshots (
          snapshot_id, agent_id, thread_id, created_at, 
          expires_at, data, token_count
        ) VALUES (?, ?, ?, ?, ?, ?, ?)
      `);
      
      stmt.run(
        snapshot.handshake.snapshotId,
        snapshot.meta.agentId,
        snapshot.activeThreads[0]?.threadId || null,
        snapshot.handshake.createdAt,
        snapshot.handshake.expiresAt,
        JSON.stringify(snapshot),
        snapshot.meta.tokenCountActual
      );
      
      return Result.ok(undefined);
    } catch (error) {
      return Result.fail(new StorageError(error.message));
    }
  }

  async findLatest(agentId: string): Promise<Result<Snapshot | null, StorageError>> {
    try {
      const row = this.db.prepare(`
        SELECT data FROM snapshots 
        WHERE agent_id = ? 
        ORDER BY created_at DESC 
        LIMIT 1
      `).get(agentId);
      
      if (!row) return Result.ok(null);
      
      const snapshotResult = Snapshot.create(JSON.parse(row.data));
      return snapshotResult.isSuccess 
        ? Result.ok(snapshotResult.value)
        : Result.fail(new StorageError('Invalid snapshot data'));
    } catch (error) {
      return Result.fail(new StorageError(error.message));
    }
  }
}
```

#### Layer 4: Presentation (Event Loop)

**Purpose:** Orchestrate the event-driven loop

**Components:**
- `EventLoop` - Main orchestrator (Singleton)
- `MemoryFlushHandler` - Handles memory flush events
- `FileChangeHandler` - Handles file change events
- `CronJobHandler` - Handles scheduled tasks

**Patterns:**
- Singleton (EventLoop)
- Observer pattern (event handlers)
- Command pattern (handlers)

**Event Loop:**
```typescript
// presentation/event-loop.ts
export class EventLoop {
  private static instance: EventLoop;
  private isRunning = false;
  
  private constructor(
    private readonly gateway: IOpenClawGateway,
    private readonly createSnapshotUseCase: CreateSnapshotUseCase,
    private readonly processEventUseCase: ProcessEventUseCase,
    private readonly rollUpUseCase: CreateRollUpUseCase
  ) {}

  static getInstance(/* dependencies */): EventLoop {
    if (!EventLoop.instance) {
      EventLoop.instance = new EventLoop(/* ... */);
    }
    return EventLoop.instance;
  }

  async start(): Promise<void> {
    if (this.isRunning) return;
    this.isRunning = true;

    // Subscribe to OpenClaw events
    this.gateway.subscribeToMemoryFlush(this.handleMemoryFlush.bind(this));
    this.gateway.subscribeToEvents(this.handleEvent.bind(this));
    
    logger.info('Event loop started');
  }

  private async handleMemoryFlush(event: MemoryFlushEvent): Promise<void> {
    logger.info(`Memory flush: ${event.agentId}:${event.sessionId}`);
    
    // Execute use case
    const result = await this.createSnapshotUseCase.execute({
      agentId: event.agentId,
      sessionId: event.sessionId,
      trigger: 'memory_flush',
      since: event.lastSnapshotTime
    });
    
    if (result.isFailure) {
      logger.error(`Snapshot creation failed: ${result.error.message}`);
      return;
    }
    
    logger.info(`Snapshot created: ${result.value.handshake.snapshotId}`);
    
    // Trigger roll-up if needed
    await this.rollUpUseCase.execute({
      agentId: event.agentId,
      trigger: 'memory_flush'
    });
  }

  private async handleEvent(event: TierAEvent): Promise<void> {
    // Process event through use case
    await this.processEventUseCase.execute(event);
  }

  async stop(): Promise<void> {
    this.isRunning = false;
    logger.info('Event loop stopped');
  }
}
```

### Dependency Injection Container

**Pattern:** Factory + Dependency Injection

```typescript
// infrastructure/di-container.ts
export class DIContainer {
  private static instance: DIContainer;
  private services: Map<string, any> = new Map();

  private constructor() {}

  static getInstance(): DIContainer {
    if (!DIContainer.instance) {
      DIContainer.instance = new DIContainer();
    }
    return DIContainer.instance;
  }

  register<T>(key: string, factory: () => T): void {
    this.services.set(key, factory);
  }

  resolve<T>(key: string): T {
    const factory = this.services.get(key);
    if (!factory) {
      throw new Error(`Service not registered: ${key}`);
    }
    return factory();
  }

  // Bootstrap all dependencies
  bootstrap(): void {
    // Infrastructure
    this.register('Database', () => new Database(':memory:'));
    this.register('ISnapshotRepository', () => 
      new SQLiteSnapshotRepository(this.resolve('Database'))
    );
    this.register('IEventRepository', () => 
      new SQLiteEventRepository(this.resolve('Database'))
    );
    this.register('IOpenClawGateway', () => 
      new OpenClawGatewayAdapter()
    );
    this.register('IFileSystem', () => 
      new NodeFileSystemAdapter()
    );
    this.register('ITokenEstimator', () => 
      new TiktokenEstimator()
    );

    // Application Services
    this.register('RerankerService', () => 
      new RerankerService()
    );
    this.register('CreateSnapshotUseCase', () => 
      new CreateSnapshotUseCase(
        this.resolve('ISnapshotRepository'),
        this.resolve('IEventRepository'),
        this.resolve('ITokenEstimator'),
        this.resolve('RerankerService')
      )
    );

    // Event Loop
    this.register('EventLoop', () => 
      EventLoop.getInstance(
        this.resolve('IOpenClawGateway'),
        this.resolve('CreateSnapshotUseCase'),
        this.resolve('ProcessEventUseCase'),
        this.resolve('CreateRollUpUseCase')
      )
    );
  }
}
```

### Plugin Entry Point

```typescript
// index.ts
import { DIContainer } from './infrastructure/di-container';
import { EventLoop } from './presentation/event-loop';

export async function initialize(gateway: Gateway): Promise<void> {
  logger.info('Initializing Cognitive Controller...');

  // Bootstrap DI container
  const container = DIContainer.getInstance();
  container.bootstrap();

  // Start event loop
  const eventLoop = container.resolve<EventLoop>('EventLoop');
  await eventLoop.start();

  logger.info('Cognitive Controller initialized');
}

export async function shutdown(): Promise<void> {
  logger.info('Shutting down Cognitive Controller...');

  const container = DIContainer.getInstance();
  const eventLoop = container.resolve<EventLoop>('EventLoop');
  await eventLoop.stop();

  logger.info('Cognitive Controller shut down');
}
```

### Directory Structure

```
src/
├── domain/                    # Layer 1: Pure business logic
│   ├── entities/
│   │   ├── thread.ts
│   │   └── snapshot.ts
│   ├── value-objects/
│   │   ├── atomic-fact.ts
│   │   ├── hard-constraint.ts
│   │   ├── confidence-gradient.ts
│   │   └── token-budget.ts
│   └── services/
│       └── domain-services.ts
│
├── application/               # Layer 2: Use cases & ports
│   ├── ports/
│   │   ├── storage.port.ts
│   │   ├── openclaw.port.ts
│   │   ├── filesystem.port.ts
│   │   └── tokenizer.port.ts
│   ├── use-cases/
│   │   ├── create-snapshot.use-case.ts
│   │   ├── process-event.use-case.ts
│   │   └── create-rollup.use-case.ts
│   └── services/
│       ├── reranker.service.ts
│       └── confidence-calculator.service.ts
│
├── infrastructure/            # Layer 3: Adapters
│   ├── adapters/
│   │   ├── sqlite-snapshot.repository.ts
│   │   ├── sqlite-event.repository.ts
│   │   ├── openclaw-gateway.adapter.ts
│   │   ├── node-filesystem.adapter.ts
│   │   └── tiktoken.estimator.ts
│   ├── di-container.ts
│   └── database/
│       └── schema.sql
│
├── presentation/              # Layer 4: Event loop
│   ├── event-loop.ts
│   ├── handlers/
│   │   ├── memory-flush.handler.ts
│   │   ├── file-change.handler.ts
│   │   └── cron-job.handler.ts
│   └── controllers/
│       └── health-check.controller.ts
│
├── shared/                    # Cross-cutting concerns
│   ├── result.ts             # Result<T, E> monad
│   ├── logger.ts
│   └── errors.ts
│
└── index.ts                   # Plugin entry point
```

### Key Design Patterns

#### 1. Singleton Pattern
**Where:** EventLoop, DIContainer  
**Why:** Single instance per plugin, global coordination

#### 2. Factory Pattern
**Where:** DIContainer, Snapshot.create()  
**Why:** Controlled object creation with validation

#### 3. Repository Pattern
**Where:** SnapshotRepository, EventRepository  
**Why:** Abstract data access, testable

#### 4. Adapter Pattern
**Where:** All infrastructure adapters  
**Why:** Decouple from external systems

#### 5. Observer Pattern
**Where:** Event handlers  
**Why:** React to OpenClaw events

#### 6. Result Pattern (Railway-Oriented Programming)
**Where:** All use cases and repositories  
**Why:** Explicit error handling, no exceptions

```typescript
// shared/result.ts
export class Result<T, E> {
  private constructor(
    public readonly isSuccess: boolean,
    public readonly value?: T,
    public readonly error?: E
  ) {}

  static ok<T, E>(value: T): Result<T, E> {
    return new Result(true, value, undefined);
  }

  static fail<T, E>(error: E): Result<T, E> {
    return new Result(false, undefined, error);
  }

  get isFailure(): boolean {
    return !this.isSuccess;
  }
}
```

### Benefits of This Architecture

#### 1. Testability
- Domain logic pure functions (easy to test)
- Ports allow mocking external systems
- Use cases testable in isolation

#### 2. Maintainability
- Clear separation of concerns
- Dependencies point inward
- Easy to locate and modify code

#### 3. Flexibility
- Swap adapters without changing domain
- Add new use cases easily
- Extend without modifying existing code

#### 4. OpenClaw Integration
- Adapter isolates OpenClaw specifics
- Easy to update when OpenClaw changes
- Can mock for testing

#### 5. Per-Agent Isolation
- DIContainer can create per-agent instances
- Thread-safe by design
- Clear boundaries

### System Context

```
┌─────────────────────────────────────────────────────────────┐
│                        OpenClaw Core                         │
│  ┌────────────┐  ┌────────────┐  ┌────────────┐            │
│  │   User     │  │   Tools    │  │   LLM      │            │
│  │  Messages  │  │  Execution │  │  Inference │            │
│  └─────┬──────┘  └─────┬──────┘  └─────┬──────┘            │
│        │               │               │                     │
│        └───────────────┴───────────────┘                     │
│                        │                                     │
│                   Plugin Hooks                               │
└────────────────────────┼────────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────────┐
│              Cognitive Controller Plugin                     │
│                                                              │
│  ┌──────────────────────────────────────────────────────┐  │
│  │              Presentation Layer (Event Loop)          │  │
│  │  ┌──────────┐   ┌──────────┐   ┌──────────┐         │  │
│  │  │ Memory   │   │   File   │   │   Cron   │         │  │
│  │  │  Flush   │   │  Change  │   │   Job    │         │  │
│  │  │ Handler  │   │ Handler  │   │ Handler  │         │  │
│  │  └──────────┘   └──────────┘   └──────────┘         │  │
│  └──────────────────────────────────────────────────────┘  │
│                         │                                    │
│                         ▼                                    │
│  ┌──────────────────────────────────────────────────────┐  │
│  │           Application Layer (Use Cases)              │  │
│  │  ┌──────────┐   ┌──────────┐   ┌──────────┐         │  │
│  │  │  Create  │   │ Process  │   │  Create  │         │  │
│  │  │ Snapshot │   │  Event   │   │ Roll-Up  │         │  │
│  │  └──────────┘   └──────────┘   └──────────┘         │  │
│  └──────────────────────────────────────────────────────┘  │
│                         │                                    │
│                         ▼                                    │
│  ┌──────────────────────────────────────────────────────┐  │
│  │              Domain Layer (Business Logic)           │  │
│  │  ┌──────────┐   ┌──────────┐   ┌──────────┐         │  │
│  │  │ Snapshot │   │  Thread  │   │   Fact   │         │  │
│  │  │ (Entity) │   │ (Entity) │   │  (Value) │         │  │
│  │  └──────────┘   └──────────┘   └──────────┘         │  │
│  └──────────────────────────────────────────────────────┘  │
│                         │                                    │
│                         ▼                                    │
│  ┌──────────────────────────────────────────────────────┐  │
│  │         Infrastructure Layer (Adapters)              │  │
│  │  ┌──────────┐   ┌──────────┐   ┌──────────┐         │  │
│  │  │  SQLite  │   │ OpenClaw │   │   File   │         │  │
│  │  │   Repo   │   │ Gateway  │   │  System  │         │  │
│  │  └──────────┘   └──────────┘   └──────────┘         │  │
│  └──────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────┘
                         │
                         ▼
                  ┌──────────────┐
                  │   Markdown   │
                  │    Vault     │
                  │  (Roll-Ups)  │
                  └──────────────┘
```

### Component Responsibilities

| Component | Layer | Responsibility | Inputs | Outputs |
|-----------|-------|---------------|--------|---------|
| EventLoop | Presentation | Orchestrate event-driven flow | OpenClaw events | Use case invocations |
| CreateSnapshotUseCase | Application | Assemble snapshots from events | Events, threads, constraints | Snapshot |
| RerankerService | Application | Score and prioritize context | Candidates, context | Ranked candidates |
| Snapshot (Entity) | Domain | Immutable snapshot with validation | Snapshot data | Validated snapshot |
| Thread (Entity) | Domain | Thread lifecycle and confidence | Thread data, signals | Updated thread |
| SQLiteSnapshotRepository | Infrastructure | Persist snapshots | Snapshot | Storage result |
| OpenClawGatewayAdapter | Infrastructure | Interface with OpenClaw | Event subscriptions | OpenClaw events |
| NodeFileSystemAdapter | Infrastructure | File operations | Roll-up content | File write result |

---

## Data Models

### Event Schema

```typescript
interface TierAEvent {
  event_id: number;           // Monotonically increasing
  timestamp: string;          // ISO 8601
  event_type: 'user_message' | 'tool_result' | 'state_transition';
  domain: 'system' | 'personal' | 'professional';
  source_id: string;          // Reference to originating entity
  content: string;            // Event payload
  metadata: Record<string, any>;
  tags: string[];
}
```

### Thread Schema

```typescript
interface Thread {
  thread_id: string;          // UUID
  title: string;              // Brief description
  next_step: string;          // Immediate action required
  confidence: number;         // 0.0 - 1.0 (softmax gradient)
  status: 'active' | 'stalled' | 'done';
  blocked_by: BlockerType[];
  created_at: string;         // ISO 8601
  updated_at: string;         // ISO 8601
  last_activity_at: string;   // ISO 8601
}

type BlockerType = 
  | 'external_dependency' 
  | 'missing_credential' 
  | 'user_input' 
  | 'rate_limit' 
  | 'unknown';
```

### Snapshot Schema

```typescript
interface Snapshot {
  // Metadata
  handshake: {
    snapshot_id: string;      // UUID
    created_at: string;       // ISO 8601
    expires_at: string;       // ISO 8601
    source_event_ids: number[]; // Event range
    schema_version: string;   // "1.0"
  };
  
  // Core fields
  objective_now: string;      // ~20 tokens: current objective
  
  active_threads: Thread[];   // Max 7 threads, ~100 tokens
  
  recent_facts: AtomicFact[]; // Max 20 facts, ~150 tokens
  
  hard_constraints: string[]; // Verbatim, ~80 tokens
  
  decisions: Decision[];      // 2-3 key decisions, ~100 tokens
  
  open_questions: string[];   // ~50 tokens
  
  tool_state: {               // ~30 tokens
    available: string[];
    unavailable: string[];
    health: 'healthy' | 'degraded' | 'failed';
  };
  
  last_actions: string[];     // Last 3-5 actions, ~50 tokens
}
```

### Atomic Fact Schema

```typescript
interface AtomicFact {
  fact_id: string;            // UUID
  timestamp: string;          // ISO 8601
  source_type: 'tool' | 'user' | 'system' | 'db';
  source_id: string;          // Link to originating event
  content: string;            // Verifiable statement
  tags: string[];
  confidence: number;         // 0.0 - 1.0
}
```

### Decision Schema

```typescript
interface Decision {
  decision_id: string;        // UUID
  what: string;               // What was decided
  why: string;                // Rationale
  who: 'user' | 'system' | 'operator';
  when: string;               // ISO 8601 timestamp
  how: string;                // Implementation approach
  revisit_if: string[];       // Conditions for revisiting
}
```

---

## SQLite Schema

### Tables

```sql
-- Events table (append-only)
CREATE TABLE events (
  event_id INTEGER PRIMARY KEY AUTOINCREMENT,
  timestamp TEXT NOT NULL,
  event_type TEXT NOT NULL,
  domain TEXT NOT NULL,
  source_id TEXT NOT NULL,
  content TEXT NOT NULL,
  metadata TEXT,  -- JSON blob
  tags TEXT       -- JSON array
);

CREATE INDEX idx_events_timestamp ON events(timestamp);
CREATE INDEX idx_events_type ON events(event_type);
CREATE INDEX idx_events_domain ON events(domain);

-- Threads table
CREATE TABLE threads (
  thread_id TEXT PRIMARY KEY,
  title TEXT NOT NULL,
  next_step TEXT,
  confidence REAL NOT NULL DEFAULT 0.5,
  status TEXT NOT NULL DEFAULT 'active',
  blocked_by TEXT,  -- JSON array
  created_at TEXT NOT NULL,
  updated_at TEXT NOT NULL,
  last_activity_at TEXT NOT NULL
);

CREATE INDEX idx_threads_status ON threads(status);
CREATE INDEX idx_threads_confidence ON threads(confidence);
CREATE INDEX idx_threads_updated ON threads(updated_at);

-- Snapshots table
CREATE TABLE snapshots (
  snapshot_id TEXT PRIMARY KEY,
  thread_id TEXT,
  created_at TEXT NOT NULL,
  expires_at TEXT NOT NULL,
  event_range_from INTEGER NOT NULL,
  event_range_to INTEGER NOT NULL,
  snapshot_data TEXT NOT NULL,  -- JSON blob
  token_count INTEGER,
  FOREIGN KEY (thread_id) REFERENCES threads(thread_id) ON DELETE CASCADE
);

CREATE INDEX idx_snapshots_thread ON snapshots(thread_id);
CREATE INDEX idx_snapshots_created ON snapshots(created_at);
CREATE INDEX idx_snapshots_expires ON snapshots(expires_at);

-- Facts table
CREATE TABLE facts (
  fact_id TEXT PRIMARY KEY,
  timestamp TEXT NOT NULL,
  source_type TEXT NOT NULL,
  source_id TEXT NOT NULL,
  content TEXT NOT NULL,
  tags TEXT,  -- JSON array
  confidence REAL NOT NULL DEFAULT 1.0
);

CREATE INDEX idx_facts_timestamp ON facts(timestamp);
CREATE INDEX idx_facts_source ON facts(source_id);

-- Decisions table
CREATE TABLE decisions (
  decision_id TEXT PRIMARY KEY,
  what TEXT NOT NULL,
  why TEXT NOT NULL,
  who TEXT NOT NULL,
  when TEXT NOT NULL,
  how TEXT NOT NULL,
  revisit_if TEXT  -- JSON array
);

CREATE INDEX idx_decisions_when ON decisions(when);
CREATE INDEX idx_decisions_who ON decisions(who);
```

---

## Component Design

### Observer Component

**Purpose:** Capture high-signal events from OpenClaw and persist to SQLite.

**Interfaces:**

```typescript
interface IObserver {
  // Hook into OpenClaw events
  onUserMessage(message: string, metadata: any): Promise<void>;
  onToolResult(toolName: string, params: any, result: any): Promise<void>;
  onStateTransition(from: string, to: string, reason: string): Promise<void>;
  
  // Heartbeat
  emitHeartbeat(): void;
  getLastHeartbeat(): Heartbeat;
  
  // Buffer management
  flushBuffer(): Promise<void>;
  getBufferSize(): number;
}

interface Heartbeat {
  timestamp: string;
  event_count: number;
  active_thread_count: number;
  status: 'healthy' | 'stale' | 'degraded';
}
```

**Implementation Details:**

1. **Event Capture Flow:**
   - Hook receives event from OpenClaw
   - Observer creates TierAEvent object
   - Assigns monotonic event_id
   - Classifies domain (system/personal/professional)
   - Writes to SQLite within 100ms
   - Emits heartbeat signal

2. **Buffer Management:**
   - In-memory buffer holds up to 1000 events
   - Buffer flushes on: successful write, buffer full, or manual trigger
   - On write failure: retry 3 times with exponential backoff
   - On persistent failure: write to fallback JSON file

3. **Domain Classification:**
   - System: Plugin lifecycle, errors, health checks
   - Personal: User preferences, settings
   - Professional: Task work, tool execution, decisions

**Error Handling:**
- SQLite write failure → buffer + retry
- Buffer overflow → fallback JSON file
- Hook failure → log error, continue (don't crash OpenClaw)

---

### Reranker Component

**Purpose:** Score and prioritize context candidates to build optimal snapshots.

**Interfaces:**

```typescript
interface IReranker {
  // Scoring
  scoreCandidate(candidate: Candidate, context: ScoringContext): number;
  rankCandidates(candidates: Candidate[]): RankedCandidate[];
  
  // Assembly
  assembleSnapshot(rankedCandidates: RankedCandidate[], budget: number): Snapshot;
  validateSnapshot(snapshot: Snapshot): ValidationResult;
}

interface Candidate {
  type: 'fact' | 'constraint' | 'decision' | 'thread' | 'question';
  content: any;
  source_event_id: number;
  timestamp: string;
}

interface ScoringContext {
  current_objective: string;
  active_threads: Thread[];
  last_snapshot: Snapshot | null;
  recent_errors: TierAEvent[];
}

interface RankedCandidate extends Candidate {
  score: number;
  score_breakdown: {
    risk: number;
    recency: number;
    relevance: number;
    progress: number;
  };
}
```

**Scoring Algorithm:**

```typescript
function scoreCandidate(candidate: Candidate, context: ScoringContext): number {
  const weights = {
    risk: 0.4,      // Highest priority
    recency: 0.3,
    relevance: 0.2,
    progress: 0.1
  };
  
  const scores = {
    risk: calculateRiskScore(candidate),
    recency: calculateRecencyScore(candidate),
    relevance: calculateRelevanceScore(candidate, context),
    progress: calculateProgressScore(candidate, context)
  };
  
  return (
    scores.risk * weights.risk +
    scores.recency * weights.recency +
    scores.relevance * weights.relevance +
    scores.progress * weights.progress
  );
}

function calculateRiskScore(candidate: Candidate): number {
  // Constraints and errors get maximum score (1.0)
  if (candidate.type === 'constraint') return 1.0;
  if (isError(candidate)) return 0.9;
  return 0.0;
}

function calculateRecencyScore(candidate: Candidate): number {
  const ageMinutes = getAgeInMinutes(candidate.timestamp);
  // Exponential decay: score = e^(-age/60)
  return Math.exp(-ageMinutes / 60);
}

function calculateRelevanceScore(candidate: Candidate, context: ScoringContext): number {
  // Semantic similarity to current objective
  // For v1: keyword matching, upgrade to embeddings in v2
  const objectiveKeywords = extractKeywords(context.current_objective);
  const candidateKeywords = extractKeywords(candidate.content);
  const overlap = intersection(objectiveKeywords, candidateKeywords);
  return overlap.length / objectiveKeywords.length;
}

function calculateProgressScore(candidate: Candidate, context: ScoringContext): number {
  // Active/moving threads score higher than stalled
  if (candidate.type === 'thread') {
    const thread = candidate.content as Thread;
    if (thread.status === 'active' && thread.confidence > 0.7) return 1.0;
    if (thread.status === 'stalled') return 0.3;
    if (thread.status === 'done') return 0.1;
  }
  return 0.5;
}
```

**Assembly Process:**

1. Extract candidates from event stream (last N hours)
2. Score each candidate using priority algorithm
3. Sort by score (descending)
4. Pack fields in order until token budget reached:
   - Handshake metadata (~20 tokens)
   - Objective (~20 tokens)
   - Constraints (all, ~80 tokens)
   - Active threads (top 7, ~100 tokens)
   - Recent facts (top 15, ~150 tokens)
   - Decisions (top 3, ~100 tokens)
   - Questions + tool_state + actions (~50 tokens)
5. Validate: all constraints present, objective clear, at least 1 next step
6. Return snapshot or error

**Token Estimation:**

```typescript
function estimateTokens(text: string): number {
  // Rough approximation: 1 token ≈ 4 characters
  return Math.ceil(text.length / 4);
}

function estimateSnapshotTokens(snapshot: Snapshot): number {
  let total = 0;
  total += estimateTokens(JSON.stringify(snapshot.handshake));
  total += estimateTokens(snapshot.objective_now);
  total += snapshot.active_threads.reduce((sum, t) => sum + estimateTokens(JSON.stringify(t)), 0);
  total += snapshot.recent_facts.reduce((sum, f) => sum + estimateTokens(f.content), 0);
  total += snapshot.hard_constraints.reduce((sum, c) => sum + estimateTokens(c), 0);
  total += snapshot.decisions.reduce((sum, d) => sum + estimateTokens(JSON.stringify(d)), 0);
  total += estimateTokens(JSON.stringify(snapshot.tool_state));
  total += snapshot.last_actions.reduce((sum, a) => sum + estimateTokens(a), 0);
  return total;
}
```

---

### Snapshot Assembly Component

**Purpose:** Build and validate structured snapshots within token budget.

**Interfaces:**

```typescript
interface ISnapshotAssembly {
  createSnapshot(context: AssemblyContext): Promise<Snapshot>;
  validateSnapshot(snapshot: Snapshot): ValidationResult;
  estimateTokens(snapshot: Snapshot): number;
  trimToFit(snapshot: Snapshot, maxTokens: number): Snapshot;
}

interface AssemblyContext {
  objective: string;
  threads: Thread[];
  facts: AtomicFact[];
  constraints: string[];
  decisions: Decision[];
  questions: string[];
  toolState: ToolState;
  lastActions: string[];
  eventRange: { from: number; to: number };
}

interface ValidationResult {
  valid: boolean;
  errors: string[];
  warnings: string[];
  tokenCount: number;
}
```

**Validation Rules:**

```typescript
function validateSnapshot(snapshot: Snapshot): ValidationResult {
  const errors: string[] = [];
  const warnings: string[] = [];
  
  // Required fields
  if (!snapshot.objective_now) errors.push("Missing objective_now");
  if (!snapshot.handshake) errors.push("Missing handshake metadata");
  
  // Token budget
  const tokenCount = estimateTokens(snapshot);
  if (tokenCount > 500) errors.push(`Token count ${tokenCount} exceeds budget 500`);
  if (tokenCount < 300) warnings.push(`Token count ${tokenCount} below target 300`);
  
  // Constraints
  if (snapshot.hard_constraints.length === 0) {
    warnings.push("No hard constraints present");
  }
  
  // Threads
  if (snapshot.active_threads.length === 0) {
    warnings.push("No active threads");
  }
  if (snapshot.active_threads.length > 7) {
    errors.push(`Too many threads: ${snapshot.active_threads.length} (max 7)`);
  }
  
  // Next steps
  const hasNextStep = snapshot.active_threads.some(t => t.next_step);
  if (!hasNextStep) warnings.push("No thread has a defined next_step");
  
  // Schema version
  if (snapshot.handshake.schema_version !== "1.0") {
    errors.push(`Unsupported schema version: ${snapshot.handshake.schema_version}`);
  }
  
  return {
    valid: errors.length === 0,
    errors,
    warnings,
    tokenCount
  };
}
```

**Trimming Strategy:**

```typescript
function trimToFit(snapshot: Snapshot, maxTokens: number): Snapshot {
  let current = estimateTokens(snapshot);
  
  while (current > maxTokens) {
    // Priority order for removal (never remove constraints)
    if (snapshot.open_questions.length > 0) {
      snapshot.open_questions.pop();
    } else if (snapshot.last_actions.length > 3) {
      snapshot.last_actions.shift();
    } else if (snapshot.recent_facts.length > 10) {
      // Remove lowest-scored fact
      snapshot.recent_facts = snapshot.recent_facts
        .sort((a, b) => b.confidence - a.confidence)
        .slice(0, -1);
    } else if (snapshot.decisions.length > 2) {
      snapshot.decisions.pop();
    } else if (snapshot.active_threads.length > 5) {
      // Remove lowest-confidence thread
      snapshot.active_threads = snapshot.active_threads
        .sort((a, b) => b.confidence - a.confidence)
        .slice(0, -1);
    } else {
      // Can't trim further without losing critical data
      break;
    }
    
    current = estimateTokens(snapshot);
  }
  
  return snapshot;
}
```

---

### SQLite Store Component

**Purpose:** Persist and query operational memory with ACID guarantees.

**Interfaces:**

```typescript
interface ISQLiteStore {
  // Initialization
  initialize(): Promise<void>;
  migrate(toVersion: string): Promise<void>;
  
  // Events
  insertEvent(event: TierAEvent): Promise<number>;
  getEvents(filter: EventFilter): Promise<TierAEvent[]>;
  purgeEvents(olderThan: string): Promise<number>;
  
  // Threads
  upsertThread(thread: Thread): Promise<void>;
  getThread(threadId: string): Promise<Thread | null>;
  getActiveThreads(): Promise<Thread[]>;
  updateThreadConfidence(threadId: string, confidence: number): Promise<void>;
  
  // Snapshots
  insertSnapshot(snapshot: Snapshot, threadId: string): Promise<void>;
  getLatestSnapshot(threadId?: string): Promise<Snapshot | null>;
  getSnapshots(threadId: string): Promise<Snapshot[]>;
  purgeOldSnapshots(threadId: string, keepCount: number): Promise<number>;
  
  // Facts
  insertFact(fact: AtomicFact): Promise<void>;
  getFacts(filter: FactFilter): Promise<AtomicFact[]>;
  
  // Decisions
  insertDecision(decision: Decision): Promise<void>;
  getDecisions(filter: DecisionFilter): Promise<Decision[]>;
  
  // Health
  healthCheck(): Promise<HealthStatus>;
  backup(): Promise<string>;
}

interface EventFilter {
  since?: string;
  until?: string;
  eventType?: string;
  domain?: string;
  limit?: number;
}

interface HealthStatus {
  healthy: boolean;
  eventCount: number;
  snapshotCount: number;
  threadCount: number;
  dbSizeMB: number;
  lastWrite: string;
}
```

**Implementation Details:**

1. **Connection Management:**
   - Use better-sqlite3 for synchronous API
   - Single connection (SQLite doesn't support concurrent writes well)
   - WAL mode for better concurrency
   - Busy timeout: 5000ms

2. **Transaction Handling:**
   - Wrap multi-statement operations in transactions
   - Rollback on error
   - Log transaction failures

3. **Retry Logic:**
   ```typescript
   async function withRetry<T>(
     operation: () => Promise<T>,
     maxRetries: number = 3
   ): Promise<T> {
     for (let i = 0; i < maxRetries; i++) {
       try {
         return await operation();
       } catch (error) {
         if (i === maxRetries - 1) throw error;
         await sleep(Math.pow(2, i) * 100); // Exponential backoff
       }
     }
     throw new Error("Retry exhausted");
   }
   ```

4. **Backup Strategy:**
   - On corruption detection: copy DB file to `.backup` suffix
   - Initialize new database
   - Log backup location
   - Attempt to recover data from backup (best effort)

---

### Roll-Up Engine Component

**Purpose:** Consolidate episodic memory into durable, structured Markdown files.

**Interfaces:**

```typescript
interface IRollUpEngine {
  // Trigger roll-up
  triggerRollUp(reason: RollUpReason): Promise<RollUpResult>;
  scheduleRollUp(intervalMinutes: number): void;
  
  // Generation
  generateCanonicalRollUp(context: RollUpContext): Promise<string>;
  writeRollUp(content: string, filename: string): Promise<void>;
  
  // Queries
  getLastRollUp(): Promise<RollUpMetadata | null>;
  getRollUpHistory(limit: number): Promise<RollUpMetadata[]>;
}

type RollUpReason = 'scheduled' | 'idle' | 'manual' | 'expiry_threshold';

interface RollUpContext {
  events: TierAEvent[];
  snapshot: Snapshot;
  threads: Thread[];
  facts: AtomicFact[];
  decisions: Decision[];
  constraints: string[];
}

interface RollUpResult {
  success: boolean;
  filename: string;
  eventCount: number;
  factCount: number;
  decisionCount: number;
  errors: string[];
}

interface RollUpMetadata {
  filename: string;
  created_at: string;
  event_range: { from: number; to: number };
  fact_count: number;
  decision_count: number;
}
```

**Canonical Roll-Up Template:**

```markdown
# Roll-Up: YYYY-MM-DD HH:MM

**Event Range:** #1234 - #5678  
**Generated:** 2026-03-03T14:30:00Z  
**Reason:** scheduled

---

## Hard Constraints

> These are verbatim and must never be paraphrased.

- [Constraint 1 text exactly as stated]
- [Constraint 2 text exactly as stated]

---

## Decisions

### Decision: [What]

- **Why:** [Rationale]
- **Who:** [user|system|operator]
- **When:** [ISO timestamp]
- **How:** [Implementation approach]
- **Revisit If:** [Conditions]
- **Source:** Event #[event_id]

---

## Facts

### [Timestamp] — [Source Type]

[Fact content]

**Source:** Event #[event_id]  
**Tags:** [tag1, tag2]

---

## Thread Transitions

- Thread `[thread_id]`: active → stalled (blocked by: user_input)
- Thread `[thread_id]`: stalled → done

---

## Warnings & Errors

- [Timestamp]: [Error description] (Event #[event_id])

---

## Statistics

- Events processed: 234
- Facts extracted: 18
- Decisions recorded: 3
- Threads updated: 5
```

**Generation Algorithm:**

```typescript
async function generateCanonicalRollUp(context: RollUpContext): Promise<string> {
  const sections: string[] = [];
  
  // Header
  sections.push(generateHeader(context));
  
  // Hard Constraints (verbatim)
  sections.push(generateConstraintsSection(context.constraints));
  
  // Decisions (structured)
  sections.push(generateDecisionsSection(context.decisions));
  
  // Facts (with source traceability)
  sections.push(generateFactsSection(context.facts));
  
  // Thread transitions
  sections.push(generateThreadTransitionsSection(context.threads));
  
  // Warnings & Errors
  const errors = context.events.filter(e => e.event_type === 'error');
  sections.push(generateWarningsSection(errors));
  
  // Statistics
  sections.push(generateStatistics(context));
  
  return sections.join('\n\n---\n\n');
}

function generateConstraintsSection(constraints: string[]): string {
  let section = '## Hard Constraints\n\n';
  section += '> These are verbatim and must never be paraphrased.\n\n';
  constraints.forEach(c => {
    section += `- ${c}\n`;
  });
  return section;
}

function generateDecisionsSection(decisions: Decision[]): string {
  let section = '## Decisions\n\n';
  decisions.forEach(d => {
    section += `### Decision: ${d.what}\n\n`;
    section += `- **Why:** ${d.why}\n`;
    section += `- **Who:** ${d.who}\n`;
    section += `- **When:** ${d.when}\n`;
    section += `- **How:** ${d.how}\n`;
    if (d.revisit_if && d.revisit_if.length > 0) {
      section += `- **Revisit If:** ${d.revisit_if.join(', ')}\n`;
    }
    section += '\n';
  });
  return section;
}

function generateFactsSection(facts: AtomicFact[]): string {
  let section = '## Facts\n\n';
  facts.forEach(f => {
    section += `### ${f.timestamp} — ${f.source_type}\n\n`;
    section += `${f.content}\n\n`;
    section += `**Source:** Event #${f.source_id}\n`;
    if (f.tags.length > 0) {
      section += `**Tags:** ${f.tags.join(', ')}\n`;
    }
    section += '\n';
  });
  return section;
}
```

**Trigger Conditions:**

```typescript
class RollUpScheduler {
  private intervalMinutes: number = 30;
  private idleThresholdMinutes: number = 10;
  private lastEventTime: Date = new Date();
  
  shouldTriggerRollUp(): { trigger: boolean; reason: RollUpReason } {
    const now = new Date();
    const minutesSinceLastEvent = (now.getTime() - this.lastEventTime.getTime()) / 60000;
    
    // Idle detection
    if (minutesSinceLastEvent >= this.idleThresholdMinutes) {
      return { trigger: true, reason: 'idle' };
    }
    
    // Scheduled interval (handled by cron/timer)
    // This is checked externally
    
    return { trigger: false, reason: 'scheduled' };
  }
  
  onEvent(event: TierAEvent): void {
    this.lastEventTime = new Date(event.timestamp);
  }
}
```

**File Naming Convention:**

```typescript
function generateRollUpFilename(timestamp: Date): string {
  const year = timestamp.getFullYear();
  const month = String(timestamp.getMonth() + 1).padStart(2, '0');
  const day = String(timestamp.getDate()).padStart(2, '0');
  const hour = String(timestamp.getHours()).padStart(2, '0');
  const minute = String(timestamp.getMinutes()).padStart(2, '0');
  
  return `memory/rollups/${year}-${month}-${day}-${hour}${minute}.canonical.md`;
}
```

---

### Thread Management Component

**Purpose:** Track work items with status, confidence, and blockers.

**Interfaces:**

```typescript
interface IThreadManager {
  // CRUD operations
  createThread(title: string, nextStep: string): Promise<Thread>;
  updateThread(threadId: string, updates: Partial<Thread>): Promise<void>;
  getThread(threadId: string): Promise<Thread | null>;
  getActiveThreads(): Promise<Thread[]>;
  
  // Confidence management
  updateConfidence(threadId: string, signal: ConfidenceSignal): Promise<void>;
  decayConfidence(threadId: string, hours: number): Promise<void>;
  
  // Status transitions
  markStalled(threadId: string, blockers: BlockerType[]): Promise<void>;
  markDone(threadId: string): Promise<void>;
  reactivate(threadId: string): Promise<void>;
  
  // Queries
  getStalledThreads(): Promise<Thread[]>;
  getLowConfidenceThreads(threshold: number): Promise<Thread[]>;
}

interface ConfidenceSignal {
  type: 'progress' | 'error' | 'stall' | 'completion';
  magnitude: number;  // -1.0 to 1.0
  reason: string;
}
```

**Confidence Update Algorithm:**

```typescript
function updateConfidence(
  currentConfidence: number,
  signal: ConfidenceSignal
): number {
  // Apply signal with dampening
  const dampening = 0.3;  // Prevents wild swings
  const delta = signal.magnitude * dampening;
  
  let newConfidence = currentConfidence + delta;
  
  // Clamp to [0.0, 1.0]
  newConfidence = Math.max(0.0, Math.min(1.0, newConfidence));
  
  return newConfidence;
}

function decayConfidence(
  currentConfidence: number,
  hoursInactive: number
): number {
  // Exponential decay: confidence *= e^(-hours/24)
  // Decays to ~60% after 12 hours, ~37% after 24 hours
  const decayRate = 0.05;  // Per hour
  const decay = Math.exp(-hoursInactive * decayRate);
  
  // Never decay below 0.2 (minimum baseline)
  const decayed = currentConfidence * decay;
  return Math.max(0.2, decayed);
}

// Example signals
const signals = {
  toolSuccess: { type: 'progress', magnitude: 0.1, reason: 'Tool executed successfully' },
  toolError: { type: 'error', magnitude: -0.15, reason: 'Tool execution failed' },
  userInput: { type: 'progress', magnitude: 0.2, reason: 'User provided input' },
  stalled: { type: 'stall', magnitude: -0.3, reason: 'No progress for 30 minutes' },
  completed: { type: 'completion', magnitude: 0.5, reason: 'Thread objective achieved' }
};
```

**Status Transition Rules:**

```typescript
class ThreadStateMachine {
  transition(thread: Thread, event: ThreadEvent): Thread {
    const { status } = thread;
    
    switch (status) {
      case 'active':
        if (event.type === 'stall_detected') {
          return { ...thread, status: 'stalled', blocked_by: event.blockers };
        }
        if (event.type === 'completion') {
          return { ...thread, status: 'done', confidence: 1.0 };
        }
        break;
        
      case 'stalled':
        if (event.type === 'blocker_resolved') {
          return { ...thread, status: 'active', blocked_by: [] };
        }
        if (event.type === 'manual_reactivate') {
          return { ...thread, status: 'active' };
        }
        break;
        
      case 'done':
        // Terminal state, no transitions
        break;
    }
    
    return thread;
  }
}

interface ThreadEvent {
  type: 'stall_detected' | 'blocker_resolved' | 'completion' | 'manual_reactivate';
  blockers?: BlockerType[];
  reason?: string;
}
```

**Stall Detection:**

```typescript
async function detectStalledThreads(
  threads: Thread[],
  events: TierAEvent[]
): Promise<Thread[]> {
  const stalledThreads: Thread[] = [];
  const now = new Date();
  const stallThresholdMinutes = 30;
  
  for (const thread of threads) {
    if (thread.status !== 'active') continue;
    
    const lastActivity = new Date(thread.last_activity_at);
    const minutesSinceActivity = (now.getTime() - lastActivity.getTime()) / 60000;
    
    if (minutesSinceActivity >= stallThresholdMinutes) {
      // Analyze recent events to determine blocker
      const blockers = analyzeBlockers(thread, events);
      stalledThreads.push({ ...thread, blocked_by: blockers });
    }
  }
  
  return stalledThreads;
}

function analyzeBlockers(thread: Thread, events: TierAEvent[]): BlockerType[] {
  const blockers: BlockerType[] = [];
  
  // Check for rate limit errors
  const rateLimitErrors = events.filter(e => 
    e.content.includes('rate limit') || e.content.includes('429')
  );
  if (rateLimitErrors.length > 0) blockers.push('rate_limit');
  
  // Check for missing credentials
  const authErrors = events.filter(e =>
    e.content.includes('unauthorized') || e.content.includes('401')
  );
  if (authErrors.length > 0) blockers.push('missing_credential');
  
  // Check for user input requests
  const userInputRequests = events.filter(e =>
    e.content.includes('waiting for user') || e.event_type === 'user_input_required'
  );
  if (userInputRequests.length > 0) blockers.push('user_input');
  
  // Default to unknown if no specific blocker identified
  if (blockers.length === 0) blockers.push('unknown');
  
  return blockers;
}
```

---

## Integration with OpenClaw

### Plugin Lifecycle

```typescript
class CognitiveControllerPlugin {
  private observer: IObserver;
  private reranker: IReranker;
  private snapshotAssembly: ISnapshotAssembly;
  private sqliteStore: ISQLiteStore;
  private rollUpEngine: IRollUpEngine;
  private threadManager: IThreadManager;
  
  async initialize(openClawAPI: IOpenClawAPI): Promise<void> {
    // 1. Initialize SQLite store
    await this.sqliteStore.initialize();
    
    // 2. Run migrations if needed
    await this.sqliteStore.migrate('1.0');
    
    // 3. Register hooks with OpenClaw
    openClawAPI.onUserMessage(this.observer.onUserMessage.bind(this.observer));
    openClawAPI.onToolResult(this.observer.onToolResult.bind(this.observer));
    openClawAPI.onStateTransition(this.observer.onStateTransition.bind(this.observer));
    
    // 4. Schedule roll-ups
    this.rollUpEngine.scheduleRollUp(30);  // Every 30 minutes
    
    // 5. Start heartbeat monitoring
    this.startHeartbeatMonitoring();
    
    // 6. Expose health check endpoint
    openClawAPI.registerEndpoint('/cognitive-controller/health', this.healthCheck.bind(this));
    
    console.log('Cognitive Controller initialized successfully');
  }
  
  async shutdown(): Promise<void> {
    // Flush any buffered events
    await this.observer.flushBuffer();
    
    // Trigger final roll-up
    await this.rollUpEngine.triggerRollUp('manual');
    
    // Close database connection
    await this.sqliteStore.close();
    
    console.log('Cognitive Controller shutdown complete');
  }
  
  async healthCheck(): Promise<HealthStatus> {
    return await this.sqliteStore.healthCheck();
  }
  
  private startHeartbeatMonitoring(): void {
    setInterval(() => {
      const heartbeat = this.observer.getLastHeartbeat();
      const now = new Date();
      const lastBeat = new Date(heartbeat.timestamp);
      const minutesSinceLastBeat = (now.getTime() - lastBeat.getTime()) / 60000;
      
      if (minutesSinceLastBeat > 5) {
        // Emit stale heartbeat
        this.observer.emitHeartbeat();
      }
    }, 60000);  // Check every minute
  }
}
```

### Hook Registration

```typescript
interface IOpenClawAPI {
  // Event hooks
  onUserMessage(handler: (message: string, metadata: any) => Promise<void>): void;
  onToolResult(handler: (toolName: string, params: any, result: any) => Promise<void>): void;
  onStateTransition(handler: (from: string, to: string, reason: string) => Promise<void>): void;
  
  // Context injection
  injectContext(context: string): void;
  
  // Endpoints
  registerEndpoint(path: string, handler: () => Promise<any>): void;
  
  // UI
  showToast(message: string, type: 'info' | 'warning' | 'error'): void;
  updateLED(status: 'green' | 'amber' | 'red'): void;
}
```

### Context Injection Flow

```typescript
async function injectSnapshotIntoContext(
  openClawAPI: IOpenClawAPI,
  snapshot: Snapshot
): Promise<void> {
  // Format snapshot as context block
  const contextBlock = formatSnapshotForLLM(snapshot);
  
  // Inject into OpenClaw's context
  openClawAPI.injectContext(contextBlock);
}

function formatSnapshotForLLM(snapshot: Snapshot): string {
  return `
# Cognitive Controller Resume Block

## Current Objective
${snapshot.objective_now}

## Active Threads
${snapshot.active_threads.map(t => `
- **${t.title}** (confidence: ${t.confidence.toFixed(2)})
  - Status: ${t.status}
  - Next: ${t.next_step}
  ${t.blocked_by.length > 0 ? `- Blocked by: ${t.blocked_by.join(', ')}` : ''}
`).join('\n')}

## Hard Constraints
${snapshot.hard_constraints.map(c => `- ${c}`).join('\n')}

## Recent Facts
${snapshot.recent_facts.map(f => `- [${f.timestamp}] ${f.content}`).join('\n')}

## Key Decisions
${snapshot.decisions.map(d => `
- **${d.what}**
  - Why: ${d.why}
  - When: ${d.when}
`).join('\n')}

## Tool State
- Available: ${snapshot.tool_state.available.join(', ')}
- Health: ${snapshot.tool_state.health}

## Open Questions
${snapshot.open_questions.map(q => `- ${q}`).join('\n')}
`;
}
```

---

## Error Handling Strategy

### Error Categories

| Category | Severity | Response |
|----------|----------|----------|
| SQLite write failure | High | Buffer + retry, fallback to JSON |
| Snapshot assembly failure | High | Return last valid snapshot, log error |
| Roll-up generation failure | Medium | Log error, retry on next trigger |
| Hook registration failure | Critical | Fail plugin initialization |
| Token budget exceeded | Medium | Trim snapshot, log warning |
| Invalid data in event stream | Low | Skip invalid item, continue |

### Error Handling Patterns

```typescript
// Pattern 1: Retry with exponential backoff
async function withRetry<T>(
  operation: () => Promise<T>,
  maxRetries: number = 3,
  baseDelayMs: number = 100
): Promise<T> {
  for (let i = 0; i < maxRetries; i++) {
    try {
      return await operation();
    } catch (error) {
      if (i === maxRetries - 1) throw error;
      const delay = baseDelayMs * Math.pow(2, i);
      await sleep(delay);
    }
  }
  throw new Error('Retry exhausted');
}

// Pattern 2: Fallback chain
async function writeEventWithFallback(event: TierAEvent): Promise<void> {
  try {
    await sqliteStore.insertEvent(event);
  } catch (sqliteError) {
    try {
      await bufferEvent(event);
    } catch (bufferError) {
      try {
        await writeToFallbackJSON(event);
      } catch (jsonError) {
        console.error('All write methods failed', { sqliteError, bufferError, jsonError });
        // Last resort: log to console
        console.log('LOST_EVENT', JSON.stringify(event));
      }
    }
  }
}

// Pattern 3: Graceful degradation
async function assembleSnapshot(): Promise<Snapshot> {
  try {
    const snapshot = await fullSnapshotAssembly();
    return snapshot;
  } catch (error) {
    console.warn('Full snapshot assembly failed, using minimal snapshot', error);
    return createMinimalSnapshot();
  }
}

function createMinimalSnapshot(): Snapshot {
  return {
    handshake: {
      snapshot_id: generateUUID(),
      created_at: new Date().toISOString(),
      expires_at: new Date(Date.now() + 48 * 3600000).toISOString(),
      source_event_ids: [],
      schema_version: '1.0'
    },
    objective_now: 'Continue current work',
    active_threads: [],
    recent_facts: [],
    hard_constraints: [],
    decisions: [],
    open_questions: [],
    tool_state: { available: [], unavailable: [], health: 'unknown' },
    last_actions: []
  };
}
```

---

## UI Components

### LED Indicator

**States:**
- 🟢 Green: Healthy / Progressing (all threads active, confidence > 0.7)
- 🟠 Amber: Stalled / Needs Attention (threads stalled, confidence 0.3-0.7)
- 🔴 Red: Failure / Escalation (confidence < 0.3, critical errors)

**Update Logic:**

```typescript
function calculateLEDStatus(threads: Thread[], heartbeat: Heartbeat): LEDStatus {
  // Red: Critical errors or very low confidence
  const criticalThreads = threads.filter(t => t.confidence < 0.3);
  if (criticalThreads.length > 0) return 'red';
  
  // Red: Heartbeat stale
  if (heartbeat.status === 'stale') return 'red';
  
  // Amber: Stalled threads
  const stalledThreads = threads.filter(t => t.status === 'stalled');
  if (stalledThreads.length > 0) return 'amber';
  
  // Amber: Low confidence
  const lowConfidenceThreads = threads.filter(t => t.confidence < 0.7);
  if (lowConfidenceThreads.length > threads.length / 2) return 'amber';
  
  // Green: All good
  return 'green';
}
```

### Toast Notifications

**Types:**

```typescript
interface ToastNotification {
  message: string;
  type: 'info' | 'warning' | 'error';
  duration: number;  // milliseconds
  actions?: ToastAction[];
}

interface ToastAction {
  label: string;
  handler: () => void;
}
```

**Notification Triggers:**

```typescript
// Snapshot created
showToast({
  message: `Snapshot created: ${threadCount} threads, ${tokenCount} tokens`,
  type: 'info',
  duration: 3000
});

// Thread stalled
showToast({
  message: `Thread stalled: "${threadTitle}" (blocked by: ${blockers.join(', ')})`,
  type: 'warning',
  duration: 5000,
  actions: [
    { label: 'View Thread', handler: () => openThread(threadId) },
    { label: 'Reactivate', handler: () => reactivateThread(threadId) }
  ]
});

// Low confidence escalation
showToast({
  message: `Thread "${threadTitle}" has low confidence (${confidence.toFixed(2)}). Consider escalation.`,
  type: 'error',
  duration: 0,  // Persistent until dismissed
  actions: [
    { label: 'Escalate', handler: () => escalateThread(threadId) },
    { label: 'Dismiss', handler: () => {} }
  ]
});

// Roll-up completed
showToast({
  message: `Roll-up completed: ${factCount} facts, ${decisionCount} decisions`,
  type: 'info',
  duration: 3000
});
```

### Settings Panel

**Configuration Options:**

```typescript
interface CognitiveControllerSettings {
  // Snapshot
  snapshotTokenBudget: number;          // Default: 500
  maxActiveThreads: number;             // Default: 7
  snapshotExpiryHours: number;          // Default: 48
  
  // Roll-up
  rollUpIntervalMinutes: number;        // Default: 30
  rollUpIdleThresholdMinutes: number;   // Default: 10
  enableAutoRollUp: boolean;            // Default: true
  
  // Confidence
  confidenceDampening: number;          // Default: 0.3
  confidenceDecayRate: number;          // Default: 0.05
  lowConfidenceThreshold: number;       // Default: 0.3
  
  // Thread management
  stallDetectionMinutes: number;        // Default: 30
  
  // UI
  enableToastNotifications: boolean;    // Default: true
  enableLEDIndicator: boolean;          // Default: true
  
  // Storage
  maxSnapshotsPerThread: number;        // Default: 10
  eventRetentionHours: number;          // Default: 48
}
```

---

## Performance Considerations

### Optimization Strategies

1. **Event Capture:**
   - Batch writes when possible (flush buffer every 100ms or 10 events)
   - Use prepared statements for SQLite inserts
   - Index frequently queried columns (timestamp, event_type, thread_id)

2. **Snapshot Assembly:**
   - Cache last snapshot to avoid full rebuild
   - Incremental updates when possible
   - Lazy load facts/decisions (only fetch what's needed)

3. **Roll-Up Generation:**
   - Run in background thread (don't block main event loop)
   - Process events in chunks (1000 at a time)
   - Use streaming writes for large roll-ups

4. **Database Queries:**
   - Use indexes on all foreign keys
   - Limit result sets (default: 1000 rows)
   - Use EXPLAIN QUERY PLAN to optimize slow queries

### Memory Management

```typescript
class MemoryMonitor {
  private maxBufferSize: number = 1000;
  private maxCacheSize: number = 100;
  
  checkMemoryUsage(): MemoryUsage {
    const usage = process.memoryUsage();
    return {
      heapUsedMB: usage.heapUsed / 1024 / 1024,
      heapTotalMB: usage.heapTotal / 1024 / 1024,
      externalMB: usage.external / 1024 / 1024,
      bufferSize: this.getBufferSize(),
      cacheSize: this.getCacheSize()
    };
  }
  
  shouldFlushBuffer(): boolean {
    const usage = this.checkMemoryUsage();
    return usage.heapUsedMB > 80 || this.getBufferSize() > this.maxBufferSize;
  }
  
  shouldClearCache(): boolean {
    const usage = this.checkMemoryUsage();
    return usage.heapUsedMB > 90 || this.getCacheSize() > this.maxCacheSize;
  }
}
```

### Benchmarks (Target)

| Operation | Target Time | Notes |
|-----------|-------------|-------|
| Event capture | < 100ms | Including SQLite write |
| Snapshot assembly | < 500ms | Full rebuild |
| Snapshot assembly (incremental) | < 100ms | Update existing |
| Roll-up generation | < 5s | For 1000 events |
| Thread confidence update | < 50ms | Single thread |
| Database query (indexed) | < 10ms | Typical query |

---

## Testing Strategy

### Unit Tests

**Coverage Targets:**
- Observer: 90%
- Reranker: 85%
- Snapshot Assembly: 90%
- SQLite Store: 80%
- Thread Manager: 85%
- Roll-Up Engine: 80%

**Key Test Cases:**

```typescript
describe('Observer', () => {
  it('should capture user messages with correct metadata');
  it('should assign monotonic event IDs');
  it('should buffer events on write failure');
  it('should emit heartbeat after event capture');
  it('should classify events by domain');
});

describe('Reranker', () => {
  it('should prioritize constraints over other candidates');
  it('should score recent events higher than old events');
  it('should rank active threads higher than stalled threads');
  it('should respect token budget when assembling snapshot');
  it('should validate all constraints are included');
});

describe('Snapshot Assembly', () => {
  it('should create snapshot within 300-500 token budget');
  it('should preserve constraints verbatim');
  it('should include handshake metadata');
  it('should trim low-priority items when over budget');
  it('should validate snapshot schema');
});

describe('Thread Manager', () => {
  it('should update confidence based on signals');
  it('should decay confidence over time');
  it('should detect stalled threads');
  it('should transition thread states correctly');
  it('should identify blockers from events');
});
```

### Integration Tests

```typescript
describe('End-to-End Flow', () => {
  it('should capture event, update thread, and create snapshot', async () => {
    // 1. Capture user message
    await observer.onUserMessage('Test message', {});
    
    // 2. Verify event in database
    const events = await sqliteStore.getEvents({ limit: 1 });
    expect(events).toHaveLength(1);
    
    // 3. Update thread confidence
    await threadManager.updateConfidence(threadId, {
      type: 'progress',
      magnitude: 0.1,
      reason: 'User message received'
    });
    
    // 4. Create snapshot
    const snapshot = await snapshotAssembly.createSnapshot(context);
    
    // 5. Verify snapshot
    expect(snapshot.recent_facts).toContainEqual(
      expect.objectContaining({ content: 'Test message' })
    );
  });
  
  it('should trigger roll-up after idle period', async () => {
    // 1. Generate events
    for (let i = 0; i < 50; i++) {
      await observer.onUserMessage(`Message ${i}`, {});
    }
    
    // 2. Wait for idle threshold
    await sleep(11 * 60 * 1000);  // 11 minutes
    
    // 3. Verify roll-up triggered
    const rollUps = await rollUpEngine.getRollUpHistory(1);
    expect(rollUps).toHaveLength(1);
    expect(rollUps[0].event_count).toBe(50);
  });
});
```

### Performance Tests

```typescript
describe('Performance', () => {
  it('should capture 1000 events in < 10 seconds', async () => {
    const start = Date.now();
    for (let i = 0; i < 1000; i++) {
      await observer.onUserMessage(`Message ${i}`, {});
    }
    const duration = Date.now() - start;
    expect(duration).toBeLessThan(10000);
  });
  
  it('should assemble snapshot in < 500ms', async () => {
    const start = Date.now();
    await snapshotAssembly.createSnapshot(context);
    const duration = Date.now() - start;
    expect(duration).toBeLessThan(500);
  });
});
```

---

## Deployment

### Installation

```bash
# 1. Install plugin in OpenClaw plugins directory
cd ~/.openclaw/plugins
git clone https://github.com/your-org/cognitive-controller.git

# 2. Install dependencies
cd cognitive-controller
npm install

# 3. Build TypeScript
npm run build

# 4. Restart OpenClaw
```

### Directory Structure

```
cognitive-controller/
├── src/
│   ├── components/
│   │   ├── observer.ts
│   │   ├── reranker.ts
│   │   ├── snapshot-assembly.ts
│   │   ├── sqlite-store.ts
│   │   ├── rollup-engine.ts
│   │   └── thread-manager.ts
│   ├── models/
│   │   ├── event.ts
│   │   ├── thread.ts
│   │   ├── snapshot.ts
│   │   ├── fact.ts
│   │   └── decision.ts
│   ├── utils/
│   │   ├── token-estimator.ts
│   │   ├── uuid.ts
│   │   └── retry.ts
│   ├── index.ts
│   └── plugin.ts
├── tests/
│   ├── unit/
│   └── integration/
├── migrations/
│   └── 001_initial_schema.sql
├── memory/
│   └── rollups/
├── package.json
├── tsconfig.json
└── README.md
```

### Configuration File

**Location:** `~/.openclaw/plugins/cognitive-controller/config.json`

```json
{
  "snapshot": {
    "tokenBudget": 500,
    "maxActiveThreads": 7,
    "expiryHours": 48
  },
  "rollup": {
    "intervalMinutes": 30,
    "idleThresholdMinutes": 10,
    "enableAuto": true
  },
  "confidence": {
    "dampening": 0.3,
    "decayRate": 0.05,
    "lowThreshold": 0.3
  },
  "thread": {
    "stallDetectionMinutes": 30
  },
  "ui": {
    "enableToasts": true,
    "enableLED": true
  },
  "storage": {
    "maxSnapshotsPerThread": 10,
    "eventRetentionHours": 48,
    "databasePath": "./cognitive-controller.db"
  }
}
```

### Windows-Specific Considerations

```typescript
// Path handling
import * as path from 'path';

function getPluginDataPath(): string {
  // Use path.join for cross-platform compatibility
  const homeDir = process.env.USERPROFILE || process.env.HOME;
  return path.join(homeDir, '.openclaw', 'plugins', 'cognitive-controller');
}

function getDatabasePath(): string {
  const dataPath = getPluginDataPath();
  return path.join(dataPath, 'cognitive-controller.db');
}

// File operations
import * as fs from 'fs/promises';

async function ensureDirectoryExists(dirPath: string): Promise<void> {
  try {
    await fs.mkdir(dirPath, { recursive: true });
  } catch (error) {
    if (error.code !== 'EEXIST') throw error;
  }
}

// Reserved filename handling
const WINDOWS_RESERVED_NAMES = ['CON', 'PRN', 'AUX', 'NUL', 'COM1', 'LPT1'];

function sanitizeFilename(filename: string): string {
  // Remove invalid characters
  let sanitized = filename.replace(/[<>:"/\\|?*]/g, '_');
  
  // Check for reserved names
  const nameWithoutExt = path.parse(sanitized).name.toUpperCase();
  if (WINDOWS_RESERVED_NAMES.includes(nameWithoutExt)) {
    sanitized = `_${sanitized}`;
  }
  
  return sanitized;
}
```

---

## Migration Path

### Phase 0 → Phase 1 Migration

**Changes:**
- Add `confidence` column to threads table
- Add `blocked_by` column to threads table
- Add `token_count` column to snapshots table
- Create facts table
- Create decisions table

**Migration Script:**

```sql
-- migrations/002_phase1_enhancements.sql

-- Add confidence to threads
ALTER TABLE threads ADD COLUMN confidence REAL NOT NULL DEFAULT 0.5;
CREATE INDEX idx_threads_confidence ON threads(confidence);

-- Add blocked_by to threads
ALTER TABLE threads ADD COLUMN blocked_by TEXT;

-- Add token_count to snapshots
ALTER TABLE snapshots ADD COLUMN token_count INTEGER;

-- Create facts table
CREATE TABLE facts (
  fact_id TEXT PRIMARY KEY,
  timestamp TEXT NOT NULL,
  source_type TEXT NOT NULL,
  source_id TEXT NOT NULL,
  content TEXT NOT NULL,
  tags TEXT,
  confidence REAL NOT NULL DEFAULT 1.0
);

CREATE INDEX idx_facts_timestamp ON facts(timestamp);
CREATE INDEX idx_facts_source ON facts(source_id);

-- Create decisions table
CREATE TABLE decisions (
  decision_id TEXT PRIMARY KEY,
  what TEXT NOT NULL,
  why TEXT NOT NULL,
  who TEXT NOT NULL,
  when TEXT NOT NULL,
  how TEXT NOT NULL,
  revisit_if TEXT
);

CREATE INDEX idx_decisions_when ON decisions(when);
CREATE INDEX idx_decisions_who ON decisions(who);
```

### Data Migration

```typescript
async function migrateToPhase1(db: Database): Promise<void> {
  // 1. Check current schema version
  const version = await db.get('PRAGMA user_version');
  if (version.user_version >= 2) {
    console.log('Already at Phase 1 schema');
    return;
  }
  
  // 2. Run migration script
  const migrationSQL = await fs.readFile('migrations/002_phase1_enhancements.sql', 'utf-8');
  await db.exec(migrationSQL);
  
  // 3. Update schema version
  await db.exec('PRAGMA user_version = 2');
  
  console.log('Migration to Phase 1 complete');
}
```

---

## Security Considerations

### Data Privacy

1. **PII Handling:**
   - Never log sensitive user data
   - Sanitize event content before storage
   - Provide data export/deletion endpoints for GDPR compliance

2. **Access Control:**
   - Plugin data directory should be user-readable only
   - Database file permissions: 600 (owner read/write only)
   - Roll-up files permissions: 600

3. **Input Validation:**
   ```typescript
   function validateEventInput(event: any): TierAEvent {
     // Validate required fields
     if (!event.timestamp || !event.event_type || !event.content) {
       throw new Error('Invalid event: missing required fields');
     }
     
     // Sanitize content
     const sanitized = {
       ...event,
       content: sanitizeContent(event.content),
       metadata: sanitizeMetadata(event.metadata)
     };
     
     return sanitized as TierAEvent;
   }
   
   function sanitizeContent(content: string): string {
     // Remove potential injection attacks
     return content
       .replace(/<script[^>]*>.*?<\/script>/gi, '')
       .replace(/javascript:/gi, '')
       .substring(0, 10000);  // Limit length
   }
   ```

### SQL Injection Prevention

```typescript
// Always use parameterized queries
async function insertEvent(event: TierAEvent): Promise<number> {
  const stmt = db.prepare(`
    INSERT INTO events (timestamp, event_type, domain, source_id, content, metadata, tags)
    VALUES (?, ?, ?, ?, ?, ?, ?)
  `);
  
  const result = stmt.run(
    event.timestamp,
    event.event_type,
    event.domain,
    event.source_id,
    event.content,
    JSON.stringify(event.metadata),
    JSON.stringify(event.tags)
  );
  
  return result.lastInsertRowid as number;
}
```

---

## Monitoring and Observability

### Metrics

```typescript
interface Metrics {
  // Event capture
  eventsCapture: number;
  eventsCaptureErrors: number;
  eventsBuffered: number;
  
  // Snapshots
  snapshotsCreated: number;
  snapshotsCreationErrors: number;
  averageSnapshotTokens: number;
  
  // Threads
  activeThreads: number;
  stalledThreads: number;
  averageConfidence: number;
  
  // Roll-ups
  rollUpsGenerated: number;
  rollUpErrors: number;
  
  // Storage
  databaseSizeMB: number;
  eventCount: number;
  snapshotCount: number;
  
  // Performance
  averageEventCaptureMs: number;
  averageSnapshotAssemblyMs: number;
}

class MetricsCollector {
  private metrics: Metrics;
  
  recordEventCapture(durationMs: number): void {
    this.metrics.eventsCapture++;
    this.updateAverage('averageEventCaptureMs', durationMs);
  }
  
  recordSnapshotCreation(tokenCount: number, durationMs: number): void {
    this.metrics.snapshotsCreated++;
    this.updateAverage('averageSnapshotTokens', tokenCount);
    this.updateAverage('averageSnapshotAssemblyMs', durationMs);
  }
  
  getMetrics(): Metrics {
    return { ...this.metrics };
  }
  
  exportPrometheus(): string {
    // Export in Prometheus format for monitoring systems
    return `
# HELP cognitive_controller_events_captured Total events captured
# TYPE cognitive_controller_events_captured counter
cognitive_controller_events_captured ${this.metrics.eventsCapture}

# HELP cognitive_controller_snapshots_created Total snapshots created
# TYPE cognitive_controller_snapshots_created counter
cognitive_controller_snapshots_created ${this.metrics.snapshotsCreated}

# HELP cognitive_controller_active_threads Current active threads
# TYPE cognitive_controller_active_threads gauge
cognitive_controller_active_threads ${this.metrics.activeThreads}
    `.trim();
  }
}
```

### Logging

```typescript
enum LogLevel {
  DEBUG = 0,
  INFO = 1,
  WARN = 2,
  ERROR = 3
}

class Logger {
  private level: LogLevel = LogLevel.INFO;
  
  debug(message: string, context?: any): void {
    if (this.level <= LogLevel.DEBUG) {
      console.debug(`[DEBUG] ${message}`, context);
    }
  }
  
  info(message: string, context?: any): void {
    if (this.level <= LogLevel.INFO) {
      console.info(`[INFO] ${message}`, context);
    }
  }
  
  warn(message: string, context?: any): void {
    if (this.level <= LogLevel.WARN) {
      console.warn(`[WARN] ${message}`, context);
    }
  }
  
  error(message: string, error?: Error, context?: any): void {
    if (this.level <= LogLevel.ERROR) {
      console.error(`[ERROR] ${message}`, { error, context });
    }
  }
}
```

---

## Future Enhancements (Out of Scope for Phase 0/1)

### Phase 2: Elastic Resolver

- Fail-fast triage policies
- Tactic swap on repeated failures
- Model escalation based on confidence
- Automatic retry with different approaches

### Phase 3: Semantic Memory

- Vector embeddings for facts and decisions
- Semantic search across roll-ups
- Similarity-based context retrieval
- Integration with external vector databases

### Phase 4: Multi-Agent Coordination

- Shared memory across multiple agents
- Agent-to-agent communication via events
- Distributed thread management
- Consensus-based decision making

### Phase 5: External Integrations

- Obsidian vault indexing
- Notion database sync
- GitHub issue tracking integration
- Slack notifications for critical events

---

## Conclusion

The Cognitive Controller provides a robust foundation for managing long-term context in OpenClaw agents. By combining structured snapshots, bounded memory, and intelligent reranking, it prevents context window bloat while maintaining high-signal working memory.

Phase 0 and Phase 1 establish the core infrastructure and memory management capabilities. Future phases will build on this foundation to add elastic resolution, semantic memory, and external integrations.

The design prioritizes:
- **Stability:** Structured data prevents drift
- **Bounded complexity:** Fixed token budgets and snapshot limits
- **Recoverability:** Multiple recovery points per thread
- **Operator trust:** Transparent health signals and audit trails

---

**Document Owner:** Project Team  
**Review Status:** Draft  
**Next Steps:** Begin Phase 0 implementation
