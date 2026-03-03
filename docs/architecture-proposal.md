# Cognitive Controller: Architecture Proposal

**Date:** 2026-03-03  
**Status:** Proposal for Review  
**Pattern:** Hexagonal Architecture + Event-Driven Loop

---

## Architectural Vision

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
│  Adapters → Application → Domain → Infrastructure            │
└─────────────────────────────────────────────────────────────┘
```

---

## Proposed Architecture: Hexagonal + Event-Driven

### Core Principles

1. **Domain at the Center** - Pure business logic, no dependencies
2. **Ports Define Contracts** - Interfaces for all external interactions
3. **Adapters Implement Ports** - OpenClaw, SQLite, File System adapters
4. **Event Loop Orchestrates** - Horizontal flow through vertical layers
5. **Dependency Inversion** - All dependencies point inward

---

## Layer Structure

### Layer 1: Domain (Core)

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
    
    // Create immutable snapshot
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

---

### Layer 2: Application (Use Cases)

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

---

### Layer 3: Infrastructure (Adapters)

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

---

### Layer 4: Presentation (Event Loop)

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

---

## Dependency Injection Container

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

---

## Plugin Entry Point

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

---

## Directory Structure

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

---

## Key Design Patterns

### 1. Singleton Pattern
**Where:** EventLoop, DIContainer  
**Why:** Single instance per plugin, global coordination

### 2. Factory Pattern
**Where:** DIContainer, Snapshot.create()  
**Why:** Controlled object creation with validation

### 3. Repository Pattern
**Where:** SnapshotRepository, EventRepository  
**Why:** Abstract data access, testable

### 4. Adapter Pattern
**Where:** All infrastructure adapters  
**Why:** Decouple from external systems

### 5. Observer Pattern
**Where:** Event handlers  
**Why:** React to OpenClaw events

### 6. Result Pattern (Railway-Oriented Programming)
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

---

## Benefits of This Architecture

### 1. Testability
- Domain logic pure functions (easy to test)
- Ports allow mocking external systems
- Use cases testable in isolation

### 2. Maintainability
- Clear separation of concerns
- Dependencies point inward
- Easy to locate and modify code

### 3. Flexibility
- Swap adapters without changing domain
- Add new use cases easily
- Extend without modifying existing code

### 4. OpenClaw Integration
- Adapter isolates OpenClaw specifics
- Easy to update when OpenClaw changes
- Can mock for testing

### 5. Per-Agent Isolation
- DIContainer can create per-agent instances
- Thread-safe by design
- Clear boundaries

---

## Trade-offs

### Pros
✅ Clean, maintainable architecture  
✅ Highly testable  
✅ Flexible and extensible  
✅ Clear separation of concerns  
✅ Follows SOLID principles  

### Cons
⚠️ More boilerplate initially  
⚠️ Steeper learning curve  
⚠️ More files and abstractions  

### Verdict
**Worth it for a 4-6 week project.** The upfront investment in clean architecture will pay off during implementation and maintenance.

---

## Next Steps

1. **Review this proposal** - Does this match your vision?
2. **Refine if needed** - Adjust patterns or structure
3. **Update design.md** - Incorporate this architecture
4. **Create tasks.md** - Break down implementation by layer
5. **Start with Domain** - Build from the inside out

---

**Questions for You:**

1. Does the horizontal (event loop) + vertical (layers) model match your vision?
2. Are you comfortable with the Result pattern for error handling?
3. Should we use a DI framework (like InversifyJS) or keep the simple DIContainer?
4. Any specific patterns you want to add or remove?

