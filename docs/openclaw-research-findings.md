# OpenClaw Research Findings: Plugin Architecture & Integration

**Date:** 2026-03-03  
**Source:** OpenClaw Documentation Database  
**Status:** Q1-Q8 Answered, Critical Implications Identified

---

## Executive Summary

OpenClaw's plugin system is **npm-based** with **Gateway restart required** for changes. This has significant implications for Cognitive Controller's deployment and development workflow.

**Critical Findings:**
1. ✅ Plugins are npm packages (standard package.json)
2. ⚠️ **No hot reload** - Gateway restart required for plugin changes
3. ⚠️ **No explicit plugin API** - must investigate GitHub source code
4. ✅ Robust event system via WebSocket API and Webhooks
5. ⚠️ **No shutdown hook documented** - may need to handle abrupt termination
6. ✅ Sandboxing available for file system and network access
7. ⚠️ **No inter-plugin communication** - plugins are isolated

---

## Q1: Plugin Registration API

### Answer

**Declaration:**
- Plugins are npm packages installed via `npm install`
- Must be published on npmjs.com
- Source must be hosted on GitHub (for community listing)

**Manifest:**
- Standard `package.json` (npm ecosystem)
- Required fields for community listing:
  - Plugin name
  - npm package name
  - One-line description

**Registration:**
- Installed as npm dependencies
- Configured in `~/.openclaw/openclaw.json` (global) or `<workspace>/skills/` (workspace-specific)

### Implications for Cognitive Controller

✅ **Good News:**
- Standard npm packaging (familiar tooling)
- Can use TypeScript, standard build tools
- Can publish to npm for distribution

⚠️ **Challenges:**
- Must create proper npm package structure
- Need to decide: global plugin or workspace-specific?
- Community listing requires GitHub hosting

### Action Items

1. Create `package.json` for Cognitive Controller:
```json
{
  "name": "@openclaw/cognitive-controller",
  "version": "0.1.0",
  "description": "Event-driven cognitive layer for OpenClaw with bounded memory management",
  "main": "dist/index.js",
  "types": "dist/index.d.ts",
  "keywords": ["openclaw", "plugin", "memory", "cognitive", "context-management"],
  "repository": {
    "type": "git",
    "url": "https://github.com/your-org/cognitive-controller.git"
  },
  "peerDependencies": {
    "openclaw": ">=0.1.0"
  }
}
```

2. Decide deployment model:
   - **Option A:** Global plugin (installed once, used by all agents)
   - **Option B:** Workspace-specific (per-agent installation)
   - **Recommendation:** Global plugin with per-agent configuration

---

## Q2: Plugin Initialization Sequence

### Answer

**Timing:**
- Plugins are classified as "Infrastructure"
- **Gateway restart required** for plugin configuration changes
- No hot reload support

**Sequence:**
- Exact serial/parallel initialization not specified
- QMD manager (search sidecar) initializes on Gateway startup
- Update timers armed before first call

**Dependencies:**
- No explicit plugin-level dependency declaration system documented

### Implications for Cognitive Controller

⚠️ **Critical Issue: No Hot Reload**
- Every code change requires Gateway restart
- Slow development iteration cycle
- Production deployments require downtime

⚠️ **Unknown Initialization Order**
- Can't guarantee our plugin initializes before/after others
- Must handle "Gateway not ready" state gracefully
- May need to poll for Gateway availability

✅ **Mitigation Strategies:**
1. **Development:** Use file watchers + auto-restart scripts
2. **Production:** Design for fast initialization (<1 second)
3. **Resilience:** Implement "lazy initialization" pattern

### Action Items

1. Create development script with auto-restart:
```json
{
  "scripts": {
    "dev": "nodemon --watch src --exec 'npm run build && openclaw restart'",
    "build": "tsc",
    "start": "node dist/index.js"
  }
}
```

2. Implement lazy initialization:
```typescript
class CognitiveController {
  private initialized = false;
  
  async ensureInitialized(): Promise<void> {
    if (this.initialized) return;
    
    try {
      await this.sqliteStore.initialize();
      await this.registerHooks();
      this.initialized = true;
    } catch (error) {
      logger.error('Initialization failed, will retry on next call', error);
      throw error;
    }
  }
  
  async onEvent(event: any): Promise<void> {
    await this.ensureInitialized();
    // Handle event...
  }
}
```

3. Add initialization timeout:
```typescript
const INIT_TIMEOUT = 10000; // 10 seconds

async function initializeWithTimeout(): Promise<void> {
  const timeout = new Promise((_, reject) => 
    setTimeout(() => reject(new Error('Initialization timeout')), INIT_TIMEOUT)
  );
  
  await Promise.race([
    cognitiveController.initialize(),
    timeout
  ]);
}
```

---

## Q3: Hooks and Events

### Answer

**Gateway Events (WebSocket API):**
- Server-push events: `agent`, `chat`, `presence`, `health`, `heartbeat`, `cron`
- Typed WebSocket API

**Webhooks System:**
- Actions: `wake` (enqueue system events), `agent` (run isolated agent turns)
- Can be restricted via `allowedAgentIds`

**Custom Transforms:**
- JS/TS transform modules in `~/.openclaw/hooks/transforms`
- Custom logic registration

### Implications for Cognitive Controller

✅ **Excellent News:**
- Rich event system for Observer component
- WebSocket API for real-time event capture
- Webhooks for external triggers

⚠️ **Questions Remaining:**
- What is the exact WebSocket endpoint URL?
- What are the event payload schemas?
- Can we register custom event types?

### Action Items

1. **Immediate:** Map our Observer hooks to OpenClaw events:

| Our Hook | OpenClaw Event | Priority |
|----------|---------------|----------|
| `onUserMessage` | `chat` event | High |
| `onToolResult` | `agent` event (tool completion) | High |
| `onStateTransition` | `agent` event (state change) | Medium |
| `emitHeartbeat` | `heartbeat` event | Medium |

2. **Research Needed:**
   - Q9: WebSocket API endpoint URL
   - Q10: Complete event schema
   - Q13: Can we emit custom events?

3. **Prototype WebSocket client:**
```typescript
import WebSocket from 'ws';

class GatewayClient {
  private ws: WebSocket;
  
  async connect(url: string): Promise<void> {
    this.ws = new WebSocket(url);
    
    this.ws.on('message', (data) => {
      const event = JSON.parse(data.toString());
      this.handleEvent(event);
    });
    
    this.ws.on('error', (error) => {
      logger.error('WebSocket error', error);
    });
  }
  
  private handleEvent(event: any): void {
    switch (event.type) {
      case 'chat':
        observer.onUserMessage(event.message, event.metadata);
        break;
      case 'agent':
        // Handle agent events (tool results, state transitions)
        break;
      case 'heartbeat':
        // Update our heartbeat tracking
        break;
    }
  }
}
```

---

## Q4: API Exposure Between Plugins

### Answer

**Inter-plugin Communication:**
- No specific service registry documented
- No versioning system for plugin APIs
- Plugins appear to be isolated

### Implications for Cognitive Controller

⚠️ **Isolation Challenge:**
- Can't easily share functionality with other plugins
- Can't depend on other plugins' services
- Must be self-contained

✅ **Simplification:**
- No need to worry about API versioning
- No dependency conflicts
- Simpler deployment

### Action Items

1. **Design Decision:** Make Cognitive Controller fully self-contained
   - Bundle all dependencies
   - No external plugin dependencies
   - Expose functionality via OpenClaw's native mechanisms (Webhooks, events)

2. **Future Enhancement:** If inter-plugin communication is needed:
   - Use OpenClaw's event system as message bus
   - Emit custom events that other plugins can subscribe to
   - Document event schemas for other plugin authors

---

## Q5: Plugin Shutdown Sequence

### Answer

**Shutdown Hook:**
- No specific `shutdown()` or `cleanup()` hook documented
- No defined time limit for cleanup

### Implications for Cognitive Controller

⚠️ **Critical Risk: Data Loss**
- If Gateway crashes or is killed, we may lose buffered events
- SQLite transactions may not complete
- Snapshots may be incomplete

⚠️ **Mitigation Required:**
- Aggressive flushing of buffers
- Short transaction windows
- Periodic checkpoints

### Action Items

1. **Implement aggressive flushing:**
```typescript
class Observer {
  private flushInterval = 1000; // Flush every 1 second
  
  constructor() {
    setInterval(() => this.flushBuffer(), this.flushInterval);
  }
  
  async flushBuffer(): Promise<void> {
    if (this.buffer.length === 0) return;
    
    try {
      await this.sqliteStore.insertEvents(this.buffer);
      this.buffer = [];
    } catch (error) {
      logger.error('Buffer flush failed', error);
      // Keep buffer for next attempt
    }
  }
}
```

2. **Add process signal handlers:**
```typescript
process.on('SIGTERM', async () => {
  logger.info('SIGTERM received, flushing buffers...');
  await observer.flushBuffer();
  await sqliteStore.close();
  process.exit(0);
});

process.on('SIGINT', async () => {
  logger.info('SIGINT received, flushing buffers...');
  await observer.flushBuffer();
  await sqliteStore.close();
  process.exit(0);
});
```

3. **Use WAL mode for SQLite:**
```typescript
// Enable Write-Ahead Logging for crash resilience
db.pragma('journal_mode = WAL');
db.pragma('synchronous = NORMAL'); // Balance between safety and performance
```

4. **Add periodic checkpoints:**
```typescript
setInterval(() => {
  db.pragma('wal_checkpoint(PASSIVE)');
}, 60000); // Checkpoint every minute
```

---

## Q6: Error Isolation

### Answer

**Crashes:**
- If configuration fails strict validation, Gateway refuses to start
- No mention of circuit breakers

**Diagnostics:**
- `openclaw doctor` tool scans for issues
- Repairs "extra gateway installs" and "stale config/state"

**Hot-Reloading:**
- Not supported for plugins
- Manual restart required after errors

### Implications for Cognitive Controller

⚠️ **Configuration Validation Critical:**
- Invalid config prevents Gateway startup
- Must provide schema validation
- Must provide helpful error messages

⚠️ **No Circuit Breakers:**
- If our plugin throws exceptions, may crash Gateway
- Must catch all exceptions
- Must never throw unhandled errors

### Action Items

1. **Add comprehensive error handling:**
```typescript
class CognitiveController {
  async onEvent(event: any): Promise<void> {
    try {
      await this.handleEvent(event);
    } catch (error) {
      logger.error('Event handling failed', { event, error });
      // Never throw - log and continue
    }
  }
  
  private async handleEvent(event: any): Promise<void> {
    try {
      await this.observer.capture(event);
    } catch (captureError) {
      logger.error('Event capture failed', captureError);
      // Try fallback
      await this.fallbackCapture(event);
    }
  }
}
```

2. **Add configuration validation:**
```typescript
import Ajv from 'ajv';

const configSchema = {
  type: 'object',
  properties: {
    snapshotTokenBudget: { type: 'number', minimum: 300, maximum: 1000 },
    maxActiveThreads: { type: 'number', minimum: 1, maximum: 10 },
    rollUpIntervalMinutes: { type: 'number', minimum: 1, maximum: 1440 }
  },
  required: ['snapshotTokenBudget', 'maxActiveThreads']
};

function validateConfig(config: any): void {
  const ajv = new Ajv();
  const validate = ajv.compile(configSchema);
  
  if (!validate(config)) {
    throw new Error(`Invalid configuration: ${JSON.stringify(validate.errors)}`);
  }
}
```

3. **Add health self-check:**
```typescript
async function healthCheck(): Promise<HealthStatus> {
  const checks = {
    sqliteConnected: false,
    observerRunning: false,
    rerankerHealthy: false
  };
  
  try {
    await sqliteStore.healthCheck();
    checks.sqliteConnected = true;
  } catch (error) {
    logger.error('SQLite health check failed', error);
  }
  
  // ... more checks
  
  const healthy = Object.values(checks).every(v => v);
  
  return {
    healthy,
    checks,
    timestamp: new Date().toISOString()
  };
}
```

---

## Q7: Configuration Mechanism

### Answer

**Location:**
- Global: `~/.openclaw/openclaw.json`
- Workspace-specific: `<workspace>/skills/`

**Format:**
- JSON5 (JSON with comments and trailing commas)

**Schema Extension:**
- Plugins managed via `skills` block
- Per-skill overrides, environment variables, API keys supported

**Hot Reload:**
- Not supported for plugins (Infrastructure category)

### Implications for Cognitive Controller

✅ **Configuration Strategy Clear:**
- Use `skills` block in `openclaw.json`
- Support both global and workspace-specific config
- Use JSON5 format (allows comments)

⚠️ **No Hot Reload:**
- Config changes require Gateway restart
- Development iteration slower
- Production config changes require downtime

### Action Items

1. **Define configuration schema:**
```json5
{
  "skills": {
    "cognitive-controller": {
      // Snapshot configuration
      "snapshotTokenBudget": 500,
      "maxActiveThreads": 7,
      "snapshotExpiryHours": 48,
      
      // Roll-up configuration
      "rollUpIntervalMinutes": 30,
      "rollUpIdleThresholdMinutes": 10,
      "enableAutoRollUp": true,
      
      // Confidence configuration
      "confidenceDampening": 0.3,
      "confidenceDecayRate": 0.05,
      "lowConfidenceThreshold": 0.3,
      
      // Thread management
      "stallDetectionMinutes": 30,
      
      // UI
      "enableToastNotifications": true,
      "enableLEDIndicator": true,
      
      // Storage
      "maxSnapshotsPerThread": 10,
      "eventRetentionHours": 48,
      "databasePath": "./cognitive-controller.db"
    }
  }
}
```

2. **Create config loader:**
```typescript
import fs from 'fs/promises';
import JSON5 from 'json5';

async function loadConfig(): Promise<CognitiveControllerConfig> {
  const globalConfigPath = path.join(
    os.homedir(),
    '.openclaw',
    'openclaw.json'
  );
  
  const configText = await fs.readFile(globalConfigPath, 'utf-8');
  const config = JSON5.parse(configText);
  
  const ourConfig = config.skills?.['cognitive-controller'] || {};
  
  // Merge with defaults
  return {
    ...DEFAULT_CONFIG,
    ...ourConfig
  };
}
```

3. **Add config validation on startup:**
```typescript
async function initialize(): Promise<void> {
  const config = await loadConfig();
  validateConfig(config);
  
  // Initialize components with config
  const sqliteStore = new SQLiteStore(config.databasePath);
  const observer = new Observer(config);
  // ...
}
```

---

## Q8: Permissions and Capabilities

### Answer

**File System:**
- Access relative to Agent Workspace by default
- Can reach absolute paths unless sandboxing enabled

**Network:**
- Sandboxed tools: no network egress by default (`network: "none"`)
- Must explicitly opt-in for network access

**Process Spawning:**
- `exec` tool and `process` tool available
- Governed by Tool Policy

**Internals:**
- Webhooks can be restricted via `allowedAgentIds`

### Implications for Cognitive Controller

✅ **File System Access:**
- Can write SQLite database to workspace
- Can write roll-ups to memory directory
- May need to handle sandboxing restrictions

⚠️ **Network Access:**
- If sandboxed, can't make external API calls
- Not an issue for Phase 0/1 (no external APIs)
- Phase 2+ (semantic search) may need network access

✅ **Process Spawning:**
- Not needed for our plugin
- All operations are in-process

### Action Items

1. **Design for sandboxed environment:**
```typescript
// Always use workspace-relative paths
function getDatabasePath(workspace: string): string {
  return path.join(workspace, '.cognitive-controller', 'memory.db');
}

function getRollUpPath(workspace: string): string {
  return path.join(workspace, 'memory', 'rollups');
}
```

2. **Add permission checks:**
```typescript
async function checkPermissions(workspace: string): Promise<void> {
  const dbPath = getDatabasePath(workspace);
  const rollUpPath = getRollUpPath(workspace);
  
  try {
    await fs.access(dbPath, fs.constants.W_OK);
  } catch (error) {
    throw new Error(`No write permission for database: ${dbPath}`);
  }
  
  try {
    await fs.mkdir(rollUpPath, { recursive: true });
  } catch (error) {
    throw new Error(`Cannot create roll-up directory: ${rollUpPath}`);
  }
}
```

3. **Document permission requirements:**
```markdown
## Permissions Required

Cognitive Controller requires the following permissions:

- **File System (Write):** 
  - `<workspace>/.cognitive-controller/` - SQLite database
  - `<workspace>/memory/rollups/` - Roll-up files

- **File System (Read):**
  - `~/.openclaw/openclaw.json` - Configuration

- **Network:** None (Phase 0/1)

If running in sandboxed mode, ensure these paths are whitelisted.
```

---

## Critical Gaps & Next Research Priorities

### High Priority (Blocking Phase 0)

**Q9: WebSocket API Endpoint**
- Need exact URL format
- Authentication requirements
- Connection lifecycle

**Q10: Event Schema**
- Complete list of event types
- Payload structure for each
- Event versioning

**Q15: Context Injection**
- How to inject snapshot before LLM calls
- Where injected context goes
- Token budget for plugin context

### Medium Priority (Needed for Phase 1)

**Q20: memory_search Tool Interface**
- Function signature
- Query syntax
- Response format

**Q28: Session Store**
- Storage format and location
- Can we access session data?
- Can we add custom metadata?

**Q32: Tool Registration**
- How to register custom tools
- Tool schema format
- Async/long-running tool support

### Low Priority (Phase 2+)

**Q44: Model Fallback Mechanism**
- How to trigger fallback
- Can we influence decisions?

**Q72: Web Control UI**
- Can we add custom panels?
- How to expose our LED status?

---

## Recommendations

### Immediate Actions

1. **Create npm package structure** (Q1)
2. **Implement lazy initialization** (Q2)
3. **Add comprehensive error handling** (Q6)
4. **Design configuration schema** (Q7)
5. **Add aggressive buffer flushing** (Q5)

### Research Priorities

1. **Investigate GitHub source code** for internal plugin API
2. **Test WebSocket connection** to Gateway
3. **Prototype event capture** from Gateway events
4. **Test context injection** mechanism
5. **Verify file system permissions** in sandboxed mode

### Design Decisions

1. **Deployment Model:** Global plugin with per-agent configuration
2. **Initialization:** Lazy initialization with retry logic
3. **Error Handling:** Catch all exceptions, never crash Gateway
4. **Configuration:** JSON5 in `skills` block
5. **Permissions:** Design for sandboxed environment

---

**Document Owner:** Project Team  
**Status:** Q1-Q8 Answered, Action Items Identified  
**Next Steps:** Research Q9-Q15 (high priority), prototype WebSocket client

---

## Gateway & Event System (Q9-Q14)

### Q9: Gateway WebSocket API Endpoint

**Answer:**

**URL Format:**
- Default: `ws://127.0.0.1:18789`
- Configurable via bind host setting
- HTTP content served at paths like `/__openclaw__/canvas/`
- Core WebSocket API uses primary bind host and port

**Authentication:**
- **Mandatory handshake required**
- First frame MUST be a `connect` request
- If `OPENCLAW_GATEWAY_TOKEN` configured, must provide in `connect.params.auth.token`
- Socket immediately closed if auth fails
- New devices require pairing approval
- Approved devices receive device token for subsequent connections

**Connection Lifecycle:**
- Mandatory handshake (non-JSON or non-connect first frame = hard close)
- **Events are NOT replayed** - clients responsible for state refresh on reconnection
- Connection gaps require full state refresh

### Implications for Cognitive Controller

🔴 **Critical: No Event Replay**
- If our WebSocket disconnects, we lose events
- Must implement state recovery mechanism
- Cannot rely on Gateway to replay missed events

🔴 **Authentication Required**
- Must handle device pairing flow
- Must store and use device token
- Must handle auth failures gracefully

⚠️ **Reconnection Strategy Critical**
- Must detect disconnections immediately
- Must refresh state on reconnection
- Must not assume event continuity

### Action Items

1. **Implement robust WebSocket client:**

```typescript
import WebSocket from 'ws';

class GatewayWebSocketClient {
  private ws: WebSocket | null = null;
  private deviceToken: string | null = null;
  private reconnectAttempts = 0;
  private maxReconnectAttempts = 10;
  private reconnectDelay = 1000; // Start with 1 second
  
  async connect(url: string, gatewayToken?: string): Promise<void> {
    this.ws = new WebSocket(url);
    
    this.ws.on('open', async () => {
      logger.info('WebSocket connected, sending handshake');
      await this.sendHandshake(gatewayToken);
    });
    
    this.ws.on('message', (data) => {
      this.handleMessage(data);
    });
    
    this.ws.on('close', () => {
      logger.warn('WebSocket closed, attempting reconnection');
      this.handleDisconnection();
    });
    
    this.ws.on('error', (error) => {
      logger.error('WebSocket error', error);
    });
  }
  
  private async sendHandshake(gatewayToken?: string): Promise<void> {
    const connectFrame = {
      type: 'connect',
      params: {
        auth: {
          token: this.deviceToken || gatewayToken
        }
      }
    };
    
    this.ws?.send(JSON.stringify(connectFrame));
  }
  
  private handleMessage(data: WebSocket.Data): void {
    try {
      const message = JSON.parse(data.toString());
      
      if (message.type === 'connect_ack') {
        // Store device token for future connections
        this.deviceToken = message.deviceToken;
        this.reconnectAttempts = 0; // Reset on successful connection
        logger.info('Handshake successful, device token received');
      } else if (message.type === 'event') {
        this.handleEvent(message);
      }
    } catch (error) {
      logger.error('Failed to parse message', error);
    }
  }
  
  private handleDisconnection(): void {
    if (this.reconnectAttempts >= this.maxReconnectAttempts) {
      logger.error('Max reconnection attempts reached, giving up');
      return;
    }
    
    this.reconnectAttempts++;
    const delay = this.reconnectDelay * Math.pow(2, this.reconnectAttempts - 1);
    
    logger.info(`Reconnecting in ${delay}ms (attempt ${this.reconnectAttempts})`);
    
    setTimeout(() => {
      this.connect(this.url, this.gatewayToken);
    }, delay);
  }
  
  private handleEvent(message: any): void {
    // Delegate to Observer
    observer.onGatewayEvent(message.event, message.payload);
  }
}
```

2. **Implement state recovery on reconnection:**

```typescript
class StateRecoveryManager {
  private lastKnownStateVersion: number = 0;
  
  async recoverState(): Promise<void> {
    logger.info('Recovering state after reconnection');
    
    // 1. Get latest snapshot from SQLite
    const lastSnapshot = await sqliteStore.getLatestSnapshot();
    
    // 2. Get latest events from SQLite
    const lastEvents = await sqliteStore.getEvents({
      limit: 100,
      orderBy: 'timestamp DESC'
    });
    
    // 3. Rebuild working state
    await this.rebuildWorkingState(lastSnapshot, lastEvents);
    
    // 4. Mark as recovered
    logger.info('State recovery complete');
  }
  
  private async rebuildWorkingState(
    snapshot: Snapshot | null,
    events: TierAEvent[]
  ): Promise<void> {
    if (!snapshot) {
      logger.warn('No snapshot found, starting fresh');
      return;
    }
    
    // Restore threads
    for (const thread of snapshot.active_threads) {
      await threadManager.upsertThread(thread);
    }
    
    // Restore facts
    for (const fact of snapshot.recent_facts) {
      await sqliteStore.insertFact(fact);
    }
    
    logger.info(`Restored ${snapshot.active_threads.length} threads and ${snapshot.recent_facts.length} facts`);
  }
}
```

3. **Add connection health monitoring:**

```typescript
class ConnectionHealthMonitor {
  private lastMessageTime: Date = new Date();
  private healthCheckInterval = 30000; // 30 seconds
  
  startMonitoring(): void {
    setInterval(() => {
      this.checkHealth();
    }, this.healthCheckInterval);
  }
  
  private checkHealth(): void {
    const now = new Date();
    const timeSinceLastMessage = now.getTime() - this.lastMessageTime.getTime();
    
    if (timeSinceLastMessage > this.healthCheckInterval * 2) {
      logger.warn('No messages received for 60 seconds, connection may be stale');
      // Trigger reconnection
      gatewayClient.reconnect();
    }
  }
  
  onMessageReceived(): void {
    this.lastMessageTime = new Date();
  }
}
```

---

### Q10: Event Schema

**Answer:**

**Event Types:**
- `agent` - Agent activity
- `chat` - Chat messages
- `presence` - User presence
- `health` - System health
- `heartbeat` - Periodic heartbeat
- `cron` - Scheduled tasks
- `tick` - Timer events
- `shutdown` - Gateway shutdown

**Payload Structure:**
```typescript
{
  type: "event",
  event: string,        // Event type (agent, chat, etc.)
  payload: any,         // Event-specific payload
  seq?: number,         // Optional sequence number
  stateVersion?: number // Optional state version
}
```

**Versioning:**
- Protocol includes optional `stateVersion` in event frame
- Defined using TypeBox schemas
- JSON Schema and Swift models generated for type safety

### Implications for Cognitive Controller

✅ **Clear Event Structure:**
- Well-defined event types
- Sequence numbers for ordering
- State version for consistency

⚠️ **Event Payload Schemas Unknown:**
- Need to discover payload structure for each event type
- Must handle unknown event types gracefully
- Must validate payloads before processing

### Action Items

1. **Define event type mappings:**

```typescript
enum GatewayEventType {
  AGENT = 'agent',
  CHAT = 'chat',
  PRESENCE = 'presence',
  HEALTH = 'health',
  HEARTBEAT = 'heartbeat',
  CRON = 'cron',
  TICK = 'tick',
  SHUTDOWN = 'shutdown'
}

interface GatewayEvent {
  type: 'event';
  event: GatewayEventType;
  payload: any;
  seq?: number;
  stateVersion?: number;
}
```

2. **Implement event router:**

```typescript
class EventRouter {
  private handlers: Map<GatewayEventType, EventHandler[]> = new Map();
  
  register(eventType: GatewayEventType, handler: EventHandler): void {
    if (!this.handlers.has(eventType)) {
      this.handlers.set(eventType, []);
    }
    this.handlers.get(eventType)!.push(handler);
  }
  
  async route(event: GatewayEvent): Promise<void> {
    const handlers = this.handlers.get(event.event);
    
    if (!handlers || handlers.length === 0) {
      logger.debug(`No handlers for event type: ${event.event}`);
      return;
    }
    
    for (const handler of handlers) {
      try {
        await handler(event.payload, event);
      } catch (error) {
        logger.error(`Handler failed for event ${event.event}`, error);
        // Continue with other handlers
      }
    }
  }
}

// Register our handlers
eventRouter.register(GatewayEventType.CHAT, async (payload) => {
  await observer.onUserMessage(payload.message, payload.metadata);
});

eventRouter.register(GatewayEventType.AGENT, async (payload) => {
  if (payload.type === 'tool_result') {
    await observer.onToolResult(payload.toolName, payload.params, payload.result);
  } else if (payload.type === 'state_transition') {
    await observer.onStateTransition(payload.from, payload.to, payload.reason);
  }
});

eventRouter.register(GatewayEventType.HEARTBEAT, async (payload) => {
  observer.emitHeartbeat();
});
```

3. **Add payload validation:**

```typescript
import Ajv from 'ajv';

const chatPayloadSchema = {
  type: 'object',
  properties: {
    message: { type: 'string' },
    metadata: { type: 'object' }
  },
  required: ['message']
};

function validatePayload(eventType: GatewayEventType, payload: any): boolean {
  const ajv = new Ajv();
  const schema = getSchemaForEventType(eventType);
  
  if (!schema) {
    logger.warn(`No schema defined for event type: ${eventType}`);
    return true; // Allow unknown event types
  }
  
  const validate = ajv.compile(schema);
  const valid = validate(payload);
  
  if (!valid) {
    logger.error(`Invalid payload for ${eventType}`, validate.errors);
    return false;
  }
  
  return true;
}
```

4. **Research needed: Discover actual payload schemas**
   - Monitor Gateway events in development
   - Log all event payloads
   - Build schema from observed data
   - Document in separate file

---

### Q11: Event Ordering & Reliability

**Answer:**

**Ordering:**
- Event frame includes optional `seq` (sequence number)
- Clients can track order of incoming messages

**Loss Policy:**
- **Events are NOT replayed**
- If client misses events due to network interruption, must perform full refresh
- No at-least-once or exactly-once guarantees

**Idempotency:**
- Side-effecting methods (send, agent) require idempotency keys
- Server maintains short-lived dedupe cache
- Allows safe retries

### Implications for Cognitive Controller

🔴 **Critical: No Delivery Guarantees**
- Events can be lost during disconnection
- Must handle gaps in sequence numbers
- Must implement own reliability layer

🔴 **Must Track Sequence Numbers**
- Detect gaps in sequence
- Trigger state recovery on gaps
- Log missing events

✅ **Idempotency Support**
- Can safely retry operations
- Server deduplicates

### Action Items

1. **Implement sequence tracking:**

```typescript
class SequenceTracker {
  private lastSeq: number = 0;
  private gapDetected: boolean = false;
  
  checkSequence(event: GatewayEvent): boolean {
    if (!event.seq) {
      // No sequence number, can't track
      return true;
    }
    
    if (this.lastSeq === 0) {
      // First event
      this.lastSeq = event.seq;
      return true;
    }
    
    const expectedSeq = this.lastSeq + 1;
    
    if (event.seq !== expectedSeq) {
      const gap = event.seq - expectedSeq;
      logger.warn(`Sequence gap detected: expected ${expectedSeq}, got ${event.seq} (gap: ${gap})`);
      this.gapDetected = true;
      this.lastSeq = event.seq;
      return false; // Gap detected
    }
    
    this.lastSeq = event.seq;
    this.gapDetected = false;
    return true;
  }
  
  hasGap(): boolean {
    return this.gapDetected;
  }
  
  reset(): void {
    this.lastSeq = 0;
    this.gapDetected = false;
  }
}

// Usage
const sequenceTracker = new SequenceTracker();

async function handleEvent(event: GatewayEvent): Promise<void> {
  const sequenceOk = sequenceTracker.checkSequence(event);
  
  if (!sequenceOk) {
    // Gap detected, trigger state recovery
    logger.warn('Triggering state recovery due to sequence gap');
    await stateRecoveryManager.recoverState();
  }
  
  // Process event normally
  await eventRouter.route(event);
}
```

2. **Implement idempotency for our operations:**

```typescript
class IdempotencyManager {
  private cache: Map<string, any> = new Map();
  private cacheExpiry = 300000; // 5 minutes
  
  async execute<T>(
    key: string,
    operation: () => Promise<T>
  ): Promise<T> {
    // Check cache
    if (this.cache.has(key)) {
      logger.debug(`Idempotency hit for key: ${key}`);
      return this.cache.get(key);
    }
    
    // Execute operation
    const result = await operation();
    
    // Cache result
    this.cache.set(key, result);
    
    // Set expiry
    setTimeout(() => {
      this.cache.delete(key);
    }, this.cacheExpiry);
    
    return result;
  }
}

// Usage
const idempotencyManager = new IdempotencyManager();

async function captureEvent(event: TierAEvent): Promise<void> {
  const key = `capture:${event.event_id}`;
  
  await idempotencyManager.execute(key, async () => {
    await sqliteStore.insertEvent(event);
  });
}
```

3. **Add gap recovery strategy:**

```typescript
class GapRecoveryStrategy {
  async recover(lastSeq: number, currentSeq: number): Promise<void> {
    const gap = currentSeq - lastSeq;
    
    logger.info(`Recovering from sequence gap: ${gap} events missed`);
    
    // Strategy 1: Full state refresh (safest)
    await stateRecoveryManager.recoverState();
    
    // Strategy 2: Partial recovery (if gap is small)
    if (gap < 10) {
      logger.info('Small gap, attempting partial recovery');
      await this.partialRecovery(lastSeq, currentSeq);
    }
  }
  
  private async partialRecovery(lastSeq: number, currentSeq: number): Promise<void> {
    // Try to infer what happened from SQLite state
    const recentEvents = await sqliteStore.getEvents({
      limit: 100,
      orderBy: 'timestamp DESC'
    });
    
    // Check if we have continuity in our local store
    const hasLocalContinuity = this.checkLocalContinuity(recentEvents);
    
    if (hasLocalContinuity) {
      logger.info('Local continuity maintained, no recovery needed');
    } else {
      logger.warn('Local continuity broken, performing full recovery');
      await stateRecoveryManager.recoverState();
    }
  }
}
```

---

### Q12: Event Delivery Latency

**Answer:**

**Latency:**
- No specific millisecond benchmarks provided
- System supports real-time streaming of tool output and assistant blocks

**Batching:**
- Block streaming tunable via `blockStreamingCoalesce`
- Merges consecutive chunks before sending
- Reduces "single-line spam"
- Configurable `idleMs` gaps

**Replay:**
- Not supported
- Clients must "refresh on gaps"

### Implications for Cognitive Controller

✅ **Real-Time Capable:**
- Low enough latency for real-time event capture
- Streaming support indicates good performance

⚠️ **Batching May Delay Events:**
- Events may be coalesced before delivery
- Must not assume instant delivery
- Must handle delayed events gracefully

### Action Items

1. **Measure actual latency in development:**

```typescript
class LatencyMonitor {
  private latencies: number[] = [];
  
  recordLatency(eventTimestamp: string): void {
    const now = Date.now();
    const eventTime = new Date(eventTimestamp).getTime();
    const latency = now - eventTime;
    
    this.latencies.push(latency);
    
    if (this.latencies.length > 1000) {
      this.latencies.shift(); // Keep last 1000
    }
    
    if (latency > 1000) {
      logger.warn(`High event latency detected: ${latency}ms`);
    }
  }
  
  getStats(): { p50: number; p95: number; p99: number } {
    const sorted = [...this.latencies].sort((a, b) => a - b);
    
    return {
      p50: sorted[Math.floor(sorted.length * 0.5)],
      p95: sorted[Math.floor(sorted.length * 0.95)],
      p99: sorted[Math.floor(sorted.length * 0.99)]
    };
  }
}
```

2. **Add latency alerts:**

```typescript
const LATENCY_THRESHOLD = 500; // 500ms

function checkLatency(latency: number): void {
  if (latency > LATENCY_THRESHOLD) {
    logger.warn(`Event latency exceeded threshold: ${latency}ms > ${LATENCY_THRESHOLD}ms`);
    
    // Update LED to amber (degraded performance)
    updateLED('amber');
  }
}
```

---

### Q13: Custom Events

**Answer:**

**Plugin Emission:**
- Documentation focuses on Gateway emitting core events to clients
- Plugins can register custom logic via JS/TS transform modules
- **No explicit API for plugins to emit arbitrary custom events**

### Implications for Cognitive Controller

⚠️ **Limited Custom Event Support:**
- Can't easily emit custom events for other plugins
- Must use Gateway's native event types
- Inter-plugin communication limited

✅ **Transform Modules Available:**
- Can alter how payloads are handled
- Can add custom logic to event processing

### Action Items

1. **Use native event types for our purposes:**

```typescript
// Instead of custom events, use existing event types with custom payloads
// Example: Use 'agent' event type with custom metadata

async function emitSnapshotCreated(snapshot: Snapshot): Promise<void> {
  // Log to our own system
  logger.info('Snapshot created', {
    snapshot_id: snapshot.handshake.snapshot_id,
    thread_count: snapshot.active_threads.length,
    token_count: estimateTokens(snapshot)
  });
  
  // If we need to notify other systems, use webhooks or file system
  await writeSnapshotMetadata(snapshot);
}
```

2. **Investigate transform modules for custom logic:**

```typescript
// ~/.openclaw/hooks/transforms/cognitive-controller.ts

export function transformAgentEvent(event: any): any {
  // Add our custom metadata to agent events
  if (event.type === 'tool_result') {
    event.cognitiveController = {
      snapshotId: getCurrentSnapshotId(),
      threadId: getCurrentThreadId(),
      confidence: getCurrentConfidence()
    };
  }
  
  return event;
}
```

---

### Q14: Backpressure & Slow Consumers

**Answer:**

**Queue Management:**
- If main queue is busy, scheduled tasks (heartbeats) are skipped and retried later
- System doesn't block on slow consumers

**Retry Mechanisms:**
- Subagent announce deliveries: retry up to 3 times if requester session busy
- Stale entries expire after 5 minutes to prevent infinite loops

**Rate Limiting:**
- Control-plane (RPC) rate-limited to 3 requests per 60 seconds per device
- Prevents flooding system with configuration updates

### Implications for Cognitive Controller

✅ **Gateway Handles Backpressure:**
- Won't block if we're slow
- Scheduled tasks skipped if we're busy

⚠️ **We May Miss Events:**
- If we're too slow, events may be skipped
- Must process events quickly (<100ms target)
- Must not block event loop

⚠️ **Rate Limiting on Control Plane:**
- Can't make frequent configuration updates
- Must batch configuration changes
- 3 requests per 60 seconds limit

### Action Items

1. **Ensure fast event processing:**

```typescript
const EVENT_PROCESSING_TIMEOUT = 100; // 100ms target

async function processEventWithTimeout(event: GatewayEvent): Promise<void> {
  const start = Date.now();
  
  try {
    await Promise.race([
      eventRouter.route(event),
      new Promise((_, reject) => 
        setTimeout(() => reject(new Error('Event processing timeout')), EVENT_PROCESSING_TIMEOUT)
      )
    ]);
  } catch (error) {
    if (error.message === 'Event processing timeout') {
      logger.warn(`Event processing exceeded ${EVENT_PROCESSING_TIMEOUT}ms`, { event });
    } else {
      logger.error('Event processing failed', error);
    }
  }
  
  const duration = Date.now() - start;
  
  if (duration > EVENT_PROCESSING_TIMEOUT) {
    logger.warn(`Slow event processing: ${duration}ms`, { event: event.event });
  }
}
```

2. **Add async processing queue:**

```typescript
import { Queue } from 'bull';

const eventQueue = new Queue('cognitive-controller-events', {
  redis: { host: 'localhost', port: 6379 }
});

// Fast path: Capture event immediately
async function captureEvent(event: GatewayEvent): Promise<void> {
  // Quick capture to SQLite (should be <10ms)
  await sqliteStore.insertEvent({
    event_id: generateEventId(),
    timestamp: new Date().toISOString(),
    event_type: event.event,
    payload: event.payload
  });
  
  // Queue for async processing
  await eventQueue.add('process', { event });
}

// Slow path: Process event asynchronously
eventQueue.process('process', async (job) => {
  const { event } = job.data;
  
  // Heavy processing (snapshot assembly, reranking, etc.)
  await processEventHeavy(event);
});
```

3. **Monitor processing performance:**

```typescript
class ProcessingMonitor {
  private processingTimes: number[] = [];
  
  recordProcessingTime(duration: number): void {
    this.processingTimes.push(duration);
    
    if (this.processingTimes.length > 1000) {
      this.processingTimes.shift();
    }
  }
  
  getStats(): { avg: number; p95: number; p99: number } {
    const sorted = [...this.processingTimes].sort((a, b) => a - b);
    const sum = sorted.reduce((a, b) => a + b, 0);
    
    return {
      avg: sum / sorted.length,
      p95: sorted[Math.floor(sorted.length * 0.95)],
      p99: sorted[Math.floor(sorted.length * 0.99)]
    };
  }
  
  isHealthy(): boolean {
    const stats = this.getStats();
    return stats.p95 < 100; // 95th percentile under 100ms
  }
}
```

4. **Respect rate limits:**

```typescript
class RateLimiter {
  private requests: Date[] = [];
  private limit = 3;
  private window = 60000; // 60 seconds
  
  async checkLimit(): Promise<boolean> {
    const now = new Date();
    
    // Remove requests outside window
    this.requests = this.requests.filter(
      req => now.getTime() - req.getTime() < this.window
    );
    
    if (this.requests.length >= this.limit) {
      logger.warn('Rate limit reached, request denied');
      return false;
    }
    
    this.requests.push(now);
    return true;
  }
}

// Usage
const rateLimiter = new RateLimiter();

async function updateConfiguration(config: any): Promise<void> {
  const allowed = await rateLimiter.checkLimit();
  
  if (!allowed) {
    throw new Error('Rate limit exceeded: 3 requests per 60 seconds');
  }
  
  // Proceed with configuration update
  await gateway.updateConfig(config);
}
```

---

## Summary: Gateway & Event System (Q9-Q14)

### Critical Findings

🔴 **No Event Replay** (Q9, Q11)
- Events lost during disconnection are NOT replayed
- Must implement state recovery on reconnection
- Must track sequence numbers to detect gaps

🔴 **Authentication Required** (Q9)
- Mandatory handshake with device token
- Must handle pairing flow
- Must store token securely

🔴 **No Delivery Guarantees** (Q11)
- Events can be lost
- No at-least-once or exactly-once semantics
- Must implement own reliability layer

⚠️ **Limited Custom Events** (Q13)
- Can't emit arbitrary custom events
- Must use native event types
- Inter-plugin communication limited

✅ **Good Performance** (Q12)
- Real-time streaming supported
- Low latency (exact numbers TBD)
- Backpressure handled by Gateway

✅ **Clear Event Structure** (Q10)
- Well-defined event types
- Sequence numbers available
- State versioning supported

### Implementation Priorities

**High Priority:**
1. Implement robust WebSocket client with reconnection
2. Implement state recovery on connection gaps
3. Implement sequence tracking and gap detection
4. Add authentication and device token management

**Medium Priority:**
5. Implement fast event processing (<100ms)
6. Add latency monitoring
7. Add processing performance monitoring
8. Respect rate limits on control plane

**Low Priority:**
9. Investigate transform modules for custom logic
10. Add idempotency for operations

### Next Research Priorities

**Q15-Q19:** Context & Memory Management
- How to inject snapshot before LLM calls
- Context window management strategy
- Compaction mechanism details

**Q20-Q27:** Memory Search & Indexing
- memory_search tool interface
- Vector indexing details
- Performance characteristics

---

**Document Updated:** 2026-03-03  
**Questions Answered:** Q1-Q14 (14/100)  
**Next Batch:** Q15-Q27 (Context & Memory)
