# OpenClaw Research Findings: Error Handling & Performance (Q48-Q56)

**Date:** 2026-03-03  
**Source:** OpenClaw Documentation Database  
**Status:** Q48-Q56 Answered

---

## Error Handling & Reliability (Q48-Q52)

### Q48: Error Taxonomy

**Answer:**

OpenClaw categorizes errors based on context:

**RPC Errors:**
- Control-plane RPCs return status codes (e.g., `UNAVAILABLE`)
- Include metadata like `retryAfterMs` for rate limits

**Network Errors:**
- Recoverable issues flagged for retry
- Examples: `TypeError: fetch failed`, `Network request for 'getUpdates' failed!`

**Tool Execution Errors:**
- Specific error when message enqueued during execution: `"Skipped due to queued user message"`
- Graceful degradation: `memory_get` returns `{ text: "", path }` instead of `ENOENT`

**Auth State Errors:**
- Cooldowns: Transient issues (rate limits, timeouts)
- Disables: Terminal issues (billing/credit failures)

### Implications for Cognitive Controller

✅ **Structured Error Handling:**
- Can categorize errors by type
- Different recovery strategies per category

✅ **Graceful Degradation:**
- Follow OpenClaw pattern: return empty structures instead of throwing
- Prevents workflow crashes

⚠️ **No Global Taxonomy:**
- Must build our own error classification
- Map OpenClaw errors to our categories


### Action Items

1. **Implement error taxonomy:**

```typescript
enum ErrorCategory {
  RPC = 'rpc',
  Network = 'network',
  ToolExecution = 'tool_execution',
  Auth = 'auth',
  Storage = 'storage',
  Unknown = 'unknown'
}

enum ErrorSeverity {
  Transient = 'transient',    // Retry possible
  Terminal = 'terminal',       // No retry
  Degraded = 'degraded'        // Partial functionality
}

interface CategorizedError {
  category: ErrorCategory;
  severity: ErrorSeverity;
  originalError: Error;
  retryable: boolean;
  retryAfterMs?: number;
  context: Record<string, any>;
}

class ErrorClassifier {
  classify(error: Error, context: any): CategorizedError {
    // RPC errors
    if (error.message.includes('UNAVAILABLE')) {
      return {
        category: ErrorCategory.RPC,
        severity: ErrorSeverity.Transient,
        originalError: error,
        retryable: true,
        retryAfterMs: this.extractRetryAfter(error),
        context
      };
    }
    
    // Network errors
    if (error.message.includes('fetch failed') || error.message.includes('Network request')) {
      return {
        category: ErrorCategory.Network,
        severity: ErrorSeverity.Transient,
        originalError: error,
        retryable: true,
        context
      };
    }
    
    // Tool execution errors
    if (error.message.includes('Skipped due to queued user message')) {
      return {
        category: ErrorCategory.ToolExecution,
        severity: ErrorSeverity.Degraded,
        originalError: error,
        retryable: false,
        context
      };
    }
    
    // Auth errors
    if (error.message.includes('rate limit') || error.message.includes('billing')) {
      const isTerminal = error.message.includes('billing') || error.message.includes('credit');
      return {
        category: ErrorCategory.Auth,
        severity: isTerminal ? ErrorSeverity.Terminal : ErrorSeverity.Transient,
        originalError: error,
        retryable: !isTerminal,
        context
      };
    }
    
    // Unknown
    return {
      category: ErrorCategory.Unknown,
      severity: ErrorSeverity.Terminal,
      originalError: error,
      retryable: false,
      context
    };
  }
  
  private extractRetryAfter(error: Error): number | undefined {
    const match = error.message.match(/retryAfterMs[:\s]+(\d+)/);
    return match ? parseInt(match[1]) : undefined;
  }
}

const errorClassifier = new ErrorClassifier();
```

2. **Implement graceful degradation:**

```typescript
class GracefulCognitiveController {
  async getSnapshot(agentId: string): Promise<Snapshot | null> {
    try {
      const snapshot = await sqliteStore.getLatestSnapshot(agentId);
      return snapshot;
    } catch (error) {
      const classified = errorClassifier.classify(error as Error, { agentId });
      
      if (classified.severity === ErrorSeverity.Transient) {
        logger.warn(`Transient error getting snapshot, returning empty: ${error.message}`);
        return this.createEmptySnapshot(agentId);
      }
      
      logger.error(`Terminal error getting snapshot: ${error.message}`);
      throw error;
    }
  }
  
  private createEmptySnapshot(agentId: string): Snapshot {
    return {
      handshake: {
        snapshot_id: generateUUID(),
        created_at: new Date().toISOString(),
        expires_at: new Date(Date.now() + 48 * 3600000).toISOString(),
        source_event_ids: [],
        schema_version: '1.0'
      },
      objective_now: 'No objective (degraded mode)',
      active_threads: [],
      recent_facts: [],
      hard_constraints: [],
      decisions: [],
      open_questions: [],
      tool_state: { available: [], unavailable: [], health: 'degraded' },
      last_actions: []
    };
  }
}
```

---

### Q49: Handling Transient Errors

**Answer:**

**Automatic Retry:**
- Cron jobs use exponential backoff: 30s, 1m, 5m, 15m, 60m
- Resets upon successful run

**Retry Limits:**
- Subagent announcements: 3 retries, force-expire after 5 minutes

**Idempotency:**
- Gateway requires idempotency keys for side-effecting methods (`send`, `agent`)
- Allows safe retries without duplication

**Channel Customization:**
- Telegram and other channels have configurable retry policies
- Parameters: attempts, min/max delay, jitter

### Implications for Cognitive Controller

✅ **Retry Infrastructure:**
- Can leverage OpenClaw's retry patterns
- Exponential backoff is standard

✅ **Idempotency Required:**
- Must implement idempotency keys for snapshot operations
- Prevents duplicate snapshots on retry

### Action Items

1. **Implement retry with exponential backoff:**

```typescript
class RetryManager {
  private readonly backoffIntervals = [30000, 60000, 300000, 900000, 3600000]; // 30s, 1m, 5m, 15m, 60m
  
  async withRetry<T>(
    operation: () => Promise<T>,
    options: {
      maxAttempts?: number;
      idempotencyKey?: string;
      context?: any;
    } = {}
  ): Promise<T> {
    const maxAttempts = options.maxAttempts || 5;
    let lastError: Error | null = null;
    
    for (let attempt = 0; attempt < maxAttempts; attempt++) {
      try {
        return await operation();
      } catch (error) {
        lastError = error as Error;
        const classified = errorClassifier.classify(lastError, options.context);
        
        if (!classified.retryable) {
          throw error;
        }
        
        if (attempt < maxAttempts - 1) {
          const backoffMs = classified.retryAfterMs || this.backoffIntervals[attempt];
          logger.warn(`Attempt ${attempt + 1} failed, retrying after ${backoffMs}ms: ${lastError.message}`);
          await this.sleep(backoffMs);
        }
      }
    }
    
    throw new Error(`Operation failed after ${maxAttempts} attempts: ${lastError?.message}`);
  }
  
  private sleep(ms: number): Promise<void> {
    return new Promise(resolve => setTimeout(resolve, ms));
  }
}

const retryManager = new RetryManager();
```

2. **Implement idempotency:**

```typescript
class IdempotentSnapshotStore {
  private processedKeys: Set<string> = new Set();
  private readonly maxCacheSize = 10000;
  
  async saveSnapshot(snapshot: Snapshot, idempotencyKey?: string): Promise<void> {
    const key = idempotencyKey || snapshot.handshake.snapshot_id;
    
    // Check if already processed
    if (this.processedKeys.has(key)) {
      logger.info(`Snapshot already saved (idempotency): ${key}`);
      return;
    }
    
    // Save snapshot
    await retryManager.withRetry(
      async () => {
        await sqliteStore.saveSnapshot(snapshot);
      },
      { idempotencyKey: key, context: { snapshotId: snapshot.handshake.snapshot_id } }
    );
    
    // Mark as processed
    this.processedKeys.add(key);
    
    // Prevent unbounded growth
    if (this.processedKeys.size > this.maxCacheSize) {
      const keysArray = Array.from(this.processedKeys);
      this.processedKeys = new Set(keysArray.slice(-this.maxCacheSize / 2));
    }
  }
}

const idempotentStore = new IdempotentSnapshotStore();
```

---

### Q50: Catastrophic Failure

**Answer:**

**Gateway Recovery:**
- Runs under supervisors: `launchd` (macOS) or `systemd` (Linux)
- Automatic restart on process crash

**Session/Job Persistence:**
- Cron jobs persisted to `jobs.json`
- Session transcripts stored as JSONL files
- Not lost during crash/restart

**Event Recovery:**
- **No event replay** - events lost during disconnection are GONE
- Must perform full refresh to synchronize

**State Integrity:**
- `openclaw doctor` repairs catastrophic state loss
- Checks for missing state directories
- Repairs permissions

### Implications for Cognitive Controller

✅ **Persistence Strategy:**
- Must persist snapshots to disk immediately
- Can't rely on in-memory state

⚠️ **No Event Replay:**
- Critical: Must implement state recovery
- Can't assume we'll receive all events

⚠️ **Crash Risk:**
- No shutdown hook (from Q14)
- Must flush aggressively

### Action Items

1. **Implement aggressive persistence:**

```typescript
class AggressivePersistence {
  private flushInterval: NodeJS.Timeout | null = null;
  private pendingWrites: Map<string, Snapshot> = new Map();
  
  start(): void {
    // Flush every 5 seconds
    this.flushInterval = setInterval(() => {
      this.flush();
    }, 5000);
    
    logger.info('Aggressive persistence started (5s flush interval)');
  }
  
  stop(): void {
    if (this.flushInterval) {
      clearInterval(this.flushInterval);
      this.flushInterval = null;
    }
    
    // Final flush
    this.flush();
  }
  
  async queueSnapshot(snapshot: Snapshot): Promise<void> {
    this.pendingWrites.set(snapshot.handshake.snapshot_id, snapshot);
    
    // If queue is large, flush immediately
    if (this.pendingWrites.size > 10) {
      await this.flush();
    }
  }
  
  private async flush(): Promise<void> {
    if (this.pendingWrites.size === 0) return;
    
    const snapshots = Array.from(this.pendingWrites.values());
    this.pendingWrites.clear();
    
    logger.info(`Flushing ${snapshots.length} snapshots to disk`);
    
    for (const snapshot of snapshots) {
      try {
        await sqliteStore.saveSnapshot(snapshot);
      } catch (error) {
        logger.error(`Failed to flush snapshot ${snapshot.handshake.snapshot_id}: ${error.message}`);
        // Re-queue for next flush
        this.pendingWrites.set(snapshot.handshake.snapshot_id, snapshot);
      }
    }
  }
}

const aggressivePersistence = new AggressivePersistence();
```

2. **Implement state recovery:**

```typescript
class StateRecovery {
  async recoverFromCrash(agentId: string): Promise<void> {
    logger.info(`Recovering state for agent: ${agentId}`);
    
    // 1. Check for pending writes
    const pendingSnapshots = await this.findPendingSnapshots(agentId);
    
    if (pendingSnapshots.length > 0) {
      logger.warn(`Found ${pendingSnapshots.length} pending snapshots, recovering...`);
      for (const snapshot of pendingSnapshots) {
        await sqliteStore.saveSnapshot(snapshot);
      }
    }
    
    // 2. Verify database integrity
    const isHealthy = await sqliteStore.checkIntegrity(agentId);
    
    if (!isHealthy) {
      logger.error(`Database corrupted for agent ${agentId}, attempting repair`);
      await this.repairDatabase(agentId);
    }
    
    // 3. Rebuild indexes if needed
    await this.rebuildIndexes(agentId);
    
    logger.info(`State recovery complete for agent: ${agentId}`);
  }
  
  private async findPendingSnapshots(agentId: string): Promise<Snapshot[]> {
    // Check for .tmp files or write-ahead logs
    const ccDir = pathManager.getCognitiveControllerDir(agentId);
    const tmpFiles = await fs.readdir(ccDir).then(files => 
      files.filter(f => f.endsWith('.tmp') || f.endsWith('.wal'))
    );
    
    const snapshots: Snapshot[] = [];
    
    for (const file of tmpFiles) {
      try {
        const content = await fs.readFile(path.join(ccDir, file), 'utf-8');
        const snapshot = JSON.parse(content);
        snapshots.push(snapshot);
      } catch (error) {
        logger.warn(`Failed to recover from ${file}: ${error.message}`);
      }
    }
    
    return snapshots;
  }
  
  private async repairDatabase(agentId: string): Promise<void> {
    const dbPath = pathManager.getDatabasePath(agentId);
    const backupPath = `${dbPath}.backup`;
    
    // Backup corrupted database
    await fs.copyFile(dbPath, backupPath);
    
    // Recreate database
    await sqliteStore.initialize(dbPath);
    
    logger.info(`Database repaired for agent: ${agentId}`);
  }
  
  private async rebuildIndexes(agentId: string): Promise<void> {
    // Rebuild any indexes that may be corrupted
    await sqliteStore.rebuildIndexes(agentId);
  }
}

const stateRecovery = new StateRecovery();
```

3. **Implement health check integration:**

```typescript
class HealthCheckIntegration {
  async registerHealthCheck(): Promise<void> {
    // Register with OpenClaw health system
    gateway.registerHealthCheck('cognitive-controller', async () => {
      const agents = await this.getActiveAgents();
      const results = [];
      
      for (const agentId of agents) {
        const health = await this.checkAgentHealth(agentId);
        results.push(health);
      }
      
      return {
        status: results.every(r => r.healthy) ? 'healthy' : 'degraded',
        agents: results
      };
    });
  }
  
  private async checkAgentHealth(agentId: string): Promise<any> {
    try {
      // Check database connectivity
      const dbHealthy = await sqliteStore.checkIntegrity(agentId);
      
      // Check recent snapshot
      const latestSnapshot = await sqliteStore.getLatestSnapshot(agentId);
      const snapshotAge = latestSnapshot 
        ? Date.now() - new Date(latestSnapshot.handshake.created_at).getTime()
        : Infinity;
      
      // Check pending writes
      const pendingCount = aggressivePersistence.pendingWrites.size;
      
      return {
        agentId,
        healthy: dbHealthy && snapshotAge < 3600000 && pendingCount < 50,
        database: dbHealthy ? 'ok' : 'corrupted',
        lastSnapshotAge: snapshotAge,
        pendingWrites: pendingCount
      };
    } catch (error) {
      return {
        agentId,
        healthy: false,
        error: error.message
      };
    }
  }
  
  private async getActiveAgents(): Promise<string[]> {
    // Get list of agents with Cognitive Controller enabled
    const config = await gateway.getConfig();
    return config.agents.list
      .filter(a => a.skills?.['cognitive-controller']?.enabled)
      .map(a => a.id);
  }
}
```

---

### Q51: Rate Limit Management

**Answer:**

**Detection:**
- Gateway detects via API responses (429 errors)
- RPC-level limits return `UNAVAILABLE` status

**Backoff Strategy:**
- Rate-limited RPCs provide `retryAfterMs` value
- Webhook auth failures rate-limited per client address

**Coordination:**
- Control-plane write RPCs: 3 requests per 60 seconds per device/IP
- Auth profiles enter "cooldown" state on rate limit

### Implications for Cognitive Controller

✅ **Rate Limit Awareness:**
- Can detect and respect rate limits
- Use provided `retryAfterMs` values

⚠️ **Write Limits:**
- 3 writes per 60 seconds is VERY restrictive
- Must batch operations carefully

### Action Items

1. **Implement rate limit tracking:**

```typescript
class RateLimitTracker {
  private limits: Map<string, RateLimit> = new Map();
  
  recordRequest(operation: string): void {
    const limit = this.getLimit(operation);
    limit.requests.push(Date.now());
    
    // Clean old requests
    const cutoff = Date.now() - limit.windowMs;
    limit.requests = limit.requests.filter(t => t > cutoff);
  }
  
  canMakeRequest(operation: string): boolean {
    const limit = this.getLimit(operation);
    const cutoff = Date.now() - limit.windowMs;
    const recentRequests = limit.requests.filter(t => t > cutoff);
    
    return recentRequests.length < limit.maxRequests;
  }
  
  getRetryAfter(operation: string): number {
    const limit = this.getLimit(operation);
    const cutoff = Date.now() - limit.windowMs;
    const recentRequests = limit.requests.filter(t => t > cutoff);
    
    if (recentRequests.length < limit.maxRequests) {
      return 0;
    }
    
    // Return time until oldest request expires
    const oldestRequest = Math.min(...recentRequests);
    return (oldestRequest + limit.windowMs) - Date.now();
  }
  
  private getLimit(operation: string): RateLimit {
    if (!this.limits.has(operation)) {
      // Default: 3 requests per 60 seconds (OpenClaw control-plane limit)
      this.limits.set(operation, {
        maxRequests: 3,
        windowMs: 60000,
        requests: []
      });
    }
    return this.limits.get(operation)!;
  }
}

interface RateLimit {
  maxRequests: number;
  windowMs: number;
  requests: number[];
}

const rateLimitTracker = new RateLimitTracker();
```

2. **Implement batching to respect limits:**

```typescript
class BatchedSnapshotWriter {
  private batchQueue: Snapshot[] = [];
  private flushTimer: NodeJS.Timeout | null = null;
  
  async queueSnapshot(snapshot: Snapshot): Promise<void> {
    this.batchQueue.push(snapshot);
    
    // Schedule flush if not already scheduled
    if (!this.flushTimer) {
      this.scheduleFlush();
    }
  }
  
  private scheduleFlush(): void {
    // Wait until we can make a request
    const retryAfter = rateLimitTracker.getRetryAfter('snapshot_write');
    
    if (retryAfter > 0) {
      logger.info(`Rate limited, scheduling flush in ${retryAfter}ms`);
      this.flushTimer = setTimeout(() => this.flush(), retryAfter);
    } else {
      // Flush immediately
      this.flushTimer = setTimeout(() => this.flush(), 0);
    }
  }
  
  private async flush(): Promise<void> {
    this.flushTimer = null;
    
    if (this.batchQueue.length === 0) return;
    
    // Check if we can make request
    if (!rateLimitTracker.canMakeRequest('snapshot_write')) {
      logger.warn('Rate limit reached, rescheduling flush');
      this.scheduleFlush();
      return;
    }
    
    // Take batch (up to 10 snapshots)
    const batch = this.batchQueue.splice(0, 10);
    
    logger.info(`Flushing batch of ${batch.length} snapshots`);
    
    try {
      // Write batch to database (single transaction)
      await sqliteStore.saveBatch(batch);
      rateLimitTracker.recordRequest('snapshot_write');
    } catch (error) {
      logger.error(`Batch write failed: ${error.message}`);
      // Re-queue snapshots
      this.batchQueue.unshift(...batch);
    }
    
    // Schedule next flush if queue not empty
    if (this.batchQueue.length > 0) {
      this.scheduleFlush();
    }
  }
}

const batchedWriter = new BatchedSnapshotWriter();
```

---

### Q52: Logging Infrastructure

**Answer:**

**Destinations:**
- Gateway logs to stdout in foreground
- View/follow via CLI: `openclaw logs --follow`

**Levels:**
- Debug output: `--verbose` flag
- Provides npm notice-level logs and shell-level tracing

**Structure:**
- Cron run history and session transcripts in JSONL format

**Custom Fields:**
- Webhook logs avoid sensitive raw payloads
- No explicit API for plugins to inject custom fields into core Gateway logs

### Implications for Cognitive Controller

✅ **Standard Logging:**
- Use stdout for logs
- Follow OpenClaw conventions

⚠️ **No Custom Fields:**
- Can't inject into Gateway logs
- Must maintain separate log files

### Action Items

1. **Implement structured logging:**

```typescript
import winston from 'winston';

const logger = winston.createLogger({
  level: process.env.VERBOSE ? 'debug' : 'info',
  format: winston.format.combine(
    winston.format.timestamp(),
    winston.format.json()
  ),
  transports: [
    // Stdout for Gateway integration
    new winston.transports.Console({
      format: winston.format.combine(
        winston.format.colorize(),
        winston.format.simple()
      )
    }),
    // Separate file for detailed logs
    new winston.transports.File({
      filename: path.join(pathManager.getCognitiveControllerDir('global'), 'cognitive-controller.log'),
      format: winston.format.json()
    })
  ]
});

// Add agent context to logs
function createAgentLogger(agentId: string): winston.Logger {
  return logger.child({ agentId });
}
```

2. **Implement JSONL audit log:**

```typescript
class AuditLogger {
  private streams: Map<string, fs.WriteStream> = new Map();
  
  async logEvent(agentId: string, event: AuditEvent): Promise<void> {
    const stream = this.getStream(agentId);
    const line = JSON.stringify(event) + '\n';
    
    return new Promise((resolve, reject) => {
      stream.write(line, (error) => {
        if (error) reject(error);
        else resolve();
      });
    });
  }
  
  private getStream(agentId: string): fs.WriteStream {
    if (!this.streams.has(agentId)) {
      const logPath = path.join(
        pathManager.getCognitiveControllerDir(agentId),
        'audit.jsonl'
      );
      
      const stream = fs.createWriteStream(logPath, { flags: 'a' });
      this.streams.set(agentId, stream);
    }
    
    return this.streams.get(agentId)!;
  }
  
  close(): void {
    for (const stream of this.streams.values()) {
      stream.end();
    }
    this.streams.clear();
  }
}

interface AuditEvent {
  timestamp: string;
  agentId: string;
  eventType: string;
  snapshotId?: string;
  details: Record<string, any>;
}

const auditLogger = new AuditLogger();
```

---

## Performance & Scalability (Q53-Q56)

### Q53: Performance Baselines

**Answer:**

**Event Handling:**
- WebSocket for real-time server-push events
- Event types: agent, chat, health, tick

**Tool/Turn Timing:**
- Default agent turn timeout: 120 seconds
- Memory search timeout: 4000ms (4 seconds)

**Context Management:**
- Memory search targets ~400 token chunks
- Compaction triggered when < 20,000 token floor remains

### Implications for Cognitive Controller

✅ **Timing Constraints:**
- Snapshot creation must complete within turn timeout
- Target: < 5 seconds for snapshot creation

✅ **Chunk Size:**
- 400 tokens is good target for roll-up chunks
- Aligns with OpenClaw's memory search

### Action Items

1. **Implement performance monitoring:**

```typescript
class PerformanceMonitor {
  private metrics: Map<string, Metric[]> = new Map();
  
  startTimer(operation: string): () => void {
    const start = Date.now();
    
    return () => {
      const duration = Date.now() - start;
      this.recordMetric(operation, duration);
    };
  }
  
  recordMetric(operation: string, duration: number): void {
    if (!this.metrics.has(operation)) {
      this.metrics.set(operation, []);
    }
    
    const metrics = this.metrics.get(operation)!;
    metrics.push({ timestamp: Date.now(), duration });
    
    // Keep last 1000 metrics
    if (metrics.length > 1000) {
      metrics.shift();
    }
    
    // Log slow operations
    if (duration > 5000) {
      logger.warn(`Slow operation: ${operation} took ${duration}ms`);
    }
  }
  
  getStats(operation: string): PerformanceStats | null {
    const metrics = this.metrics.get(operation);
    if (!metrics || metrics.length === 0) return null;
    
    const durations = metrics.map(m => m.duration);
    const sorted = durations.sort((a, b) => a - b);
    
    return {
      count: durations.length,
      min: sorted[0],
      max: sorted[sorted.length - 1],
      mean: durations.reduce((a, b) => a + b, 0) / durations.length,
      p50: sorted[Math.floor(sorted.length * 0.5)],
      p95: sorted[Math.floor(sorted.length * 0.95)],
      p99: sorted[Math.floor(sorted.length * 0.99)]
    };
  }
}

interface Metric {
  timestamp: number;
  duration: number;
}

interface PerformanceStats {
  count: number;
  min: number;
  max: number;
  mean: number;
  p50: number;
  p95: number;
  p99: number;
}

const perfMonitor = new PerformanceMonitor();
```

2. **Optimize snapshot creation:**

```typescript
class OptimizedSnapshotAssembly {
  async createSnapshot(context: any): Promise<Snapshot> {
    const endTimer = perfMonitor.startTimer('snapshot_creation');
    
    try {
      // Parallel extraction
      const [objective, threads, facts, constraints, decisions, questions, toolState, actions] = 
        await Promise.all([
          this.extractObjective(context),
          this.extractThreads(context),
          this.extractFacts(context),
          this.extractConstraints(context),
          this.extractDecisions(context),
          this.extractQuestions(context),
          this.extractToolState(context),
          this.extractActions(context)
        ]);
      
      const snapshot: Snapshot = {
        handshake: {
          snapshot_id: generateUUID(),
          created_at: new Date().toISOString(),
          expires_at: new Date(Date.now() + 48 * 3600000).toISOString(),
          source_event_ids: context.eventIds || [],
          schema_version: '1.0'
        },
        objective_now: objective,
        active_threads: threads,
        recent_facts: facts,
        hard_constraints: constraints,
        decisions: decisions,
        open_questions: questions,
        tool_state: toolState,
        last_actions: actions
      };
      
      return snapshot;
    } finally {
      endTimer();
    }
  }
  
  // Extraction methods run in parallel
  private async extractObjective(context: any): Promise<string> {
    const endTimer = perfMonitor.startTimer('extract_objective');
    try {
      // Extract objective from context
      return context.objective || 'Continue work';
    } finally {
      endTimer();
    }
  }
  
  // ... other extraction methods
}
```

3. **Set performance targets:**

```typescript
const PERFORMANCE_TARGETS = {
  snapshot_creation: 5000,      // 5 seconds max
  snapshot_save: 1000,          // 1 second max
  fact_extraction: 2000,        // 2 seconds max
  roll_up_write: 500,           // 500ms max
  event_capture: 100            // 100ms max
};

class PerformanceValidator {
  validateOperation(operation: string, duration: number): boolean {
    const target = PERFORMANCE_TARGETS[operation];
    
    if (!target) {
      logger.warn(`No performance target for operation: ${operation}`);
      return true;
    }
    
    if (duration > target) {
      logger.error(`Performance target missed: ${operation} took ${duration}ms (target: ${target}ms)`);
      return false;
    }
    
    return true;
  }
}
```

---

### Q54: Resource Limits

**Answer:**

**Sandbox Limits:**
- Memory: 1GB
- Swap: 2GB
- CPU: 1 core

**Concurrency:**
- Cron jobs: 1 concurrent run (default, configurable)
- Global agent concurrency: `agents.defaults.maxConcurrent`

**Data Caps:**
- Telegram media: 5MB max
- Heartbeat acknowledgments: 300 characters max (dropped if exceeded)

### Implications for Cognitive Controller

✅ **Resource Awareness:**
- Must operate within 1GB memory limit
- CPU-intensive operations should be async

⚠️ **Concurrency Limits:**
- Must respect global concurrency settings
- Can't assume unlimited parallel operations

### Action Items

1. **Implement resource monitoring:**

```typescript
class ResourceMonitor {
  private memoryCheckInterval: NodeJS.Timeout | null = null;
  
  start(): void {
    // Check memory every 30 seconds
    this.memoryCheckInterval = setInterval(() => {
      this.checkMemory();
    }, 30000);
  }
  
  stop(): void {
    if (this.memoryCheckInterval) {
      clearInterval(this.memoryCheckInterval);
      this.memoryCheckInterval = null;
    }
  }
  
  private checkMemory(): void {
    const usage = process.memoryUsage();
    const heapUsedMB = usage.heapUsed / 1024 / 1024;
    const heapTotalMB = usage.heapTotal / 1024 / 1024;
    const rssMB = usage.rss / 1024 / 1024;
    
    logger.debug(`Memory usage: Heap ${heapUsedMB.toFixed(2)}MB / ${heapTotalMB.toFixed(2)}MB, RSS ${rssMB.toFixed(2)}MB`);
    
    // Warn if approaching 1GB limit
    if (rssMB > 800) {
      logger.warn(`High memory usage: ${rssMB.toFixed(2)}MB (limit: 1024MB)`);
      this.triggerGarbageCollection();
    }
    
    // Critical if over 900MB
    if (rssMB > 900) {
      logger.error(`Critical memory usage: ${rssMB.toFixed(2)}MB, triggering emergency cleanup`);
      this.emergencyCleanup();
    }
  }
  
  private triggerGarbageCollection(): void {
    if (global.gc) {
      logger.info('Triggering garbage collection');
      global.gc();
    }
  }
  
  private async emergencyCleanup(): Promise<void> {
    logger.warn('Emergency cleanup initiated');
    
    // Clear caches
    tokenEstimator.encoders.clear();
    errorClassifier.processedKeys?.clear();
    
    // Flush pending writes
    await aggressivePersistence.flush();
    
    // Force GC
    this.triggerGarbageCollection();
  }
}

const resourceMonitor = new ResourceMonitor();
```

2. **Implement memory-efficient operations:**

```typescript
class MemoryEfficientStore {
  async processLargeDataset(agentId: string, items: any[]): Promise<void> {
    // Process in chunks to avoid memory spikes
    const chunkSize = 100;
    
    for (let i = 0; i < items.length; i += chunkSize) {
      const chunk = items.slice(i, i + chunkSize);
      
      await this.processChunk(agentId, chunk);
      
      // Allow GC between chunks
      await this.sleep(10);
    }
  }
  
  private async processChunk(agentId: string, chunk: any[]): Promise<void> {
    // Process chunk
    for (const item of chunk) {
      await sqliteStore.insertFact(item);
    }
  }
  
  private sleep(ms: number): Promise<void> {
    return new Promise(resolve => setTimeout(resolve, ms));
  }
}
```

---

### Q55: Performance Monitoring

**Answer:**

**Health Checks:**
- `health` command over WebSocket
- CLI command: `openclaw health`

**Real-time Metrics:**
- Control UI dashboard for sessions and Gateway health
- Indicator events for background tasks (heartbeats, etc.)

**Diagnostics:**
- `openclaw status`: Gateway runtime snapshot (PID, port)
- `openclaw doctor`: Diagnostics and repairs

### Implications for Cognitive Controller

✅ **Health Integration:**
- Can register health checks with Gateway
- Expose metrics via Control UI

✅ **Diagnostics:**
- Can implement `doctor`-style diagnostics
- Self-healing capabilities

### Action Items

1. **Register health checks:**

```typescript
class CognitiveControllerHealth {
  async registerHealthChecks(): Promise<void> {
    // Register with Gateway
    gateway.registerHealthCheck('cognitive-controller', async () => {
      return await this.getHealthStatus();
    });
    
    logger.info('Health checks registered');
  }
  
  async getHealthStatus(): Promise<HealthStatus> {
    const agents = await this.getActiveAgents();
    const agentHealth = await Promise.all(
      agents.map(agentId => this.checkAgentHealth(agentId))
    );
    
    const allHealthy = agentHealth.every(h => h.status === 'healthy');
    
    return {
      status: allHealthy ? 'healthy' : 'degraded',
      timestamp: new Date().toISOString(),
      agents: agentHealth,
      metrics: {
        totalSnapshots: agentHealth.reduce((sum, h) => sum + h.snapshotCount, 0),
        totalFacts: agentHealth.reduce((sum, h) => sum + h.factCount, 0),
        pendingWrites: aggressivePersistence.pendingWrites.size
      }
    };
  }
  
  private async checkAgentHealth(agentId: string): Promise<AgentHealth> {
    try {
      const dbHealthy = await sqliteStore.checkIntegrity(agentId);
      const latestSnapshot = await sqliteStore.getLatestSnapshot(agentId);
      const snapshotCount = await sqliteStore.getSnapshotCount(agentId);
      const factCount = await sqliteStore.getFactCount(agentId);
      
      const snapshotAge = latestSnapshot
        ? Date.now() - new Date(latestSnapshot.handshake.created_at).getTime()
        : Infinity;
      
      const status = dbHealthy && snapshotAge < 3600000 ? 'healthy' : 'degraded';
      
      return {
        agentId,
        status,
        database: dbHealthy ? 'ok' : 'corrupted',
        lastSnapshotAge: snapshotAge,
        snapshotCount,
        factCount
      };
    } catch (error) {
      return {
        agentId,
        status: 'unhealthy',
        error: error.message,
        snapshotCount: 0,
        factCount: 0
      };
    }
  }
  
  private async getActiveAgents(): Promise<string[]> {
    const config = await gateway.getConfig();
    return config.agents.list
      .filter(a => a.skills?.['cognitive-controller']?.enabled)
      .map(a => a.id);
  }
}

interface HealthStatus {
  status: 'healthy' | 'degraded' | 'unhealthy';
  timestamp: string;
  agents: AgentHealth[];
  metrics: {
    totalSnapshots: number;
    totalFacts: number;
    pendingWrites: number;
  };
}

interface AgentHealth {
  agentId: string;
  status: 'healthy' | 'degraded' | 'unhealthy';
  database?: string;
  lastSnapshotAge?: number;
  snapshotCount: number;
  factCount: number;
  error?: string;
}
```

2. **Implement diagnostics:**

```typescript
class CognitiveControllerDoctor {
  async diagnose(agentId?: string): Promise<DiagnosticReport> {
    logger.info('Running diagnostics...');
    
    const agents = agentId ? [agentId] : await this.getAllAgents();
    const reports: AgentDiagnostic[] = [];
    
    for (const id of agents) {
      reports.push(await this.diagnoseAgent(id));
    }
    
    return {
      timestamp: new Date().toISOString(),
      agents: reports,
      summary: this.summarize(reports)
    };
  }
  
  private async diagnoseAgent(agentId: string): Promise<AgentDiagnostic> {
    const issues: string[] = [];
    const fixes: string[] = [];
    
    // Check database
    const dbPath = pathManager.getDatabasePath(agentId);
    const dbExists = await fs.access(dbPath).then(() => true).catch(() => false);
    
    if (!dbExists) {
      issues.push('Database file missing');
      fixes.push('Recreate database');
      await sqliteStore.initialize(dbPath);
    }
    
    // Check database integrity
    const dbHealthy = await sqliteStore.checkIntegrity(agentId);
    if (!dbHealthy) {
      issues.push('Database corrupted');
      fixes.push('Repair database');
      await stateRecovery.repairDatabase(agentId);
    }
    
    // Check roll-up directory
    const rollUpDir = pathManager.getRollUpDir(agentId);
    const rollUpExists = await fs.access(rollUpDir).then(() => true).catch(() => false);
    
    if (!rollUpExists) {
      issues.push('Roll-up directory missing');
      fixes.push('Create roll-up directory');
      await fs.mkdir(rollUpDir, { recursive: true });
    }
    
    // Check snapshot count
    const snapshotCount = await sqliteStore.getSnapshotCount(agentId);
    if (snapshotCount === 0) {
      issues.push('No snapshots found');
      fixes.push('Create initial snapshot');
      // Don't auto-fix, just report
    }
    
    // Check for orphaned files
    const orphanedFiles = await this.findOrphanedFiles(agentId);
    if (orphanedFiles.length > 0) {
      issues.push(`${orphanedFiles.length} orphaned files found`);
      fixes.push('Clean up orphaned files');
      // Don't auto-fix, just report
    }
    
    return {
      agentId,
      healthy: issues.length === 0,
      issues,
      fixes,
      snapshotCount
    };
  }
  
  private async findOrphanedFiles(agentId: string): Promise<string[]> {
    const ccDir = pathManager.getCognitiveControllerDir(agentId);
    const files = await fs.readdir(ccDir);
    
    return files.filter(f => f.endsWith('.tmp') || f.endsWith('.bak'));
  }
  
  private summarize(reports: AgentDiagnostic[]): string {
    const healthy = reports.filter(r => r.healthy).length;
    const total = reports.length;
    
    if (healthy === total) {
      return `All ${total} agents healthy`;
    } else {
      return `${healthy}/${total} agents healthy, ${total - healthy} need attention`;
    }
  }
  
  private async getAllAgents(): Promise<string[]> {
    const config = await gateway.getConfig();
    return config.agents.list.map(a => a.id);
  }
}

interface DiagnosticReport {
  timestamp: string;
  agents: AgentDiagnostic[];
  summary: string;
}

interface AgentDiagnostic {
  agentId: string;
  healthy: boolean;
  issues: string[];
  fixes: string[];
  snapshotCount: number;
}

const doctor = new CognitiveControllerDoctor();
```

---

### Q56: Scalability Limits

**Answer:**

**Memory Indexing:**
- Embedding cache: 50,000 entries (default)
- Prevents SQLite bloat

**Session Indexing:**
- Background sync triggered after:
  - 100KB (deltaBytes) OR
  - 50 messages (deltaMessages)

**Configuration Depth:**
- `$include` supported up to 10 levels deep

**Agent Limits:**
- No hard cap on agent count
- Multiple isolated agents run simultaneously
- Each has own workspace and session store

### Implications for Cognitive Controller

✅ **Scalability:**
- Can support many agents simultaneously
- Per-agent isolation prevents interference

⚠️ **Cache Limits:**
- Must respect 50K embedding cache limit
- Implement our own caching strategy

⚠️ **Sync Thresholds:**
- 100KB or 50 messages triggers sync
- Must align our snapshot frequency

### Action Items

1. **Implement cache management:**

```typescript
class CacheManager {
  private cache: Map<string, CacheEntry> = new Map();
  private readonly maxEntries = 10000; // Conservative limit
  
  get(key: string): any | null {
    const entry = this.cache.get(key);
    
    if (!entry) return null;
    
    // Check expiration
    if (entry.expiresAt < Date.now()) {
      this.cache.delete(key);
      return null;
    }
    
    // Update access time
    entry.lastAccess = Date.now();
    
    return entry.value;
  }
  
  set(key: string, value: any, ttlMs: number = 3600000): void {
    // Evict if at capacity
    if (this.cache.size >= this.maxEntries) {
      this.evictLRU();
    }
    
    this.cache.set(key, {
      value,
      expiresAt: Date.now() + ttlMs,
      lastAccess: Date.now()
    });
  }
  
  private evictLRU(): void {
    let oldestKey: string | null = null;
    let oldestAccess = Infinity;
    
    for (const [key, entry] of this.cache.entries()) {
      if (entry.lastAccess < oldestAccess) {
        oldestAccess = entry.lastAccess;
        oldestKey = key;
      }
    }
    
    if (oldestKey) {
      this.cache.delete(oldestKey);
      logger.debug(`Evicted cache entry: ${oldestKey}`);
    }
  }
  
  clear(): void {
    this.cache.clear();
  }
}

interface CacheEntry {
  value: any;
  expiresAt: number;
  lastAccess: number;
}

const cacheManager = new CacheManager();
```

2. **Implement scalability monitoring:**

```typescript
class ScalabilityMonitor {
  async checkScalability(): Promise<ScalabilityReport> {
    const agents = await this.getAllAgents();
    const agentMetrics = await Promise.all(
      agents.map(agentId => this.getAgentMetrics(agentId))
    );
    
    const totalSnapshots = agentMetrics.reduce((sum, m) => sum + m.snapshotCount, 0);
    const totalFacts = agentMetrics.reduce((sum, m) => sum + m.factCount, 0);
    const totalSize = agentMetrics.reduce((sum, m) => sum + m.databaseSize, 0);
    
    return {
      agentCount: agents.length,
      totalSnapshots,
      totalFacts,
      totalSize,
      averageSnapshotsPerAgent: totalSnapshots / agents.length,
      averageFactsPerAgent: totalFacts / agents.length,
      averageSizePerAgent: totalSize / agents.length,
      agents: agentMetrics
    };
  }
  
  private async getAgentMetrics(agentId: string): Promise<AgentMetrics> {
    const snapshotCount = await sqliteStore.getSnapshotCount(agentId);
    const factCount = await sqliteStore.getFactCount(agentId);
    const databaseSize = await this.getDatabaseSize(agentId);
    
    return {
      agentId,
      snapshotCount,
      factCount,
      databaseSize
    };
  }
  
  private async getDatabaseSize(agentId: string): Promise<number> {
    const dbPath = pathManager.getDatabasePath(agentId);
    const stats = await fs.stat(dbPath);
    return stats.size;
  }
  
  private async getAllAgents(): Promise<string[]> {
    const config = await gateway.getConfig();
    return config.agents.list.map(a => a.id);
  }
}

interface ScalabilityReport {
  agentCount: number;
  totalSnapshots: number;
  totalFacts: number;
  totalSize: number;
  averageSnapshotsPerAgent: number;
  averageFactsPerAgent: number;
  averageSizePerAgent: number;
  agents: AgentMetrics[];
}

interface AgentMetrics {
  agentId: string;
  snapshotCount: number;
  factCount: number;
  databaseSize: number;
}

const scalabilityMonitor = new ScalabilityMonitor();
```

---

## Summary: Q48-Q56 Key Findings

### Error Handling
- ✅ Structured error taxonomy (RPC, Network, Tool, Auth)
- ✅ Automatic retry with exponential backoff
- ✅ Idempotency required for safe retries
- ⚠️ No event replay - must implement state recovery
- ⚠️ No shutdown hook - aggressive persistence required
- ⚠️ Rate limits: 3 writes per 60 seconds (very restrictive)

### Performance
- ✅ 120s turn timeout (snapshot creation must be < 5s)
- ✅ 400 token chunks align with memory search
- ✅ Health checks and diagnostics available
- ⚠️ 1GB memory limit (must monitor usage)
- ⚠️ 1 CPU core (async operations required)

### Scalability
- ✅ No hard agent limit (multiple agents supported)
- ✅ Per-agent isolation prevents interference
- ⚠️ 50K embedding cache limit
- ⚠️ Sync triggered at 100KB or 50 messages

### Critical Implementation Priorities

1. **Aggressive Persistence (HIGHEST PRIORITY)**
   - 5-second flush interval
   - No shutdown hook means data loss risk
   - Implement state recovery

2. **Rate Limit Management**
   - 3 writes per 60 seconds is VERY restrictive
   - Must batch operations
   - Track rate limits carefully

3. **Error Classification**
   - Categorize all errors
   - Implement retry logic
   - Graceful degradation

4. **Performance Monitoring**
   - Track all operation timings
   - Target < 5s for snapshot creation
   - Monitor memory usage

5. **Health & Diagnostics**
   - Register health checks
   - Implement doctor-style diagnostics
   - Self-healing capabilities

---

**Next Steps:**
- Continue with Q57-Q100 as answers become available
- Implement aggressive persistence prototype
- Test rate limit handling
- Validate performance targets

