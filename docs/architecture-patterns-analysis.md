# Architecture Patterns Analysis: Fitting into OpenClaw

**Date:** 2026-03-03  
**Purpose:** Identify patterns that match OpenClaw's Gateway-centric architecture  
**Status:** Analysis

---

## OpenClaw's Actual Architecture

Based on research, OpenClaw uses:

```
User/Telegram → Gateway (Central Hub) → LLM
                    ↓
            [Plugins hook here]
                    ↓
        Events flow through middleware
```

**Key Characteristics:**
1. **Gateway is the center** - All events flow through it
2. **Middleware/Pipeline model** - Events pass through transforms
3. **Plugin hooks** - Plugins attach to event stream
4. **WebSocket API** - Real-time event emission
5. **No hot reload** - Gateway restart required for plugin changes

---

## Pattern Analysis: What Fits OpenClaw?

### ❌ What DOESN'T Fit

**Hexagonal Architecture (Ports & Adapters)**
- Too abstract for a plugin that hooks into an existing pipeline
- Assumes we control the application core (we don't - Gateway does)
- Over-engineered for a plugin that's essentially an event listener

**Microservices**
- OpenClaw is monolithic (Gateway-centric)
- Plugins run in-process, not as separate services
- No service mesh or inter-service communication

### ✅ What DOES Fit

#### 1. **Middleware/Pipeline Pattern** ⭐ BEST FIT

**Why it fits:**
- OpenClaw already uses this (`setupMiddleware()` in Gateway)
- Events flow through a chain of handlers
- Each handler can transform, observe, or react to events
- Non-blocking, composable, testable

**How we use it:**

```typescript
// Cognitive Controller as middleware in the event pipeline
class CognitiveControllerMiddleware {
  async handle(event: GatewayEvent, next: NextFunction): Promise<void> {
    // 1. Observe event
    await this.observer.capture(event);
    
    // 2. Update state
    await this.updateThreads(event);
    
    // 3. Check if snapshot needed
    if (this.shouldCreateSnapshot(event)) {
      await this.createSnapshot(event);
    }
    
    // 4. Pass to next middleware
    await next();
  }
}
```

**Benefits:**
- Matches OpenClaw's architecture exactly
- Easy to understand (linear flow)
- Non-blocking (async/await)
- Can be disabled without breaking Gateway

---

#### 2. **Observer Pattern** ⭐ ESSENTIAL

**Why it fits:**
- We're literally observing Gateway events
- Passive - doesn't block the main flow
- Can have multiple observers for different concerns

**How we use it:**

```typescript
// Subscribe to Gateway events
gateway.on('agent.message', (event) => observer.onMessage(event));
gateway.on('agent.tool_result', (event) => observer.onToolResult(event));
gateway.on('agent.memory_flush', (event) => observer.onMemoryFlush(event));

// Observer captures and persists
class EventObserver {
  async onMessage(event: MessageEvent): Promise<void> {
    const tierAEvent = this.transform(event);
    await this.persist(tierAEvent);
    this.emit('event_captured', tierAEvent);
  }
}
```

**Benefits:**
- Decouples us from Gateway internals
- Can observe without modifying Gateway
- Easy to add/remove observers

---

#### 3. **Event Sourcing** ⭐ CORE PATTERN

**Why it fits:**
- Gateway emits events (we don't control when)
- We need to reconstruct state from event stream
- Append-only log matches our SQLite design

**How we use it:**

```typescript
// Events are the source of truth
class EventStore {
  async append(event: TierAEvent): Promise<void> {
    // Append-only, never update
    await this.db.run(
      'INSERT INTO events (timestamp, type, content) VALUES (?, ?, ?)',
      [event.timestamp, event.type, event.content]
    );
  }
  
  async replay(fromEventId: number): Promise<Snapshot> {
    // Rebuild state by replaying events
    const events = await this.getEvents({ since: fromEventId });
    return this.reranker.buildSnapshot(events);
  }
}
```

**Benefits:**
- Perfect audit trail
- Can rebuild state at any point
- Matches Gateway's event-driven nature

---

#### 4. **Pub/Sub (Event Bus)** ⭐ INTEGRATION PATTERN

**Why it fits:**
- Gateway publishes events via WebSocket
- We subscribe to specific event types
- Decoupled, scalable, testable

**How we use it:**

```typescript
// Subscribe to specific event types
class EventBus {
  constructor(private gateway: Gateway) {
    this.subscribe('agent.message', this.handleMessage);
    this.subscribe('agent.tool_result', this.handleToolResult);
    this.subscribe('agent.memory_flush', this.handleMemoryFlush);
  }
  
  private subscribe(eventType: string, handler: EventHandler): void {
    this.gateway.on(eventType, async (event) => {
      try {
        await handler(event);
      } catch (error) {
        logger.error(`Handler failed for ${eventType}`, error);
        // Don't crash Gateway
      }
    });
  }
}
```

**Benefits:**
- Matches Gateway's WebSocket API
- Can subscribe/unsubscribe dynamically
- Isolated error handling

---

#### 5. **Strategy Pattern** (for Reranking)

**Why it fits:**
- Different scoring strategies for different contexts
- Can swap strategies without changing core logic
- Testable in isolation

**How we use it:**

```typescript
interface ScoringStrategy {
  score(candidate: Candidate, context: Context): number;
}

class RiskFirstStrategy implements ScoringStrategy {
  score(candidate: Candidate, context: Context): number {
    return candidate.isConstraint ? 1.0 : 0.0;
  }
}

class RecencyStrategy implements ScoringStrategy {
  score(candidate: Candidate, context: Context): number {
    const ageMinutes = getAge(candidate.timestamp);
    return Math.exp(-ageMinutes / 60);
  }
}

class Reranker {
  constructor(private strategies: ScoringStrategy[]) {}
  
  rank(candidates: Candidate[], context: Context): RankedCandidate[] {
    return candidates.map(c => ({
      ...c,
      score: this.strategies.reduce((sum, s) => sum + s.score(c, context), 0)
    })).sort((a, b) => b.score - a.score);
  }
}
```

---

#### 6. **Repository Pattern** (for Data Access)

**Why it fits:**
- Abstracts SQLite details
- Testable (can mock)
- Single responsibility

**How we use it:**

```typescript
interface EventRepository {
  append(event: TierAEvent): Promise<void>;
  findSince(timestamp: Date): Promise<TierAEvent[]>;
  findByThread(threadId: string): Promise<TierAEvent[]>;
}

class SQLiteEventRepository implements EventRepository {
  constructor(private db: Database) {}
  
  async append(event: TierAEvent): Promise<void> {
    await this.db.run(
      'INSERT INTO events (...) VALUES (...)',
      [event.timestamp, event.type, event.content]
    );
  }
}
```

---

#### 7. **Singleton Pattern** (for Plugin Instance)

**Why it fits:**
- Gateway loads plugin once
- Need single SQLite connection
- Global coordination point

**How we use it:**

```typescript
class CognitiveController {
  private static instance: CognitiveController;
  
  private constructor(private gateway: Gateway) {
    this.initializeDatabase();
    this.registerEventHandlers();
  }
  
  static initialize(gateway: Gateway): CognitiveController {
    if (!CognitiveController.instance) {
      CognitiveController.instance = new CognitiveController(gateway);
    }
    return CognitiveController.instance;
  }
}

// Plugin entry point
export async function initialize(gateway: Gateway): Promise<void> {
  CognitiveController.initialize(gateway);
}
```

---

## Recommended Architecture: Event-Driven Plugin

### Core Pattern: **Middleware + Observer + Event Sourcing**

```
┌─────────────────────────────────────────────────────────────┐
│                    OpenClaw Gateway                          │
│                   (Event Pipeline)                           │
└────────────────────┬────────────────────────────────────────┘
                     │
                     ▼
         ┌───────────────────────┐
         │  Event Bus (Pub/Sub)  │
         └───────────┬───────────┘
                     │
        ┌────────────┼────────────┐
        │            │            │
        ▼            ▼            ▼
   ┌────────┐  ┌────────┐  ┌────────┐
   │Message │  │  Tool  │  │ Memory │
   │Handler │  │ Result │  │ Flush  │
   │        │  │Handler │  │Handler │
   └────┬───┘  └────┬───┘  └────┬───┘
        │           │           │
        └───────────┼───────────┘
                    ▼
         ┌──────────────────────┐
         │  Event Observer      │
         │  (Capture & Persist) │
         └──────────┬───────────┘
                    ▼
         ┌──────────────────────┐
         │  Event Store         │
         │  (SQLite Append-Only)│
         └──────────┬───────────┘
                    │
        ┌───────────┼───────────┐
        │           │           │
        ▼           ▼           ▼
   ┌────────┐  ┌────────┐  ┌────────┐
   │Reranker│  │Snapshot│  │Roll-Up │
   │        │  │Assembly│  │Engine  │
   └────────┘  └────────┘  └────────┘
```

### Directory Structure (Simplified)

```
src/
├── handlers/              # Event handlers (middleware)
│   ├── message.handler.ts
│   ├── tool-result.handler.ts
│   └── memory-flush.handler.ts
│
├── observers/             # Event observers
│   └── event.observer.ts
│
├── repositories/          # Data access
│   ├── event.repository.ts
│   ├── snapshot.repository.ts
│   └── thread.repository.ts
│
├── services/              # Business logic
│   ├── reranker.service.ts
│   ├── snapshot.service.ts
│   └── rollup.service.ts
│
├── models/                # Data models
│   ├── event.ts
│   ├── snapshot.ts
│   └── thread.ts
│
├── strategies/            # Scoring strategies
│   ├── risk.strategy.ts
│   ├── recency.strategy.ts
│   └── relevance.strategy.ts
│
└── index.ts               # Plugin entry point
```

---

## Key Differences from Original Proposal

### Original (Hexagonal)
- 4 layers (Domain, Application, Infrastructure, Presentation)
- Ports & Adapters everywhere
- Dependency Injection Container
- Complex abstractions

### Revised (Event-Driven Plugin)
- 3 layers (Handlers, Services, Repositories)
- Direct Gateway integration
- Simple Singleton for coordination
- Minimal abstractions

### Why the Change?

**Original was too abstract:**
- We're not building a standalone application
- We're a plugin hooking into Gateway's event stream
- Gateway controls the flow, not us

**Revised matches reality:**
- Gateway emits events → We observe
- We don't control when events arrive
- We react, we don't orchestrate

---

## Implementation Strategy

### Phase 0: Foundation

**Pattern:** Observer + Repository

```typescript
// 1. Observe Gateway events
class EventObserver {
  constructor(
    private gateway: Gateway,
    private eventRepo: EventRepository
  ) {
    this.subscribe();
  }
  
  private subscribe(): void {
    this.gateway.on('agent.message', this.onMessage.bind(this));
    this.gateway.on('agent.tool_result', this.onToolResult.bind(this));
  }
  
  private async onMessage(event: MessageEvent): Promise<void> {
    const tierAEvent = this.transform(event);
    await this.eventRepo.append(tierAEvent);
  }
}

// 2. Persist to SQLite
class SQLiteEventRepository implements EventRepository {
  async append(event: TierAEvent): Promise<void> {
    await this.db.run(
      'INSERT INTO events (...) VALUES (...)',
      [event.timestamp, event.type, event.content]
    );
  }
}
```

### Phase 1: Working Memory

**Pattern:** Event Sourcing + Strategy

```typescript
// 1. Replay events to build snapshot
class SnapshotService {
  async createSnapshot(agentId: string): Promise<Snapshot> {
    const events = await this.eventRepo.findSince(lastSnapshotTime);
    const candidates = this.extractCandidates(events);
    const ranked = this.reranker.rank(candidates);
    return this.assemble(ranked);
  }
}

// 2. Use strategies for scoring
class Reranker {
  constructor(private strategies: ScoringStrategy[]) {}
  
  rank(candidates: Candidate[]): RankedCandidate[] {
    return candidates.map(c => ({
      ...c,
      score: this.strategies.reduce((sum, s) => sum + s.score(c), 0)
    })).sort((a, b) => b.score - a.score);
  }
}
```

---

## Comparison Table

| Pattern | Original Proposal | Revised Proposal | Reason for Change |
|---------|------------------|------------------|-------------------|
| Core Architecture | Hexagonal (Ports & Adapters) | Event-Driven Plugin | Match Gateway's pipeline model |
| Layers | 4 (Domain, App, Infra, Presentation) | 3 (Handlers, Services, Repos) | Simpler, less abstraction |
| Dependency Injection | Full DI Container | Simple Singleton | Gateway controls lifecycle |
| Event Handling | EventLoop orchestrates | Handlers react to Gateway | We don't control event flow |
| Error Handling | Result<T, E> monad | Try/catch with fallbacks | Simpler, more idiomatic |
| Testing | Mock ports | Mock Gateway events | Test what we actually integrate with |

---

## Final Recommendation

**Use: Middleware + Observer + Event Sourcing + Repository**

**Why:**
1. **Middleware** - Matches Gateway's architecture
2. **Observer** - We're literally observing events
3. **Event Sourcing** - Append-only log, rebuild state
4. **Repository** - Abstract data access, testable

**Don't use:**
- Hexagonal Architecture (too abstract for a plugin)
- Microservices (we're in-process)
- Complex DI (Gateway controls lifecycle)

**Keep:**
- Singleton (single plugin instance)
- Strategy (scoring algorithms)
- Repository (data access)

---

## Practical Integration: Extension Point Pattern

### How We Actually Hook In

Based on previous implementation experience:

**1. Event Hooks - Append to Existing Event Chain**

```typescript
// OpenClaw emits events → We hook onto the end
gateway.on('agent.message', async (event) => {
  // OpenClaw's handlers run first
  // Then our handler runs
  await cognitiveController.handleMessage(event);
});

gateway.on('agent.tool_result', async (event) => {
  await cognitiveController.handleToolResult(event);
});

// We don't replace, we extend
```

**2. UI Integration - Add to Left Sidebar Menu**

```typescript
// OpenClaw has left sidebar menu (WordPress-style)
// We just add our own item at the end

gateway.registerMenuItem({
  id: 'cognitive-controller',
  label: 'Memory Controller',
  icon: 'brain',
  position: 'bottom', // Append to end of menu
  component: CognitiveControllerPanel
});

// Our panel shows:
// - Control panel with tabs
// - Thread status
// - Snapshot viewer
// - Roll-up history
// - Health LED
```

**3. UI Panel Structure**

```
┌─────────────────────────────────────────┐
│  Cognitive Controller                   │
├─────────────────────────────────────────┤
│  [Threads] [Snapshots] [Roll-Ups] [⚙️]  │  ← Tabs
├─────────────────────────────────────────┤
│                                         │
│  Active Threads (3)                     │
│  ┌───────────────────────────────────┐ │
│  │ 🟢 API Design (0.85)              │ │
│  │    Next: Review endpoint schema   │ │
│  └───────────────────────────────────┘ │
│  ┌───────────────────────────────────┐ │
│  │ 🟠 Database Migration (0.45)      │ │
│  │    Blocked by: user_input         │ │
│  └───────────────────────────────────┘ │
│                                         │
│  Latest Snapshot: 2 mins ago           │
│  Token count: 487 / 500                │
│                                         │
└─────────────────────────────────────────┘
```

**4. Integration Points Summary**

| OpenClaw Feature | How We Hook In | What We Add |
|------------------|----------------|-------------|
| Event Stream | Append handlers to existing events | Capture & persist to SQLite |
| Left Sidebar | Add menu item at bottom | Memory Controller panel |
| Web Control UI | Register custom panel | Thread/snapshot viewer |
| Health Checks | Register health endpoint | LED status, confidence scores |
| Memory Directory | Write to existing memory/ folder | Roll-up files (auto-indexed) |
| Settings | Extend config schema | Our plugin settings |

**5. Non-Invasive Integration**

```typescript
// We don't modify OpenClaw's code
// We use their extension points

export async function initialize(gateway: Gateway): Promise<void> {
  // 1. Hook into events (append, don't replace)
  registerEventHandlers(gateway);
  
  // 2. Add UI components (extend, don't modify)
  registerUIComponents(gateway);
  
  // 3. Register health checks (add to existing)
  registerHealthChecks(gateway);
  
  // 4. Write to memory directory (use existing structure)
  setupMemoryIntegration(gateway);
  
  // OpenClaw continues working normally
  // We're just an observer + extension
}
```

**6. Graceful Degradation**

```typescript
// If our plugin fails, OpenClaw keeps working
try {
  await cognitiveController.handleEvent(event);
} catch (error) {
  logger.error('Cognitive Controller error', error);
  // Don't throw - let OpenClaw continue
  // Show error in our UI panel only
}
```

---

## Interface Requirements

### What OpenClaw Must Provide

Based on integration needs:

1. **Event Hooks**
   - `gateway.on(eventType, handler)` - Subscribe to events
   - Event types: `agent.message`, `agent.tool_result`, `agent.memory_flush`
   - Handlers run after OpenClaw's internal handlers

2. **UI Extension Points**
   - `gateway.registerMenuItem(config)` - Add sidebar menu item
   - `gateway.registerPanel(component)` - Add custom panel
   - Panel receives Gateway context (agentId, sessionId)

3. **Health Check Registration**
   - `gateway.registerHealthCheck(name, checkFn)` - Add health endpoint
   - Exposed via Web Control UI and `/health` API

4. **File System Access**
   - Read/write to agent workspace directory
   - Standard paths: `memory/`, `skills/`, `transcripts/`
   - Our files: `memory/rollups/*.md`, `cognitive-controller.db`

5. **Configuration Extension**
   - Plugin settings in `openclaw.json` or separate config file
   - Hot reload not required (Gateway restart acceptable)

### What We Provide to OpenClaw

1. **Event Handlers** - Non-blocking, error-isolated
2. **UI Components** - Self-contained React/Vue components
3. **Health Endpoint** - Returns LED status, thread count, confidence
4. **Memory Files** - Roll-ups in standard Markdown format
5. **Graceful Failure** - Never crash Gateway

---

## Revised Architecture: Plugin Extension Model

### Core Pattern: **Observer + Extension Points**

```
┌─────────────────────────────────────────────────────────────┐
│                    OpenClaw Gateway                          │
│                                                              │
│  ┌────────────┐  ┌────────────┐  ┌────────────┐           │
│  │   Events   │  │  Sidebar   │  │   Health   │           │
│  │   Stream   │  │    Menu    │  │   Checks   │           │
│  └──────┬─────┘  └──────┬─────┘  └──────┬─────┘           │
│         │                │                │                  │
│         │ Extension      │ Extension      │ Extension        │
│         │ Points         │ Points         │ Points           │
└─────────┼────────────────┼────────────────┼─────────────────┘
          │                │                │
          ▼                ▼                ▼
┌─────────────────────────────────────────────────────────────┐
│              Cognitive Controller Plugin                     │
│                                                              │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐     │
│  │    Event     │  │      UI      │  │    Health    │     │
│  │   Handlers   │  │  Components  │  │   Monitor    │     │
│  └──────┬───────┘  └──────┬───────┘  └──────┬───────┘     │
│         │                  │                  │              │
│         └──────────────────┼──────────────────┘              │
│                            ▼                                 │
│                   ┌────────────────┐                        │
│                   │  Core Services │                        │
│                   │  (Observer,    │                        │
│                   │   Reranker,    │                        │
│                   │   Snapshot)    │                        │
│                   └────────┬───────┘                        │
│                            ▼                                 │
│                   ┌────────────────┐                        │
│                   │ SQLite + Files │                        │
│                   └────────────────┘                        │
└─────────────────────────────────────────────────────────────┘
```

---

## Implementation Notes

### UI Panel Tabs

**Tab 1: Threads**
- List of active/stalled/done threads
- Confidence bars (visual)
- Next step for each thread
- Blocker indicators

**Tab 2: Snapshots**
- Latest snapshot viewer
- Token count gauge
- Field breakdown (threads, facts, constraints)
- History (last 10 snapshots)

**Tab 3: Roll-Ups**
- List of generated roll-ups
- Preview/download
- Statistics (facts, decisions, events)

**Tab 4: Settings ⚙️**
- Token budget slider
- Roll-up interval
- Confidence thresholds
- Enable/disable features

### Control Panel Features

```typescript
interface ControlPanelProps {
  agentId: string;
  sessionId: string;
  gateway: Gateway;
}

class CognitiveControllerPanel extends React.Component<ControlPanelProps> {
  render() {
    return (
      <div className="cognitive-controller-panel">
        <Tabs>
          <Tab label="Threads">
            <ThreadList threads={this.state.threads} />
          </Tab>
          <Tab label="Snapshots">
            <SnapshotViewer snapshot={this.state.latestSnapshot} />
          </Tab>
          <Tab label="Roll-Ups">
            <RollUpHistory rollups={this.state.rollups} />
          </Tab>
          <Tab label="Settings">
            <SettingsPanel config={this.state.config} />
          </Tab>
        </Tabs>
      </div>
    );
  }
}
```

---

## Next Steps

1. Document OpenClaw's extension point APIs (if available)
2. Create UI mockups for control panel
3. Design event handler registration flow
4. Plan graceful degradation scenarios
5. Update design.md with extension point integration

---

**Key Takeaway:**

We're not building a separate architecture - we're **extending OpenClaw's existing architecture** through well-defined extension points. Hook onto events, add to UI, write to memory directory. Simple, non-invasive, looks native.

