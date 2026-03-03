# OpenClaw Research Findings: Agents & Models (Q38-Q47)

**Date:** 2026-03-03  
**Source:** OpenClaw Documentation Database  
**Status:** Q38-Q47 Answered

---

## Agent Management & Multi-Agent Support (Q38-Q42)

### Q38: Agent Creation and Management

**Answer:**

**Creation:**
- CLI command: `openclaw agents add <id>`
- Triggers bootstrapping ritual
- Seeds essential files: `AGENTS.md`, `SOUL.md`
- Managed through `agents.list` block in global config

**Lifecycle:**
- Started: When Gateway routes inbound message via bindings
- No manual "pause" command
- Can disable in config or remove bindings

**Metadata:**
- Unique `id`
- Human-readable `name`
- `description`
- Identity: `vibe` and `emoji`

### Implications for Cognitive Controller

✅ **Per-Agent Deployment:**
- Each agent gets own Cognitive Controller instance
- Bootstrapping creates initial files
- Can seed with initial snapshot

⚠️ **No Pause Mechanism:**
- Can't temporarily pause agent
- Must disable or remove bindings

### Action Items

1. **Bootstrap Cognitive Controller for new agents:**

```typescript
// Hook into agent creation
async function onAgentCreated(agentId: string): Promise<void> {
  logger.info(`Bootstrapping Cognitive Controller for agent: ${agentId}`);
  
  const workspace = getAgentWorkspace(agentId);
  
  // Create directory structure
  await fs.mkdir(path.join(workspace, '.cognitive-controller'), { recursive: true });
  await fs.mkdir(path.join(workspace, 'memory', 'rollups'), { recursive: true });
  
  // Initialize SQLite database
  const dbPath = path.join(workspace, '.cognitive-controller', 'memory.db');
  await sqliteStore.initialize(dbPath);
  
  // Create initial bootstrap files
  await createBootstrapFiles(workspace);
  
  // Create initial snapshot
  const initialSnapshot = createInitialSnapshot(agentId);
  await snapshotAssembly.saveSnapshot(initialSnapshot);
  
  logger.info(`Cognitive Controller bootstrapped for agent: ${agentId}`);
}

async function createBootstrapFiles(workspace: string): Promise<void> {
  // COGNITIVE_CONTROLLER.md - Initial memory
  const ccPath = path.join(workspace, 'COGNITIVE_CONTROLLER.md');
  await fs.writeFile(ccPath, `# Cognitive Controller Memory

## Status
Initialized and ready.

## Constraints
- Preserve all constraints verbatim
- Never paraphrase hard rules
- Maintain bounded memory (1-10 snapshots per thread)
`, 'utf-8');

  // CONSTRAINTS.md - Hard constraints
  const constraintsPath = path.join(workspace, 'CONSTRAINTS.md');
  await fs.writeFile(constraintsPath, `# Hard Constraints

> These constraints must NEVER be paraphrased or modified.

- Cognitive Controller maintains bounded memory
- Snapshots are limited to 500 tokens
- All constraints are preserved verbatim
`, 'utf-8');
}

function createInitialSnapshot(agentId: string): Snapshot {
  return {
    handshake: {
      snapshot_id: generateUUID(),
      created_at: new Date().toISOString(),
      expires_at: new Date(Date.now() + 48 * 3600000).toISOString(),
      source_event_ids: [],
      schema_version: '1.0'
    },
    objective_now: 'Initialize and begin work',
    active_threads: [],
    recent_facts: [],
    hard_constraints: [
      'Preserve all constraints verbatim',
      'Never paraphrase hard rules',
      'Maintain bounded memory (1-10 snapshots per thread)'
    ],
    decisions: [],
    open_questions: [],
    tool_state: { available: [], unavailable: [], health: 'unknown' },
    last_actions: []
  };
}
```

2. **Manage agent lifecycle:**

```typescript
// Detect agent start
async function onAgentStarted(agentId: string): Promise<void> {
  logger.info(`Agent started: ${agentId}`);
  
  // Initialize Cognitive Controller for this agent
  await cognitiveController.initialize(agentId);
  
  // Load latest snapshot
  const snapshot = await snapshotAssembly.getLatestSnapshot(agentId);
  
  if (snapshot) {
    logger.info(`Loaded snapshot for agent ${agentId}: ${snapshot.handshake.snapshot_id}`);
  } else {
    logger.info(`No snapshot found for agent ${agentId}, creating initial snapshot`);
    await onAgentCreated(agentId);
  }
}

// Detect agent disabled
async function onAgentDisabled(agentId: string): Promise<void> {
  logger.info(`Agent disabled: ${agentId}`);
  
  // Create final snapshot
  await snapshotAssembly.createSnapshot({
    agentId,
    reason: 'agent_disabled'
  });
  
  // Trigger final roll-up
  await rollUpEngine.triggerRollUp('agent_disabled');
  
  // Flush buffers
  await observer.flushBuffer();
  
  logger.info(`Cognitive Controller cleaned up for agent: ${agentId}`);
}
```

---

### Q39: Agent Communication

**Answer:**

**Agent Send:**
- Explicit capability for agent coordination
- Must be enabled and allowlisted via `tools.agentToAgent`

**Delivery Guarantees:**
- Sub-agent coordination: retry up to 3 times if requester session busy
- Force-expire after 5 minutes

### Implications for Cognitive Controller

✅ **Explicit Coordination:**
- Agents can communicate if enabled
- Useful for Phase 4 (multi-agent)

⚠️ **Limited Guarantees:**
- 3 retries, then expire
- May lose messages

### Action Items

1. **Enable Agent Send (Phase 4+):**

```json5
{
  "tools": {
    "agentToAgent": {
      "enabled": true,
      "allowlist": [
        "agent-1",
        "agent-2",
        "agent-3"
      ]
    }
  }
}
```

2. **Implement inter-agent memory sharing (Phase 4+):**

```typescript
// Agent A shares memory with Agent B
async function shareMemoryWithAgent(
  fromAgentId: string,
  toAgentId: string,
  facts: AtomicFact[]
): Promise<void> {
  // Send facts via Agent Send
  await gateway.agentSend({
    from: fromAgentId,
    to: toAgentId,
    type: 'memory_share',
    payload: {
      facts: facts.map(f => ({
        content: f.content,
        timestamp: f.timestamp,
        source: fromAgentId
      }))
    }
  });
  
  logger.info(`Shared ${facts.length} facts from ${fromAgentId} to ${toAgentId}`);
}

// Agent B receives shared memory
async function onMemoryReceived(
  agentId: string,
  message: any
): Promise<void> {
  if (message.type === 'memory_share') {
    const facts = message.payload.facts;
    
    // Import facts into local store
    for (const fact of facts) {
      await sqliteStore.insertFact({
        fact_id: generateUUID(),
        timestamp: fact.timestamp,
        source_type: 'agent',
        source_id: message.from,
        content: fact.content,
        tags: ['shared', message.from],
        confidence: 0.8 // Slightly lower confidence for shared facts
      });
    }
    
    logger.info(`Imported ${facts.length} shared facts from ${message.from}`);
  }
}
```

---

### Q40: Agent Isolation

**Answer:**

**Workspaces:**
- Each agent has separate workspace directory
- Home for files and context

**Memory Indexes:**
- Isolated per agent
- Vector and BM25 indexes in separate SQLite files
- Path: `~/.openclaw/memory/<agentId>.sqlite`

**Session Stores:**
- Per-agent directories
- Path: `~/.openclaw/agents/<agentId>/sessions/`

### Implications for Cognitive Controller

✅ **Complete Isolation:**
- Our databases isolated per agent
- No cross-contamination
- Clean separation

✅ **Predictable Paths:**
- Can construct paths from agentId
- Easy to manage

### Action Items

1. **Use isolated paths:**

```typescript
class PathManager {
  private openClawHome: string;
  
  constructor() {
    this.openClawHome = path.join(os.homedir(), '.openclaw');
  }
  
  // Cognitive Controller paths (in workspace)
  getCognitiveControllerDir(agentId: string): string {
    const workspace = this.getWorkspace(agentId);
    return path.join(workspace, '.cognitive-controller');
  }
  
  getDatabasePath(agentId: string): string {
    return path.join(this.getCognitiveControllerDir(agentId), 'memory.db');
  }
  
  getRollUpDir(agentId: string): string {
    const workspace = this.getWorkspace(agentId);
    return path.join(workspace, 'memory', 'rollups');
  }
  
  // OpenClaw paths (outside workspace)
  getMemoryIndexPath(agentId: string): string {
    return path.join(this.openClawHome, 'memory', `${agentId}.sqlite`);
  }
  
  getSessionsDir(agentId: string): string {
    return path.join(this.openClawHome, 'agents', agentId, 'sessions');
  }
  
  getWorkspace(agentId: string): string {
    // Workspace path from config or default
    return path.join(this.openClawHome, 'agents', agentId, 'workspace');
  }
}

const pathManager = new PathManager();
```

2. **Verify isolation:**

```typescript
async function verifyAgentIsolation(agentId: string): Promise<boolean> {
  const dbPath = pathManager.getDatabasePath(agentId);
  const rollUpDir = pathManager.getRollUpDir(agentId);
  const sessionsDir = pathManager.getSessionsDir(agentId);
  
  // Check that paths are unique per agent
  const paths = [dbPath, rollUpDir, sessionsDir];
  
  for (const p of paths) {
    if (!p.includes(agentId)) {
      logger.error(`Path does not include agentId: ${p}`);
      return false;
    }
  }
  
  logger.info(`Agent isolation verified for: ${agentId}`);
  return true;
}
```

---

### Q41: Shared Resources

**Answer:**

**Shared Skills:**
- Workspace-specific skills per agent
- All agents load shared skills from `~/.openclaw/skills`

**Shared Memory:**
- Isolated by default
- Can share credentials if `auth-profiles.json` manually copied

**Shared Tools:**
- Core built-in tools shared
- Per-agent sandboxing can restrict access

### Implications for Cognitive Controller

✅ **Shared Skills:**
- Can deploy Cognitive Controller as shared skill
- All agents get access

⚠️ **Memory Isolation:**
- No automatic sharing
- Must implement explicitly (Phase 4+)

### Action Items

1. **Deploy as shared skill:**

```bash
# Install Cognitive Controller as shared skill
mkdir -p ~/.openclaw/skills/cognitive-controller
cp -r dist/* ~/.openclaw/skills/cognitive-controller/

# All agents now have access
```

2. **Configure per-agent:**

```json5
{
  "agents": {
    "list": [
      {
        "id": "agent-1",
        "skills": {
          "cognitive-controller": {
            "enabled": true,
            "snapshotTokenBudget": 500,
            "maxActiveThreads": 7
          }
        }
      },
      {
        "id": "agent-2",
        "skills": {
          "cognitive-controller": {
            "enabled": true,
            "snapshotTokenBudget": 300,  // Smaller budget
            "maxActiveThreads": 5
          }
        }
      }
    ]
  }
}
```

---

### Q42: Agent Concurrency Model

**Answer:**

**Simultaneous Execution:**
- Multiple agents can run simultaneously

**Resource Contention:**
- Managed globally by `agents.defaults.maxConcurrent`
- Prevents overloading host system

**Priority:**
- No traditional priority queue
- Cron jobs can be staggered or exact
- Gateway coalesces pending restarts for stability

### Implications for Cognitive Controller

✅ **Concurrent Support:**
- Must handle multiple agents simultaneously
- Thread-safe operations required

⚠️ **Resource Limits:**
- Must respect global concurrency
- Can't assume unlimited resources

### Action Items

1. **Implement thread-safe operations:**

```typescript
import { Mutex } from 'async-mutex';

class ThreadSafeCognitiveController {
  private mutexes: Map<string, Mutex> = new Map();
  
  private getMutex(agentId: string): Mutex {
    if (!this.mutexes.has(agentId)) {
      this.mutexes.set(agentId, new Mutex());
    }
    return this.mutexes.get(agentId)!;
  }
  
  async createSnapshot(agentId: string, context: any): Promise<Snapshot> {
    const mutex = this.getMutex(agentId);
    
    // Acquire lock for this agent
    const release = await mutex.acquire();
    
    try {
      // Create snapshot (thread-safe)
      const snapshot = await snapshotAssembly.createSnapshot(context);
      return snapshot;
    } finally {
      release();
    }
  }
  
  async captureEvent(agentId: string, event: TierAEvent): Promise<void> {
    const mutex = this.getMutex(agentId);
    const release = await mutex.acquire();
    
    try {
      await observer.captureEvent(event);
    } finally {
      release();
    }
  }
}
```

2. **Monitor resource usage:**

```typescript
class ResourceMonitor {
  private activeAgents: Set<string> = new Set();
  
  onAgentActive(agentId: string): void {
    this.activeAgents.add(agentId);
    logger.info(`Active agents: ${this.activeAgents.size}`);
  }
  
  onAgentIdle(agentId: string): void {
    this.activeAgents.delete(agentId);
    logger.info(`Active agents: ${this.activeAgents.size}`);
  }
  
  getActiveCount(): number {
    return this.activeAgents.size;
  }
  
  isOverloaded(maxConcurrent: number): boolean {
    return this.activeAgents.size >= maxConcurrent;
  }
}
```

---

## Model & LLM Integration (Q43-Q47)

### Q43: Model Configuration

**Answer:**

**Configuration Location:**
- Per-agent model config in `agents.list[].model`
- Global defaults in `agents.defaults.model`

**Hot Reload:**
- Model configuration changes are hot-reloaded
- No Gateway restart required for model changes

**Parameters:**
- `provider`: Model provider (e.g., "openai", "anthropic", "ollama")
- `model`: Model identifier (e.g., "gpt-4", "claude-3-opus")
- `temperature`: Sampling temperature
- `maxTokens`: Maximum response tokens
- `contextWindow`: Model's context window size

### Implications for Cognitive Controller

✅ **Dynamic Configuration:**
- Can adjust model settings without restart
- Useful for testing different models

✅ **Context Window Awareness:**
- Can read `contextWindow` to calculate compaction thresholds
- Adjust snapshot strategy based on model capacity

### Action Items

1. **Read model configuration:**

```typescript
interface ModelConfig {
  provider: string;
  model: string;
  temperature: number;
  maxTokens: number;
  contextWindow: number;
}

async function getModelConfig(agentId: string): Promise<ModelConfig> {
  const config = await gateway.getAgentConfig(agentId);
  return config.model;
}

async function calculateCompactionThreshold(agentId: string): Promise<number> {
  const modelConfig = await getModelConfig(agentId);
  const reserveTokens = 20000; // Default reserve
  const softThreshold = 4000;  // Default soft threshold
  
  const threshold = modelConfig.contextWindow - reserveTokens - softThreshold;
  
  logger.info(`Compaction threshold for ${agentId}: ${threshold} tokens`);
  return threshold;
}
```

2. **Adapt to model changes:**

```typescript
class ModelAwareSnapshotAssembly {
  private modelConfigs: Map<string, ModelConfig> = new Map();
  
  async onModelConfigChanged(agentId: string, newConfig: ModelConfig): Promise<void> {
    logger.info(`Model config changed for ${agentId}: ${newConfig.model}`);
    
    this.modelConfigs.set(agentId, newConfig);
    
    // Recalculate snapshot budget based on new context window
    const newBudget = this.calculateSnapshotBudget(newConfig.contextWindow);
    
    logger.info(`Adjusted snapshot budget for ${agentId}: ${newBudget} tokens`);
  }
  
  private calculateSnapshotBudget(contextWindow: number): number {
    // Snapshot should be ~1-2% of context window
    const budget = Math.floor(contextWindow * 0.015);
    
    // Clamp between 300-1000 tokens
    return Math.max(300, Math.min(1000, budget));
  }
}
```

---

### Q44: Model Fallback

**Answer:**

**Fallback Chain:**
- OpenClaw supports model fallback chains
- If primary model fails, tries secondary models in order

**Configuration:**
```json5
{
  "model": {
    "provider": "openai",
    "model": "gpt-4",
    "fallback": [
      { "provider": "openai", "model": "gpt-3.5-turbo" },
      { "provider": "anthropic", "model": "claude-3-sonnet" }
    ]
  }
}
```

**Trigger Conditions:**
- API errors (rate limits, timeouts)
- Model unavailability
- Quota exhaustion

### Implications for Cognitive Controller

✅ **Resilience:**
- Cognitive Controller continues working even if primary model fails
- Snapshots remain valid across model switches

⚠️ **Context Window Variation:**
- Fallback models may have different context windows
- Must recalculate thresholds on fallback

### Action Items

1. **Handle model fallback:**

```typescript
class FallbackAwareController {
  async onModelFallback(
    agentId: string,
    fromModel: string,
    toModel: string,
    reason: string
  ): Promise<void> {
    logger.warn(`Model fallback for ${agentId}: ${fromModel} → ${toModel} (${reason})`);
    
    // Get new model config
    const newConfig = await getModelConfig(agentId);
    
    // Recalculate compaction threshold
    const newThreshold = await calculateCompactionThreshold(agentId);
    
    // Check if current snapshot is still valid
    const currentSnapshot = await snapshotAssembly.getLatestSnapshot(agentId);
    
    if (currentSnapshot) {
      const snapshotTokens = estimateTokens(JSON.stringify(currentSnapshot));
      
      if (snapshotTokens > newConfig.contextWindow * 0.02) {
        logger.warn(`Snapshot too large for fallback model, creating compact version`);
        await this.createCompactSnapshot(agentId, currentSnapshot);
      }
    }
  }
  
  private async createCompactSnapshot(
    agentId: string,
    originalSnapshot: Snapshot
  ): Promise<Snapshot> {
    // Create more compact version by:
    // 1. Reducing recent_facts to top 5
    // 2. Keeping only active threads
    // 3. Trimming last_actions to 3
    
    const compactSnapshot: Snapshot = {
      ...originalSnapshot,
      recent_facts: originalSnapshot.recent_facts.slice(0, 5),
      active_threads: originalSnapshot.active_threads.filter(t => t.status === 'active'),
      last_actions: originalSnapshot.last_actions.slice(0, 3)
    };
    
    await snapshotAssembly.saveSnapshot(compactSnapshot);
    return compactSnapshot;
  }
}
```

---

### Q45: Token Estimation

**Answer:**

**Estimation Method:**
- OpenClaw uses `tiktoken` for token counting
- Model-specific tokenizers (e.g., `cl100k_base` for GPT-4)

**Accuracy:**
- Estimates are approximate but close to actual
- Used for compaction decisions

**Performance:**
- Fast enough for real-time use
- Cached for repeated content

### Implications for Cognitive Controller

✅ **Accurate Budgeting:**
- Can use same tokenizer for snapshot budgeting
- Ensures snapshots fit within budget

✅ **Real-time Validation:**
- Can validate snapshot size before injection

### Action Items

1. **Implement token estimation:**

```typescript
import { encoding_for_model } from 'tiktoken';

class TokenEstimator {
  private encoders: Map<string, any> = new Map();
  
  getEncoder(model: string): any {
    if (!this.encoders.has(model)) {
      try {
        const encoder = encoding_for_model(model as any);
        this.encoders.set(model, encoder);
      } catch (error) {
        // Fallback to cl100k_base for unknown models
        const encoder = encoding_for_model('gpt-4');
        this.encoders.set(model, encoder);
      }
    }
    return this.encoders.get(model)!;
  }
  
  estimateTokens(text: string, model: string): number {
    const encoder = this.getEncoder(model);
    const tokens = encoder.encode(text);
    return tokens.length;
  }
  
  validateSnapshotBudget(
    snapshot: Snapshot,
    budget: number,
    model: string
  ): { valid: boolean; actual: number; budget: number } {
    const snapshotText = JSON.stringify(snapshot, null, 2);
    const actual = this.estimateTokens(snapshotText, model);
    
    return {
      valid: actual <= budget,
      actual,
      budget
    };
  }
}

const tokenEstimator = new TokenEstimator();
```

2. **Enforce budget during snapshot creation:**

```typescript
async function createBudgetedSnapshot(
  context: any,
  budget: number,
  model: string
): Promise<Snapshot> {
  let snapshot = await snapshotAssembly.createSnapshot(context);
  
  // Validate budget
  let validation = tokenEstimator.validateSnapshotBudget(snapshot, budget, model);
  
  if (!validation.valid) {
    logger.warn(`Snapshot exceeds budget: ${validation.actual}/${validation.budget} tokens`);
    
    // Trim snapshot to fit budget
    snapshot = await trimSnapshotToBudget(snapshot, budget, model);
    
    // Re-validate
    validation = tokenEstimator.validateSnapshotBudget(snapshot, budget, model);
    
    if (!validation.valid) {
      throw new Error(`Unable to fit snapshot within budget: ${validation.actual}/${validation.budget}`);
    }
  }
  
  logger.info(`Snapshot within budget: ${validation.actual}/${validation.budget} tokens`);
  return snapshot;
}

async function trimSnapshotToBudget(
  snapshot: Snapshot,
  budget: number,
  model: string
): Promise<Snapshot> {
  // Iteratively trim fields until within budget
  let trimmed = { ...snapshot };
  
  // Priority: trim recent_facts, then last_actions, then active_threads
  while (tokenEstimator.estimateTokens(JSON.stringify(trimmed), model) > budget) {
    if (trimmed.recent_facts.length > 3) {
      trimmed.recent_facts = trimmed.recent_facts.slice(0, -1);
    } else if (trimmed.last_actions.length > 2) {
      trimmed.last_actions = trimmed.last_actions.slice(0, -1);
    } else if (trimmed.active_threads.length > 1) {
      trimmed.active_threads = trimmed.active_threads.slice(0, -1);
    } else {
      break; // Can't trim further
    }
  }
  
  return trimmed;
}
```

---

### Q46: Streaming Responses

**Answer:**

**Streaming Support:**
- OpenClaw supports streaming LLM responses
- Tokens arrive incrementally via WebSocket events

**Event Format:**
```json
{
  "type": "agent",
  "subtype": "chunk",
  "agentId": "agent-1",
  "sessionId": "session-123",
  "chunk": "partial response text"
}
```

**Completion Signal:**
- Final event with `subtype: "done"`
- Includes full response and metadata

### Implications for Cognitive Controller

✅ **Real-time Observation:**
- Can observe response as it's generated
- Useful for early fact extraction

⚠️ **Partial Data:**
- Must buffer chunks until completion
- Can't make decisions on partial responses

### Action Items

1. **Handle streaming events:**

```typescript
class StreamingObserver {
  private buffers: Map<string, string[]> = new Map();
  
  onChunk(event: any): void {
    const key = `${event.agentId}:${event.sessionId}`;
    
    if (!this.buffers.has(key)) {
      this.buffers.set(key, []);
    }
    
    this.buffers.get(key)!.push(event.chunk);
    
    // Optional: Early fact extraction for high-confidence patterns
    this.tryEarlyExtraction(event.chunk);
  }
  
  onDone(event: any): void {
    const key = `${event.agentId}:${event.sessionId}`;
    const chunks = this.buffers.get(key) || [];
    const fullResponse = chunks.join('');
    
    // Process complete response
    this.processCompleteResponse(event.agentId, fullResponse, event.metadata);
    
    // Clear buffer
    this.buffers.delete(key);
  }
  
  private tryEarlyExtraction(chunk: string): void {
    // Extract high-confidence facts from partial response
    // Example: "The user's name is Alice" → extract immediately
    
    const patterns = [
      /The user's name is (\w+)/,
      /The deadline is (\d{4}-\d{2}-\d{2})/,
      /The priority is (high|medium|low)/
    ];
    
    for (const pattern of patterns) {
      const match = chunk.match(pattern);
      if (match) {
        logger.info(`Early fact extraction: ${match[0]}`);
        // Store fact immediately (with lower confidence)
      }
    }
  }
  
  private async processCompleteResponse(
    agentId: string,
    response: string,
    metadata: any
  ): Promise<void> {
    // Extract facts from complete response
    const facts = await factExtractor.extract(response);
    
    // Store in SQLite
    for (const fact of facts) {
      await sqliteStore.insertFact({
        fact_id: generateUUID(),
        timestamp: new Date().toISOString(),
        source_type: 'assistant',
        source_id: metadata.messageId,
        content: fact.content,
        tags: fact.tags,
        confidence: fact.confidence
      });
    }
    
    logger.info(`Extracted ${facts.length} facts from response`);
  }
}
```

---

### Q47: Context Window Exhaustion

**Answer:**

**Behavior:**
- When context window is exhausted, OpenClaw triggers compaction
- Runs "memory flush" turn before trimming
- Agent prompted to write durable memories to disk

**Memory Flush Prompt:**
```
The context window is nearly full. Please write any important facts, 
decisions, or state to your memory files (MEMORY.md, USER.md, etc.) 
before the context is compacted.
```

**Post-Compaction:**
- Older turns removed from context
- System prompt and current request preserved
- Agent continues with fresh context

### Implications for Cognitive Controller

✅ **Hook Point Identified:**
- Memory flush is THE moment to create snapshot
- Agent is explicitly prompted to preserve state
- Perfect timing for snapshot creation

✅ **Guaranteed Execution:**
- Flush happens before every compaction
- Reliable trigger for snapshot creation

### Action Items

1. **Hook into memory flush:**

```typescript
class MemoryFlushHook {
  async onMemoryFlush(agentId: string, sessionId: string): Promise<void> {
    logger.info(`Memory flush triggered for ${agentId}:${sessionId}`);
    
    // This is THE moment to create snapshot
    const context = await this.gatherContext(agentId, sessionId);
    
    // Create snapshot
    const snapshot = await snapshotAssembly.createSnapshot(context);
    
    // Save to SQLite
    await sqliteStore.saveSnapshot(snapshot);
    
    // Write to roll-up directory for indexing
    await this.writeRollUp(agentId, snapshot);
    
    logger.info(`Snapshot created during memory flush: ${snapshot.handshake.snapshot_id}`);
  }
  
  private async gatherContext(agentId: string, sessionId: string): Promise<any> {
    // Read current session transcript
    const transcript = await gateway.getSessionTranscript(sessionId);
    
    // Read memory files
    const workspace = pathManager.getWorkspace(agentId);
    const memoryMd = await fs.readFile(path.join(workspace, 'MEMORY.md'), 'utf-8');
    const userMd = await fs.readFile(path.join(workspace, 'USER.md'), 'utf-8');
    
    // Read existing snapshots
    const existingSnapshots = await sqliteStore.getRecentSnapshots(agentId, 5);
    
    return {
      agentId,
      sessionId,
      transcript,
      memoryMd,
      userMd,
      existingSnapshots,
      timestamp: new Date().toISOString()
    };
  }
  
  private async writeRollUp(agentId: string, snapshot: Snapshot): Promise<void> {
    const rollUpDir = pathManager.getRollUpDir(agentId);
    const filename = `${snapshot.handshake.snapshot_id}.canonical.md`;
    const filepath = path.join(rollUpDir, filename);
    
    // Convert snapshot to Markdown
    const markdown = snapshotToMarkdown(snapshot);
    
    // Write to roll-up directory
    await fs.writeFile(filepath, markdown, 'utf-8');
    
    logger.info(`Roll-up written: ${filepath}`);
  }
}

function snapshotToMarkdown(snapshot: Snapshot): string {
  return `# Snapshot ${snapshot.handshake.snapshot_id}

**Created:** ${snapshot.handshake.created_at}  
**Expires:** ${snapshot.handshake.expires_at}

## Objective
${snapshot.objective_now}

## Active Threads
${snapshot.active_threads.map(t => `- [${t.status}] ${t.thread_id}: ${t.summary}`).join('\n')}

## Recent Facts
${snapshot.recent_facts.map(f => `- ${f.content} (${f.timestamp})`).join('\n')}

## Hard Constraints
${snapshot.hard_constraints.map(c => `- ${c}`).join('\n')}

## Decisions
${snapshot.decisions.map(d => `- ${d.decision} (${d.rationale})`).join('\n')}

## Open Questions
${snapshot.open_questions.map(q => `- ${q}`).join('\n')}

## Tool State
- Available: ${snapshot.tool_state.available.join(', ')}
- Unavailable: ${snapshot.tool_state.unavailable.join(', ')}
- Health: ${snapshot.tool_state.health}

## Last Actions
${snapshot.last_actions.map(a => `- ${a.action} (${a.outcome})`).join('\n')}
`;
}
```

2. **Register hook with OpenClaw:**

```typescript
// In plugin initialization
export async function initialize(gateway: Gateway): Promise<void> {
  const memoryFlushHook = new MemoryFlushHook();
  
  // Subscribe to memory flush events
  gateway.on('memory_flush', async (event) => {
    await memoryFlushHook.onMemoryFlush(event.agentId, event.sessionId);
  });
  
  logger.info('Cognitive Controller memory flush hook registered');
}
```

---

## Summary: Q38-Q47 Key Findings

### Agent Management
- ✅ Per-agent deployment with bootstrap ritual
- ✅ Complete isolation (workspace, memory, sessions)
- ✅ Shared skills from `~/.openclaw/skills`
- ✅ Agent Send for explicit coordination (Phase 4+)
- ⚠️ No pause mechanism (must disable or remove bindings)
- ⚠️ Thread-safe operations required for concurrent agents

### Model Integration
- ✅ Hot-reload model configuration (no restart)
- ✅ Context window awareness for compaction
- ✅ Model fallback chains for resilience
- ✅ Token estimation via `tiktoken`
- ✅ Streaming responses for real-time observation
- ✅ **Memory flush hook = perfect snapshot trigger**

### Critical Implementation Priorities

1. **Memory Flush Hook (HIGHEST PRIORITY)**
   - This is THE moment to create snapshots
   - Guaranteed execution before every compaction
   - Agent explicitly prompted to preserve state

2. **Per-Agent Isolation**
   - Use isolated paths for databases and roll-ups
   - Verify isolation on initialization

3. **Thread-Safe Operations**
   - Use mutexes for concurrent agent support
   - Monitor resource usage

4. **Token Budget Enforcement**
   - Use `tiktoken` for accurate estimation
   - Trim snapshots to fit budget
   - Adapt to model changes and fallbacks

5. **Bootstrap Integration**
   - Seed initial snapshot on agent creation
   - Create directory structure and initial files

---

**Next Steps:**
- Continue with Q48-Q100 as answers become available
- Implement memory flush hook prototype
- Test per-agent isolation
- Validate token budgeting with real models

