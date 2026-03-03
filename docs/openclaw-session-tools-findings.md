# OpenClaw Research Findings: Session & Tools (Q28-Q37)

**Date:** 2026-03-03  
**Source:** OpenClaw Documentation Database  
**Status:** Q28-Q37 Answered

---

## Session Store (Q28-Q31)

### Q28: Session Persistence

**Answer:**

**Storage Format:**
- JSONL (JSON Lines) transcripts
- One JSON object per line (one turn per line)

**Storage Location:**
- Outside agent workspace
- Path: `~/.openclaw/agents/<agentId>/sessions/`

**Retention Policy:**
- Main sessions: Durable (kept indefinitely)
- Cron jobs: Configurable `sessionRetention` (e.g., "24h")
- "Session Pruning" mechanism for long-term management

### Implications for Cognitive Controller

✅ **Durable Storage:**
- Sessions persist across Gateway restarts
- Can access historical sessions

✅ **Separate from Workspace:**
- Won't interfere with agent files
- Clean separation of concerns

⚠️ **Outside Our Control:**
- Can't directly write to session store
- Must use OpenClaw's APIs

### Action Items

1. **Access session transcripts:**

```typescript
import fs from 'fs/promises';
import path from 'path';
import os from 'os';

async function getSessionPath(agentId: string, sessionId: string): Promise<string> {
  const openClawHome = path.join(os.homedir(), '.openclaw');
  return path.join(openClawHome, 'agents', agentId, 'sessions', `${sessionId}.jsonl`);
}

async function readSessionTranscript(
  agentId: string,
  sessionId: string
): Promise<SessionTurn[]> {
  const sessionPath = await getSessionPath(agentId, sessionId);
  const content = await fs.readFile(sessionPath, 'utf-8');
  
  const turns: SessionTurn[] = [];
  
  for (const line of content.split('\n')) {
    if (!line.trim()) continue;
    
    try {
      const turn = JSON.parse(line);
      turns.push(turn);
    } catch (error) {
      logger.error('Failed to parse session line', error);
    }
  }
  
  return turns;
}
```

2. **Monitor session retention:**

```typescript
async function listSessions(agentId: string): Promise<string[]> {
  const sessionsDir = path.join(
    os.homedir(),
    '.openclaw',
    'agents',
    agentId,
    'sessions'
  );
  
  const files = await fs.readdir(sessionsDir);
  return files.filter(f => f.endsWith('.jsonl'));
}

async function pruneOldSessions(
  agentId: string,
  retentionDays: number = 30
): Promise<number> {
  const sessions = await listSessions(agentId);
  const now = Date.now();
  const retentionMs = retentionDays * 24 * 60 * 60 * 1000;
  
  let pruned = 0;
  
  for (const sessionFile of sessions) {
    const sessionPath = path.join(
      os.homedir(),
      '.openclaw',
      'agents',
      agentId,
      'sessions',
      sessionFile
    );
    
    const stats = await fs.stat(sessionPath);
    const age = now - stats.mtimeMs;
    
    if (age > retentionMs) {
      logger.info(`Pruning old session: ${sessionFile} (age: ${Math.floor(age / (24 * 60 * 60 * 1000))} days)`);
      await fs.unlink(sessionPath);
      pruned++;
    }
  }
  
  return pruned;
}
```

---

### Q29: Session Content

**Answer:**

**Full History:**
- Yes, transcripts contain full conversation history

**Tool Results:**
- User, Assistant, and Tool turns captured
- Execution results included

**Agent State:**
- Internal metadata tracked
- Model references included
- Optional separate reasoning blocks

### Implications for Cognitive Controller

✅ **Rich Session Data:**
- Can extract facts from tool results
- Can analyze conversation patterns
- Can track agent reasoning

✅ **Complete History:**
- No data loss in sessions
- Can reconstruct any point in time

### Action Items

1. **Extract facts from session:**

```typescript
interface SessionTurn {
  seq: number;
  timestamp: string;
  role: 'user' | 'assistant' | 'tool';
  content: string;
  metadata?: {
    toolName?: string;
    toolParams?: any;
    modelRef?: string;
    reasoning?: string;
  };
}

async function extractFactsFromSession(
  agentId: string,
  sessionId: string
): Promise<AtomicFact[]> {
  const turns = await readSessionTranscript(agentId, sessionId);
  const facts: AtomicFact[] = [];
  
  for (const turn of turns) {
    if (turn.role === 'tool' && turn.metadata?.toolName) {
      // Extract fact from tool result
      facts.push({
        fact_id: generateUUID(),
        timestamp: turn.timestamp,
        source_type: 'tool',
        source_id: `session:${sessionId}:${turn.seq}`,
        content: `Tool ${turn.metadata.toolName} returned: ${turn.content}`,
        tags: [turn.metadata.toolName],
        confidence: 1.0
      });
    } else if (turn.role === 'user') {
      // Extract fact from user message
      facts.push({
        fact_id: generateUUID(),
        timestamp: turn.timestamp,
        source_type: 'user',
        source_id: `session:${sessionId}:${turn.seq}`,
        content: turn.content,
        tags: ['user-input'],
        confidence: 1.0
      });
    }
  }
  
  return facts;
}
```

2. **Analyze conversation patterns:**

```typescript
interface ConversationStats {
  totalTurns: number;
  userTurns: number;
  assistantTurns: number;
  toolTurns: number;
  averageTurnLength: number;
  toolsUsed: string[];
  duration: number; // milliseconds
}

async function analyzeSession(
  agentId: string,
  sessionId: string
): Promise<ConversationStats> {
  const turns = await readSessionTranscript(agentId, sessionId);
  
  const stats: ConversationStats = {
    totalTurns: turns.length,
    userTurns: turns.filter(t => t.role === 'user').length,
    assistantTurns: turns.filter(t => t.role === 'assistant').length,
    toolTurns: turns.filter(t => t.role === 'tool').length,
    averageTurnLength: 0,
    toolsUsed: [],
    duration: 0
  };
  
  // Calculate average turn length
  const totalLength = turns.reduce((sum, t) => sum + t.content.length, 0);
  stats.averageTurnLength = totalLength / turns.length;
  
  // Extract tools used
  const toolsSet = new Set<string>();
  for (const turn of turns) {
    if (turn.role === 'tool' && turn.metadata?.toolName) {
      toolsSet.add(turn.metadata.toolName);
    }
  }
  stats.toolsUsed = Array.from(toolsSet);
  
  // Calculate duration
  if (turns.length > 0) {
    const start = new Date(turns[0].timestamp).getTime();
    const end = new Date(turns[turns.length - 1].timestamp).getTime();
    stats.duration = end - start;
  }
  
  return stats;
}
```

---

### Q30: Plugin Access to Session Data

**Answer:**

**Access Type:**
- Read/write/spawn capabilities
- Permissions: `sessions_list`, `sessions_history`, `sessions_send`, `sessions_spawn`, `session_status`

**API:**
- Sessions CLI
- Gateway RPC calls
- `memory_search` tool (if session indexing enabled)

**Metadata:**
- Sequence numbers (`seq`)
- Timestamps
- Role-based payload data

### Implications for Cognitive Controller

✅ **Rich API:**
- Can list sessions
- Can read history
- Can query via memory_search

⚠️ **Permission-Based:**
- Need appropriate permissions
- May need to configure tool policy

### Action Items

1. **Use sessions API:**

```typescript
// Via Gateway RPC
async function listAgentSessions(agentId: string): Promise<SessionInfo[]> {
  const response = await gateway.rpc('sessions_list', {
    agentId
  });
  
  return response.sessions;
}

async function getSessionHistory(
  agentId: string,
  sessionId: string
): Promise<SessionTurn[]> {
  const response = await gateway.rpc('sessions_history', {
    agentId,
    sessionId
  });
  
  return response.turns;
}
```

2. **Query sessions via memory_search:**

```typescript
async function searchSessionHistory(
  query: string,
  agentId: string
): Promise<MemorySearchResult[]> {
  // Requires session indexing to be enabled
  const results = await memorySearch({
    query,
    filePattern: `sessions/${agentId}/*.jsonl`,
    maxResults: 20
  });
  
  return results;
}
```

3. **Configure tool policy for session access:**

```json5
{
  "agents": {
    "defaults": {
      "toolPolicy": {
        "allow": [
          "sessions_list",
          "sessions_history",
          "session_status"
        ]
      }
    }
  }
}
```

---

### Q31: Session Isolation

**Answer:**

**Isolation:**
- Sessions fully isolated per agent
- Dedicated session store directory per agent

**Sharing:**
- No cross-talk by default
- Requires explicit mechanisms (e.g., "Agent Send" tool)

**Queries:**
- Cross-agent queries not standard
- Agents are "fully scoped brains" with separate state

### Implications for Cognitive Controller

✅ **Clean Isolation:**
- Our snapshots/events isolated per agent
- No cross-contamination

⚠️ **Multi-Agent Coordination:**
- Phase 4+ feature
- Need explicit sharing mechanism

### Action Items

1. **Ensure per-agent isolation:**

```typescript
function getDatabasePath(agentId: string): string {
  const workspace = getAgentWorkspace(agentId);
  return path.join(workspace, '.cognitive-controller', 'memory.db');
}

function getRollUpPath(agentId: string): string {
  const workspace = getAgentWorkspace(agentId);
  return path.join(workspace, 'memory', 'rollups');
}

// Each agent gets its own database and roll-up directory
```

2. **Plan for multi-agent coordination (Phase 4+):**

```typescript
// Future: Shared memory pool
interface SharedMemoryPool {
  agentIds: string[];
  sharedFacts: AtomicFact[];
  sharedDecisions: Decision[];
}

async function createSharedMemoryPool(agentIds: string[]): Promise<SharedMemoryPool> {
  // Aggregate facts from multiple agents
  const allFacts: AtomicFact[] = [];
  
  for (const agentId of agentIds) {
    const agentFacts = await sqliteStore.getFacts({ agentId });
    allFacts.push(...agentFacts);
  }
  
  // Deduplicate and merge
  const sharedFacts = deduplicateFacts(allFacts);
  
  return {
    agentIds,
    sharedFacts,
    sharedDecisions: []
  };
}
```

---

## Tool Execution (Q32-Q37)

### Q32: Custom Tool Registration

**Answer:**

**API:**
- Tools loaded from `skills/` directory in workspace
- Also from managed/bundled folders

**Schema:**
- TypeBox for protocol schemas
- Tool Policy defines allowed/denied actions

**Async Support:**
- Tools can be async or long-running
- Subject to timeout constraints
- Can run in isolated Docker containers

### Implications for Cognitive Controller

✅ **Flexible Tool System:**
- Can register custom tools if needed
- Async support for long operations

⚠️ **Tool Policy Required:**
- Must configure allow/deny lists
- Security considerations

### Action Items

1. **Register custom tool (if needed):**

```typescript
// skills/cognitive-controller-query.ts

export const tool = {
  name: 'cognitive_controller_query',
  description: 'Query Cognitive Controller memory and state',
  parameters: {
    type: 'object',
    properties: {
      query: {
        type: 'string',
        description: 'Search query for facts, decisions, or threads'
      },
      type: {
        type: 'string',
        enum: ['facts', 'decisions', 'threads', 'all'],
        description: 'Type of data to query'
      }
    },
    required: ['query']
  },
  
  async execute(params: { query: string; type?: string }): Promise<any> {
    const type = params.type || 'all';
    
    switch (type) {
      case 'facts':
        return await sqliteStore.searchFacts({ query: params.query });
      case 'decisions':
        return await sqliteStore.searchDecisions({ query: params.query });
      case 'threads':
        return await threadManager.searchThreads(params.query);
      case 'all':
        return await hybridSearch(params.query);
      default:
        throw new Error(`Unknown type: ${type}`);
    }
  }
};
```

2. **Configure tool policy:**

```json5
{
  "agents": {
    "defaults": {
      "toolPolicy": {
        "allow": [
          "cognitive_controller_query",
          "memory_search",
          "sessions_history"
        ],
        "deny": [
          // Deny dangerous operations
          "exec",
          "process"
        ]
      }
    }
  }
}
```

---

### Q33: Tool Execution Lifecycle

**Answer:**

**Pre-execution:**
- Tool Policy check (allow/deny lists)
- Authentication checks

**Timeout:**
- Default execution limit
- Overridable per run via `timeoutSeconds`

**Post-execution:**
- Results logged to transcript
- Verbose tool summaries
- Real-time output streaming to Control UI

### Implications for Cognitive Controller

✅ **Comprehensive Lifecycle:**
- Pre-execution validation
- Timeout protection
- Post-execution logging

✅ **Streaming Support:**
- Real-time output for long operations
- Good for user feedback

### Action Items

1. **Implement tool with timeout:**

```typescript
export const tool = {
  name: 'cognitive_controller_snapshot',
  description: 'Create a snapshot of current memory state',
  parameters: {
    type: 'object',
    properties: {
      reason: {
        type: 'string',
        description: 'Reason for creating snapshot'
      },
      timeoutSeconds: {
        type: 'number',
        description: 'Timeout in seconds (default: 30)',
        default: 30
      }
    }
  },
  
  async execute(params: { reason?: string; timeoutSeconds?: number }): Promise<any> {
    const timeout = params.timeoutSeconds || 30;
    
    // Create snapshot with timeout
    const snapshot = await Promise.race([
      snapshotAssembly.createSnapshot({ reason: params.reason }),
      new Promise((_, reject) => 
        setTimeout(() => reject(new Error('Snapshot creation timeout')), timeout * 1000)
      )
    ]);
    
    return {
      success: true,
      snapshot_id: snapshot.handshake.snapshot_id,
      thread_count: snapshot.active_threads.length,
      token_count: estimateTokens(snapshot)
    };
  }
};
```

2. **Add streaming output:**

```typescript
export const tool = {
  name: 'cognitive_controller_rollup',
  description: 'Generate a roll-up of recent memory',
  
  async execute(params: any, context: ToolContext): Promise<any> {
    // Stream progress updates
    context.stream('Starting roll-up generation...\n');
    
    const events = await sqliteStore.getEvents({ limit: 1000 });
    context.stream(`Loaded ${events.length} events\n`);
    
    const rollUp = await rollUpEngine.generateCanonicalRollUp({ events });
    context.stream('Roll-up generated\n');
    
    await rollUpEngine.writeRollUp(rollUp);
    context.stream('Roll-up written to disk\n');
    
    return {
      success: true,
      event_count: events.length,
      filename: rollUp.filename
    };
  }
};
```

---

### Q34: Tool Error Handling

**Answer:**

**Format:**
- Structured tool results
- Example: "Skipped due to queued user message"

**Retry Logic:**
- Gateway implements Retry Policy
- Channel adapters (e.g., Telegram) have configurable retry settings

**Propagation:**
- Errors enqueued and shown to agent in context
- Agent can react to failure in next turn

### Implications for Cognitive Controller

✅ **Structured Errors:**
- Clear error format
- Easy to parse and handle

✅ **Retry Support:**
- Automatic retries for transient failures
- Configurable per channel

✅ **Agent Awareness:**
- Agent sees errors in context
- Can adapt behavior

### Action Items

1. **Return structured errors:**

```typescript
export const tool = {
  name: 'cognitive_controller_query',
  
  async execute(params: any): Promise<any> {
    try {
      const results = await sqliteStore.searchFacts(params);
      
      return {
        success: true,
        results,
        count: results.length
      };
    } catch (error) {
      return {
        success: false,
        error: {
          type: error.name,
          message: error.message,
          retryable: isRetryableError(error)
        }
      };
    }
  }
};

function isRetryableError(error: Error): boolean {
  // Database locked, network timeout, etc.
  return error.message.includes('SQLITE_BUSY') ||
         error.message.includes('timeout') ||
         error.message.includes('ECONNREFUSED');
}
```

2. **Configure retry policy:**

```json5
{
  "agents": {
    "defaults": {
      "retryPolicy": {
        "maxAttempts": 3,
        "backoffMs": 1000,
        "backoffMultiplier": 2
      }
    }
  }
}
```

---

### Q35: Tool Execution Constraints

**Answer:**

**Concurrency:**
- Global: `agents.defaults.maxConcurrent`
- Cron: `maxConcurrentRuns` limit

**Memory:**
- Sandboxed Docker containers: hard limits (e.g., "1g" memory, "2g" swap)

**Network:**
- Sandboxed tools: no network by default (`network: "none"`)
- Must explicitly allow

### Implications for Cognitive Controller

✅ **Concurrency Control:**
- Prevents resource exhaustion
- Configurable limits

⚠️ **Sandboxing:**
- If enabled, strict memory/network limits
- Must design for constrained environment

### Action Items

1. **Configure concurrency:**

```json5
{
  "agents": {
    "defaults": {
      "maxConcurrent": 5,  // Max 5 concurrent tool executions
      "toolPolicy": {
        "sandboxing": {
          "enabled": false,  // Disable for development
          "memory": "1g",
          "swap": "2g",
          "network": "none"
        }
      }
    }
  }
}
```

2. **Design for sandboxed environment:**

```typescript
// Check if running in sandbox
function isSandboxed(): boolean {
  return process.env.OPENCLAW_SANDBOXED === 'true';
}

// Adapt behavior for sandbox
async function initializeStorage(): Promise<void> {
  if (isSandboxed()) {
    // Use in-memory storage
    logger.info('Running in sandbox, using in-memory storage');
    return initializeInMemoryStorage();
  } else {
    // Use SQLite
    logger.info('Running without sandbox, using SQLite');
    return initializeSQLiteStorage();
  }
}
```

---

### Q36: Tool Result Formatting

**Answer:**

**Format:**
- JSON envelopes (e.g., Lobster tool)
- Text or media in session transcript

**Size Limits:**
- Responses can be capped
- Example: `ackMaxChars` (default 300) for heartbeat acknowledgments

**Streaming:**
- Most tools return final result
- Control UI can stream tool output via agent events

### Implications for Cognitive Controller

✅ **Flexible Formatting:**
- JSON for structured data
- Text for human-readable output

⚠️ **Size Limits:**
- Must respect caps
- Large results may be truncated

### Action Items

1. **Format tool results:**

```typescript
export const tool = {
  name: 'cognitive_controller_status',
  
  async execute(): Promise<any> {
    const status = await getSystemStatus();
    
    // Return structured JSON
    return {
      healthy: status.healthy,
      threads: {
        active: status.activeThreads,
        stalled: status.stalledThreads,
        total: status.totalThreads
      },
      memory: {
        snapshots: status.snapshotCount,
        events: status.eventCount,
        dbSizeMB: status.dbSizeMB
      },
      lastHeartbeat: status.lastHeartbeat
    };
  }
};
```

2. **Handle size limits:**

```typescript
function truncateResult(result: any, maxChars: number = 1000): any {
  const json = JSON.stringify(result);
  
  if (json.length <= maxChars) {
    return result;
  }
  
  // Truncate and add indicator
  const truncated = json.substring(0, maxChars - 50);
  return {
    ...result,
    _truncated: true,
    _originalSize: json.length,
    _message: `Result truncated to ${maxChars} characters`
  };
}
```

---

### Q37: Agent Context and State

**Answer:**

**Context Access:**
- `sessions_history` tool allows reading past turns

**State Modification:**
- Tools can write to memory files (`MEMORY.md`, `USER.md`)
- Updates to workspace files
- Agent reads at start of every session

**Triggering Tools:**
- Lobster tool: coordinates multi-step pipelines
- Human approval gates

### Implications for Cognitive Controller

✅ **Full Context Access:**
- Can read conversation history
- Can modify agent state

✅ **File-Based State:**
- Write to memory files for persistence
- Agent picks up changes automatically

### Action Items

1. **Read agent context:**

```typescript
export const tool = {
  name: 'cognitive_controller_context',
  description: 'Get current agent context from Cognitive Controller',
  
  async execute(): Promise<any> {
    const snapshot = await snapshotAssembly.getLatestSnapshot();
    const recentEvents = await sqliteStore.getEvents({ limit: 10 });
    
    return {
      objective: snapshot.objective_now,
      activeThreads: snapshot.active_threads.map(t => ({
        title: t.title,
        status: t.status,
        confidence: t.confidence,
        nextStep: t.next_step
      })),
      recentEvents: recentEvents.map(e => ({
        type: e.event_type,
        timestamp: e.timestamp,
        content: e.content.substring(0, 100) // Truncate
      })),
      constraints: snapshot.hard_constraints
    };
  }
};
```

2. **Modify agent state:**

```typescript
export const tool = {
  name: 'cognitive_controller_update_constraint',
  description: 'Add or update a hard constraint',
  parameters: {
    type: 'object',
    properties: {
      constraint: {
        type: 'string',
        description: 'The constraint to add (verbatim)'
      }
    },
    required: ['constraint']
  },
  
  async execute(params: { constraint: string }): Promise<any> {
    // Add to SQLite
    await sqliteStore.insertConstraint(params.constraint);
    
    // Write to CONSTRAINTS.md for persistence
    const workspace = getWorkspace();
    const constraintsPath = path.join(workspace, 'CONSTRAINTS.md');
    
    const existing = await fs.readFile(constraintsPath, 'utf-8').catch(() => '');
    const updated = existing + `\n- ${params.constraint}`;
    
    await fs.writeFile(constraintsPath, updated, 'utf-8');
    
    return {
      success: true,
      constraint: params.constraint,
      message: 'Constraint added and will be preserved across sessions'
    };
  }
};
```

---

## Summary: Session & Tools (Q28-Q37)

### Critical Findings

✅ **Rich Session Data** (Q28-Q31)
- JSONL transcripts with full history
- Stored in `~/.openclaw/agents/<agentId>/sessions/`
- Complete isolation per agent
- Accessible via API and memory_search

✅ **Flexible Tool System** (Q32-Q37)
- Custom tools via skills directory
- Async and long-running support
- Structured error handling
- Streaming output
- Context access and state modification

⚠️ **Constraints** (Q35)
- Concurrency limits (configurable)
- Sandboxing: memory/network restrictions
- Size limits on results

### Implementation Priorities

**High Priority:**
1. Access session transcripts for fact extraction (Q28, Q29)
2. Register custom tools for querying Cognitive Controller (Q32)
3. Implement structured error handling (Q34)

**Medium Priority:**
4. Add streaming output for long operations (Q33, Q36)
5. Configure tool policy and concurrency (Q32, Q35)
6. Implement context access tools (Q37)

**Low Priority:**
7. Session pruning automation (Q28)
8. Multi-agent coordination planning (Q31)
9. Sandboxed environment support (Q35)

### Integration Strategy

**Session Access:**
- Read transcripts from `~/.openclaw/agents/<agentId>/sessions/`
- Extract facts from tool results and user messages
- Use sessions_history API for programmatic access

**Tool Registration:**
- Create custom tools in skills directory
- Expose Cognitive Controller functionality to agents
- Implement structured JSON responses

**State Management:**
- Write to memory files (CONSTRAINTS.md, MEMORY.md)
- Agent picks up changes automatically
- Persist critical data across sessions

---

**Document Owner:** Project Team  
**Status:** Q28-Q37 Answered (37/100 total)  
**Next Batch:** Q38-Q50 (Agent Management, Model Integration, Error Handling)
