# OpenClaw Research Findings: Context & Memory (Q15-Q27)

**Date:** 2026-03-03  
**Source:** OpenClaw Documentation Database  
**Status:** Q15-Q27 Answered

---

## Context Injection (Q15-Q19)

### Q15: Plugin Context Injection

**Answer:**

**Method:**
- No explicit `before_llm_call` hook documented for plugins
- OpenClaw injects bootstrap files into agent context on first turn:
  - `AGENTS.md`
  - `SOUL.md`
  - `USER.md`
  - Other bootstrap files
- Custom logic via JS/TS transform modules in `~/.openclaw/hooks/transforms`

**Placement:**
- System prompt (rules/instructions)
- Appended to user prompt verbatim

**Token Budget:**
- Per-file limit: `agents.defaults.bootstrapMaxChars` (default: 20,000)
- Total limit: `agents.defaults.bootstrapTotalMaxChars` (default: 150,000)

### Implications for Cognitive Controller

✅ **Clear Injection Mechanism:**
- Can use bootstrap files for snapshot injection
- 150,000 char total budget is generous (our 500 token snapshot = ~2,000 chars)

⚠️ **First Turn Only:**
- Bootstrap files only injected on first turn
- Need different mechanism for ongoing injection

⚠️ **Transform Modules:**
- Can use transforms for per-turn injection
- Need to investigate transform API

### Action Items

1. **Create bootstrap file for initial snapshot:**

```typescript
// Write snapshot to bootstrap file
async function writeSnapshotBootstrap(snapshot: Snapshot): Promise<void> {
  const workspace = getWorkspace();
  const bootstrapPath = path.join(workspace, 'COGNITIVE_CONTROLLER.md');
  
  const content = `# Cognitive Controller Memory

## Current Objective
${snapshot.objective_now}

## Active Threads
${snapshot.active_threads.map(t => `
### ${t.title}
- Status: ${t.status}
- Confidence: ${t.confidence.toFixed(2)}
- Next: ${t.next_step}
${t.blocked_by.length > 0 ? `- Blocked by: ${t.blocked_by.join(', ')}` : ''}
`).join('\n')}

## Hard Constraints
${snapshot.hard_constraints.map(c => `- ${c}`).join('\n')}

## Recent Facts
${snapshot.recent_facts.map(f => `- [${f.timestamp}] ${f.content}`).join('\n')}
`;

  await fs.writeFile(bootstrapPath, content, 'utf-8');
}
```

2. **Investigate transform modules for per-turn injection:**

```typescript
// ~/.openclaw/hooks/transforms/cognitive-controller.ts

export async function transformUserMessage(message: any): Promise<any> {
  // Get latest snapshot
  const snapshot = await snapshotAssembly.getLatestSnapshot();
  
  // Inject snapshot into message context
  message.context = {
    ...message.context,
    cognitiveController: {
      objective: snapshot.objective_now,
      activeThreads: snapshot.active_threads.length,
      confidence: calculateAverageConfidence(snapshot.active_threads)
    }
  };
  
  return message;
}
```

3. **Monitor token budget usage:**

```typescript
function estimateBootstrapSize(snapshot: Snapshot): number {
  const content = formatSnapshotForBootstrap(snapshot);
  return content.length; // Characters, not tokens
}

async function checkBootstrapBudget(snapshot: Snapshot): Promise<boolean> {
  const size = estimateBootstrapSize(snapshot);
  const maxSize = 20000; // bootstrapMaxChars
  
  if (size > maxSize) {
    logger.warn(`Snapshot too large for bootstrap: ${size} > ${maxSize}`);
    return false;
  }
  
  return true;
}
```

---

### Q16: Context Window Management Strategy

**Answer:**

**Decision Criteria:**
- Compaction triggered based on token estimate relative to model's context window

**Memory Flush Trigger:**
- Formula: `contextWindow - reserveTokensFloor - softThresholdTokens`
- Default soft threshold: 4,000 tokens
- Triggers "memory flush" turn before compaction

**Plugin Influence:**
- Plugins not mentioned as influencing timing
- Compaction cycle tunable globally in configuration

### Implications for Cognitive Controller

✅ **Predictable Compaction:**
- Clear formula for when compaction occurs
- Can calculate when our snapshot will be affected

🔴 **Critical: Memory Flush Before Compaction:**
- Agent prompted to write durable memories before trim
- **This is our opportunity to create snapshot!**
- Must hook into memory flush turn

⚠️ **Can't Prevent Compaction:**
- No plugin influence on timing
- Must work with compaction, not against it

### Action Items

1. **Hook into memory flush turn:**

```typescript
// Detect memory flush turn
function isMemoryFlushTurn(event: GatewayEvent): boolean {
  // Memory flush is a silent turn before compaction
  return event.payload?.type === 'memory_flush' || 
         event.payload?.reason === 'compaction_imminent';
}

// Handle memory flush
async function onMemoryFlush(event: GatewayEvent): Promise<void> {
  logger.info('Memory flush detected, creating snapshot');
  
  // Create snapshot immediately
  const snapshot = await snapshotAssembly.createSnapshot({
    reason: 'pre_compaction',
    urgent: true
  });
  
  // Write to bootstrap file for next session
  await writeSnapshotBootstrap(snapshot);
  
  // Trigger roll-up
  await rollUpEngine.triggerRollUp('compaction');
}
```

2. **Calculate compaction threshold:**

```typescript
interface CompactionThreshold {
  contextWindow: number;
  reserveTokensFloor: number;
  softThresholdTokens: number;
  flushThreshold: number;
}

function calculateCompactionThreshold(config: any): CompactionThreshold {
  const contextWindow = config.model.contextWindow || 128000;
  const reserveTokensFloor = config.compaction?.reserveTokensFloor || 20000;
  const softThresholdTokens = config.compaction?.softThresholdTokens || 4000;
  
  const flushThreshold = contextWindow - reserveTokensFloor - softThresholdTokens;
  
  return {
    contextWindow,
    reserveTokensFloor,
    softThresholdTokens,
    flushThreshold
  };
}

// Monitor context usage
function estimateContextUsage(events: TierAEvent[]): number {
  // Rough estimate: sum of all event content lengths / 4
  const totalChars = events.reduce((sum, e) => sum + e.content.length, 0);
  return Math.ceil(totalChars / 4); // ~4 chars per token
}

async function checkCompactionImminent(): Promise<boolean> {
  const threshold = calculateCompactionThreshold(config);
  const currentUsage = estimateContextUsage(recentEvents);
  
  const remaining = threshold.flushThreshold - currentUsage;
  
  if (remaining < 5000) {
    logger.warn(`Compaction imminent: ${remaining} tokens remaining`);
    return true;
  }
  
  return false;
}
```

3. **Proactive snapshot creation:**

```typescript
// Create snapshot before compaction is triggered
setInterval(async () => {
  const imminent = await checkCompactionImminent();
  
  if (imminent) {
    logger.info('Proactively creating snapshot before compaction');
    await snapshotAssembly.createSnapshot({
      reason: 'proactive_pre_compaction'
    });
  }
}, 60000); // Check every minute
```

---

### Q17: Context Structure

**Answer:**

**Representation:**
- Session transcripts stored as JSONL files
- Each line is a JSON object representing a turn

**Metadata:**
- Sequence numbers (`seq`)
- Timestamps
- Role-based payload data (User/Assistant/Tool)

### Implications for Cognitive Controller

✅ **Structured Format:**
- JSONL is easy to parse
- Can read session history if needed

✅ **Metadata Available:**
- Sequence numbers for ordering
- Timestamps for temporal analysis
- Role information for context

### Action Items

1. **Parse session transcripts if needed:**

```typescript
import fs from 'fs/promises';
import readline from 'readline';

async function parseSessionTranscript(sessionPath: string): Promise<SessionTurn[]> {
  const turns: SessionTurn[] = [];
  
  const fileStream = fs.createReadStream(sessionPath);
  const rl = readline.createInterface({
    input: fileStream,
    crlfDelay: Infinity
  });
  
  for await (const line of rl) {
    try {
      const turn = JSON.parse(line);
      turns.push({
        seq: turn.seq,
        timestamp: turn.timestamp,
        role: turn.role,
        content: turn.content,
        metadata: turn.metadata
      });
    } catch (error) {
      logger.error('Failed to parse transcript line', error);
    }
  }
  
  return turns;
}
```

2. **Extract facts from session history:**

```typescript
async function extractFactsFromSession(sessionPath: string): Promise<AtomicFact[]> {
  const turns = await parseSessionTranscript(sessionPath);
  const facts: AtomicFact[] = [];
  
  for (const turn of turns) {
    if (turn.role === 'tool' && turn.metadata?.toolName) {
      // Extract fact from tool result
      facts.push({
        fact_id: generateUUID(),
        timestamp: turn.timestamp,
        source_type: 'tool',
        source_id: `session:${turn.seq}`,
        content: `Tool ${turn.metadata.toolName} returned: ${turn.content}`,
        tags: [turn.metadata.toolName],
        confidence: 1.0
      });
    }
  }
  
  return facts;
}
```

---

### Q18: Compaction Mechanism

**Answer:**

**Method:**
- Agentic process (LLM-based)
- Silent "memory flush" turn before compaction
- Model prompted to write durable memories to disk

**Preservation:**
- System prompt preserved
- Current user request preserved
- Flush turn designed to prevent fact loss

### Implications for Cognitive Controller

✅ **Agentic = Intelligent:**
- LLM decides what's important
- Better than rule-based trimming

🔴 **Risk: Paraphrasing:**
- LLM may paraphrase constraints during flush
- **This is exactly the "JPEG artifacts" problem we solve!**
- Our verbatim constraints prevent this

✅ **Flush Turn = Opportunity:**
- Can inject our snapshot during flush
- Can ensure constraints are preserved verbatim

### Action Items

1. **Inject constraints during memory flush:**

```typescript
async function injectConstraintsDuringFlush(): Promise<void> {
  const constraints = await sqliteStore.getConstraints();
  
  // Write constraints to a file that won't be compacted
  const constraintsPath = path.join(workspace, 'CONSTRAINTS.md');
  
  const content = `# Hard Constraints

> These constraints must NEVER be paraphrased or modified.

${constraints.map(c => `- ${c}`).join('\n')}
`;

  await fs.writeFile(constraintsPath, content, 'utf-8');
  
  logger.info(`Preserved ${constraints.length} constraints during memory flush`);
}
```

2. **Monitor compaction events:**

```typescript
eventRouter.register(GatewayEventType.AGENT, async (payload) => {
  if (payload.type === 'compaction_started') {
    logger.info('Compaction started');
    await onCompactionStarted();
  } else if (payload.type === 'compaction_completed') {
    logger.info('Compaction completed');
    await onCompactionCompleted();
  }
});

async function onCompactionStarted(): Promise<void> {
  // Create final snapshot before compaction
  await snapshotAssembly.createSnapshot({ reason: 'compaction_started' });
  
  // Inject constraints
  await injectConstraintsDuringFlush();
}

async function onCompactionCompleted(): Promise<void> {
  // Verify constraints survived
  const constraintsPath = path.join(workspace, 'CONSTRAINTS.md');
  const exists = await fs.access(constraintsPath).then(() => true).catch(() => false);
  
  if (!exists) {
    logger.error('Constraints file missing after compaction!');
    // Recreate it
    await injectConstraintsDuringFlush();
  }
}
```

---

### Q19: Reserve Tokens Floor

**Answer:**

**Default:**
- 20,000 tokens

**Configurability:**
- Per-agent via `agents.defaults.compaction.reserveTokensFloor`

**Exhaustion:**
- Compaction strictly enforced when reserve reached
- Ensures agent always has room to generate response

### Implications for Cognitive Controller

✅ **Generous Reserve:**
- 20,000 tokens is substantial
- Our 500 token snapshot fits comfortably

✅ **Configurable:**
- Can adjust per agent if needed
- Can increase if our snapshots grow

⚠️ **Strict Enforcement:**
- When reserve hit, compaction is mandatory
- Must ensure snapshot is created before this point

### Action Items

1. **Monitor reserve usage:**

```typescript
function calculateRemainingReserve(
  contextUsage: number,
  contextWindow: number,
  reserveFloor: number
): number {
  return contextWindow - contextUsage - reserveFloor;
}

async function checkReserveStatus(): Promise<void> {
  const config = await loadConfig();
  const contextWindow = config.model.contextWindow || 128000;
  const reserveFloor = config.compaction?.reserveTokensFloor || 20000;
  const currentUsage = estimateContextUsage(recentEvents);
  
  const remaining = calculateRemainingReserve(currentUsage, contextWindow, reserveFloor);
  
  if (remaining < 1000) {
    logger.error(`Reserve nearly exhausted: ${remaining} tokens remaining`);
    // Emergency snapshot
    await snapshotAssembly.createSnapshot({ reason: 'reserve_exhausted', urgent: true });
  } else if (remaining < 5000) {
    logger.warn(`Reserve running low: ${remaining} tokens remaining`);
  }
}
```

2. **Configure reserve per agent:**

```json5
{
  "agents": {
    "defaults": {
      "compaction": {
        "reserveTokensFloor": 25000,  // Increase if needed
        "softThresholdTokens": 4000
      }
    }
  }
}
```

---

## Memory Search & Indexing (Q20-Q27)

### Q20: memory_search Tool Interface

**Answer:**

**Signature:**
- Provides semantic recall over indexed snippets

**Return Format:**
- Snippet text (capped at ~700 characters)
- File path
- Line range
- Similarity score
- Model used

### Implications for Cognitive Controller

✅ **Rich Return Format:**
- Snippet text for context
- File path for source tracing
- Similarity score for ranking
- Line range for precise location

✅ **Integration Opportunity:**
- Can use memory_search to query our roll-ups
- Semantic search over canonical facts
- Complements our structured queries

### Action Items

1. **Define memory_search interface:**

```typescript
interface MemorySearchResult {
  snippet: string;        // ~700 char max
  filePath: string;       // Relative to workspace
  lineRange: [number, number];
  similarityScore: number; // 0.0 - 1.0
  model: string;          // Embedding model used
}

interface MemorySearchParams {
  query: string;
  maxResults?: number;
  minScore?: number;
  filePattern?: string;   // Filter by file pattern
}

async function memorySearch(params: MemorySearchParams): Promise<MemorySearchResult[]> {
  // Call OpenClaw's memory_search tool
  const results = await gateway.callTool('memory_search', {
    query: params.query,
    maxResults: params.maxResults || 10
  });
  
  return results.filter(r => r.similarityScore >= (params.minScore || 0.5));
}
```

2. **Use memory_search for semantic queries:**

```typescript
// Example: Find relevant decisions
async function findRelevantDecisions(context: string): Promise<Decision[]> {
  const results = await memorySearch({
    query: `decisions related to: ${context}`,
    filePattern: 'memory/rollups/*.canonical.md',
    minScore: 0.7
  });
  
  // Parse decisions from snippets
  const decisions: Decision[] = [];
  
  for (const result of results) {
    const decision = parseDecisionFromSnippet(result.snippet);
    if (decision) {
      decisions.push(decision);
    }
  }
  
  return decisions;
}
```

3. **Combine structured + semantic search:**

```typescript
async function hybridSearch(query: string): Promise<SearchResults> {
  // Structured search (our SQLite)
  const structuredResults = await sqliteStore.searchFacts({
    query,
    limit: 10
  });
  
  // Semantic search (OpenClaw's memory_search)
  const semanticResults = await memorySearch({
    query,
    maxResults: 10,
    minScore: 0.6
  });
  
  // Merge and deduplicate
  return mergeSearchResults(structuredResults, semanticResults);
}
```

---

### Q21: Vector Indexing

**Answer:**

**Default Model:**
- `embeddinggemma-300m-qat-Q8_0.gguf` (~0.6 GB)
- Local embedding model

**Frequency:**
- Real-time file watcher on `MEMORY.md` and `memory/` directory
- 1.5s debounce before marking index "dirty"
- Background sync triggered

### Implications for Cognitive Controller

✅ **Local Embeddings:**
- No external API required
- Privacy-preserving
- Fast

✅ **Real-Time Indexing:**
- Our roll-ups indexed automatically
- 1.5s latency acceptable
- Background sync doesn't block

⚠️ **File Watcher Dependency:**
- Must write roll-ups to watched directories
- Must use `.md` extension
- Must respect debounce timing

### Action Items

1. **Write roll-ups to indexed directory:**

```typescript
async function writeRollUpForIndexing(rollUp: string): Promise<void> {
  const workspace = getWorkspace();
  const rollUpDir = path.join(workspace, 'memory', 'rollups');
  
  // Ensure directory exists
  await fs.mkdir(rollUpDir, { recursive: true });
  
  // Generate filename
  const filename = `${new Date().toISOString().split('T')[0]}-${Date.now()}.canonical.md`;
  const filePath = path.join(rollUpDir, filename);
  
  // Write file
  await fs.writeFile(filePath, rollUp, 'utf-8');
  
  logger.info(`Roll-up written to indexed directory: ${filePath}`);
  
  // Wait for debounce + indexing
  await sleep(2000); // 1.5s debounce + 0.5s buffer
  
  logger.info('Roll-up should now be indexed and searchable');
}
```

2. **Verify indexing:**

```typescript
async function verifyRollUpIndexed(filename: string): Promise<boolean> {
  // Search for a unique phrase from the roll-up
  const uniquePhrase = `Roll-up: ${filename}`;
  
  const results = await memorySearch({
    query: uniquePhrase,
    maxResults: 1
  });
  
  const indexed = results.length > 0 && results[0].filePath.includes(filename);
  
  if (!indexed) {
    logger.warn(`Roll-up not indexed yet: ${filename}`);
  }
  
  return indexed;
}
```

---

### Q22: BM25 Implementation

**Answer:**

**Strategy:**
- SQLite FTS5 (Full-Text Search 5)

**Scoring:**
- Formula: `textScore = 1 / (1 + max(0, bm25Rank))`
- Converts BM25 rank to normalized score

### Implications for Cognitive Controller

✅ **SQLite FTS5:**
- Fast keyword search
- Built into SQLite (no extra dependencies)
- Complements vector search

✅ **Normalized Scoring:**
- Scores between 0 and 1
- Easy to combine with vector scores

### Action Items

1. **Add FTS5 to our SQLite schema:**

```sql
-- Create FTS5 virtual table for facts
CREATE VIRTUAL TABLE facts_fts USING fts5(
  fact_id UNINDEXED,
  content,
  tags,
  tokenize = 'porter unicode61'
);

-- Populate from facts table
INSERT INTO facts_fts (fact_id, content, tags)
SELECT fact_id, content, json_extract(tags, '$') FROM facts;

-- Trigger to keep FTS5 in sync
CREATE TRIGGER facts_fts_insert AFTER INSERT ON facts BEGIN
  INSERT INTO facts_fts (fact_id, content, tags)
  VALUES (new.fact_id, new.content, json_extract(new.tags, '$'));
END;
```

2. **Implement BM25 search:**

```typescript
async function bm25Search(query: string, limit: number = 10): Promise<SearchResult[]> {
  const sql = `
    SELECT 
      f.fact_id,
      f.content,
      f.timestamp,
      (1.0 / (1.0 + max(0, fts.rank))) as score
    FROM facts_fts fts
    JOIN facts f ON f.fact_id = fts.fact_id
    WHERE facts_fts MATCH ?
    ORDER BY fts.rank
    LIMIT ?
  `;
  
  const results = db.prepare(sql).all(query, limit);
  
  return results.map(r => ({
    factId: r.fact_id,
    content: r.content,
    timestamp: r.timestamp,
    score: r.score,
    source: 'bm25'
  }));
}
```

3. **Combine BM25 + vector search:**

```typescript
async function hybridFactSearch(query: string): Promise<SearchResult[]> {
  // BM25 keyword search (our SQLite)
  const bm25Results = await bm25Search(query, 20);
  
  // Vector semantic search (OpenClaw)
  const vectorResults = await memorySearch({
    query,
    filePattern: 'memory/rollups/*.md',
    maxResults: 20
  });
  
  // Merge with weighted scores
  const weights = { bm25: 0.3, vector: 0.7 };
  
  return mergeAndRankResults(bm25Results, vectorResults, weights);
}
```

---

### Q23: Hybrid Search Weighting

**Answer:**

**Weights:**
- Default: 0.7 for Vector similarity, 0.3 for BM25 keyword relevance

**MMR (Maximal Marginal Relevance):**
- Can be enabled for diversity
- Iteratively re-ranks results
- Configurable lambda (default: 0.7)
- Balances relevance vs diversity

### Implications for Cognitive Controller

✅ **Clear Weighting:**
- Vector prioritized (0.7) over keyword (0.3)
- Matches our needs (semantic > exact match)

✅ **MMR Available:**
- Can enable for diverse results
- Prevents redundant facts

### Action Items

1. **Implement weighted hybrid search:**

```typescript
interface HybridSearchConfig {
  vectorWeight: number;  // Default: 0.7
  bm25Weight: number;    // Default: 0.3
  enableMMR: boolean;    // Default: false
  mmrLambda: number;     // Default: 0.7
}

async function weightedHybridSearch(
  query: string,
  config: HybridSearchConfig = {
    vectorWeight: 0.7,
    bm25Weight: 0.3,
    enableMMR: false,
    mmrLambda: 0.7
  }
): Promise<SearchResult[]> {
  const bm25Results = await bm25Search(query);
  const vectorResults = await vectorSearch(query);
  
  // Combine scores
  const combined = combineResults(bm25Results, vectorResults, config);
  
  // Apply MMR if enabled
  if (config.enableMMR) {
    return applyMMR(combined, config.mmrLambda);
  }
  
  return combined;
}

function combineResults(
  bm25: SearchResult[],
  vector: SearchResult[],
  config: HybridSearchConfig
): SearchResult[] {
  const resultMap = new Map<string, SearchResult>();
  
  // Add BM25 results
  for (const result of bm25) {
    resultMap.set(result.id, {
      ...result,
      combinedScore: result.score * config.bm25Weight
    });
  }
  
  // Add/merge vector results
  for (const result of vector) {
    const existing = resultMap.get(result.id);
    if (existing) {
      existing.combinedScore += result.score * config.vectorWeight;
    } else {
      resultMap.set(result.id, {
        ...result,
        combinedScore: result.score * config.vectorWeight
      });
    }
  }
  
  // Sort by combined score
  return Array.from(resultMap.values())
    .sort((a, b) => b.combinedScore - a.combinedScore);
}
```

2. **Implement MMR:**

```typescript
function applyMMR(
  results: SearchResult[],
  lambda: number = 0.7
): SearchResult[]  {
  const selected: SearchResult[] = [];
  const remaining = [...results];
  
  // Select first result (highest score)
  if (remaining.length > 0) {
    selected.push(remaining.shift()!);
  }
  
  // Iteratively select diverse results
  while (remaining.length > 0 && selected.length < 10) {
    let maxScore = -Infinity;
    let maxIndex = -1;
    
    for (let i = 0; i < remaining.length; i++) {
      const candidate = remaining[i];
      
      // Relevance score
      const relevance = candidate.combinedScore;
      
      // Diversity score (min similarity to selected)
      const diversity = Math.min(
        ...selected.map(s => 1 - cosineSimilarity(candidate, s))
      );
      
      // MMR score
      const mmrScore = lambda * relevance + (1 - lambda) * diversity;
      
      if (mmrScore > maxScore) {
        maxScore = mmrScore;
        maxIndex = i;
      }
    }
    
    if (maxIndex >= 0) {
      selected.push(remaining.splice(maxIndex, 1)[0]);
    }
  }
  
  return selected;
}
```

---

### Q24: Temporal Decay Formula

**Answer:**

**Recency Boost:**
- Exponential multiplier: `decayedScore = score × e^(-λ × ageInDays)`

**Decay Rate:**
- Default half-life: 30 days
- Memory's score halves every month

### Implications for Cognitive Controller

✅ **Recency Bias:**
- Recent memories prioritized
- Aligns with our design (recent facts in snapshot)

✅ **Configurable:**
- Can adjust half-life if needed
- Can disable for certain queries

### Action Items

1. **Implement temporal decay:**

```typescript
function applyTemporalDecay(
  score: number,
  timestamp: string,
  halfLifeDays: number = 30
): number {
  const now = new Date();
  const eventTime = new Date(timestamp);
  const ageInDays = (now.getTime() - eventTime.getTime()) / (1000 * 60 * 60 * 24);
  
  // Calculate decay constant from half-life
  const lambda = Math.log(2) / halfLifeDays;
  
  // Apply exponential decay
  const decayedScore = score * Math.exp(-lambda * ageInDays);
  
  return decayedScore;
}
```

2. **Apply to search results:**

```typescript
async function searchWithTemporalDecay(
  query: string,
  halfLifeDays: number = 30
): Promise<SearchResult[]> {
  const results = await hybridFactSearch(query);
  
  // Apply temporal decay to each result
  return results.map(r => ({
    ...r,
    originalScore: r.score,
    score: applyTemporalDecay(r.score, r.timestamp, halfLifeDays)
  })).sort((a, b) => b.score - a.score);
}
```

3. **Visualize decay curve:**

```typescript
function calculateDecayCurve(halfLifeDays: number): number[] {
  const lambda = Math.log(2) / halfLifeDays;
  const days = Array.from({ length: 90 }, (_, i) => i); // 90 days
  
  return days.map(d => Math.exp(-lambda * d));
}

// Example: 30-day half-life
// Day 0: 1.0 (100%)
// Day 30: 0.5 (50%)
// Day 60: 0.25 (25%)
// Day 90: 0.125 (12.5%)
```

---

### Q25: File Indexing

**Answer:**

**Scope:**
- All Markdown (`.md`) files in workspace
- Subdirectories indexed recursively

**Extensions:**
- Plugins can add external directories via `agents.defaults.memorySearch.extraPaths`

### Implications for Cognitive Controller

✅ **Automatic Indexing:**
- Our roll-ups indexed automatically
- No manual indexing required

✅ **Extensible:**
- Can add custom paths if needed
- Can index external knowledge bases

⚠️ **Must Use .md Extension:**
- Only Markdown files indexed
- Must name roll-ups with `.md`

### Action Items

1. **Ensure roll-ups use .md extension:**

```typescript
function generateRollUpFilename(date: Date): string {
  const year = date.getFullYear();
  const month = String(date.getMonth() + 1).padStart(2, '0');
  const day = String(date.getDate()).padStart(2, '0');
  const hour = String(date.getHours()).padStart(2, '0');
  const minute = String(date.getMinutes()).padStart(2, '0');
  
  // MUST use .md extension for indexing
  return `${year}-${month}-${day}-${hour}${minute}.canonical.md`;
}
```

2. **Configure extra paths if needed:**

```json5
{
  "agents": {
    "defaults": {
      "memorySearch": {
        "extraPaths": [
          "/path/to/external/knowledge/base",
          "/path/to/project/docs"
        ]
      }
    }
  }
}
```

3. **Verify indexing scope:**

```typescript
async function listIndexedFiles(): Promise<string[]> {
  const workspace = getWorkspace();
  const memoryDir = path.join(workspace, 'memory');
  
  // Recursively find all .md files
  const files = await glob('**/*.md', {
    cwd: memoryDir,
    absolute: false
  });
  
  logger.info(`Found ${files.length} Markdown files in memory directory`);
  
  return files;
}
```

---

### Q26: Index Update Latency

**Answer:**

**Searchability:**
- Near-instant (following 1.5s debounce)
- Sync scheduled on session start or when search requested

**Type:**
- Asynchronous background sync
- Chat startup not blocked

### Implications for Cognitive Controller

✅ **Fast Indexing:**
- 1.5s latency acceptable
- Real-time for practical purposes

✅ **Non-Blocking:**
- Doesn't slow down agent
- Background process

⚠️ **Not Instant:**
- 1.5s delay before searchable
- Must account for in tests

### Action Items

1. **Wait for indexing in tests:**

```typescript
async function writeAndWaitForIndexing(
  filename: string,
  content: string
): Promise<void> {
  // Write file
  await fs.writeFile(filename, content, 'utf-8');
  
  // Wait for debounce + indexing
  await sleep(2000); // 1.5s debounce + 0.5s buffer
  
  // Verify indexed
  const indexed = await verifyFileIndexed(filename);
  
  if (!indexed) {
    throw new Error(`File not indexed after 2 seconds: ${filename}`);
  }
}
```

2. **Monitor indexing lag:**

```typescript
class IndexingMonitor {
  private writeTime: Map<string, Date> = new Map();
  
  onFileWritten(filename: string): void {
    this.writeTime.set(filename, new Date());
  }
  
  async onFileIndexed(filename: string): Promise<void> {
    const writeTime = this.writeTime.get(filename);
    
    if (!writeTime) {
      logger.warn(`File indexed but no write time recorded: ${filename}`);
      return;
    }
    
    const indexTime = new Date();
    const lag = indexTime.getTime() - writeTime.getTime();
    
    logger.info(`Indexing lag for ${filename}: ${lag}ms`);
    
    if (lag > 3000) {
      logger.warn(`High indexing lag: ${lag}ms > 3000ms`);
    }
    
    this.writeTime.delete(filename);
  }
}
```

---

### Q27: Performance Characteristics

**Answer:**

**Caching:**
- Embedding cache in SQLite (default: 50,000 entries)
- Avoids re-embedding unchanged text during updates

**Efficiency:**
- QMD sidecar (experimental) provides local-first search engine
- High-performance hybrid retrieval
- Doesn't tax main Gateway process

### Implications for Cognitive Controller

✅ **Efficient Caching:**
- 50,000 entry cache is generous
- Our roll-ups won't be re-embedded unnecessarily

✅ **QMD Sidecar:**
- Offloads search from Gateway
- Better performance
- Optional (falls back to built-in)

⚠️ **Cache Size:**
- 50,000 entries may fill over time
- Need to monitor cache usage

### Action Items

1. **Monitor embedding cache:**

```typescript
async function checkEmbeddingCacheSize(): Promise<number> {
  // Query OpenClaw's embedding cache
  const cacheSize = await gateway.query('SELECT COUNT(*) FROM embedding_cache');
  
  const maxSize = 50000;
  const usage = (cacheSize / maxSize) * 100;
  
  logger.info(`Embedding cache usage: ${cacheSize}/${maxSize} (${usage.toFixed(1)}%)`);
  
  if (usage > 90) {
    logger.warn('Embedding cache nearly full, may need cleanup');
  }
  
  return cacheSize;
}
```

2. **Optimize for cache hits:**

```typescript
// Write roll-ups in consistent format to maximize cache hits
function formatRollUpForCaching(rollUp: RollUpData): string {
  // Use consistent formatting
  // Avoid timestamps in content (use metadata instead)
  // Group similar content together
  
  return `# Roll-Up

## Facts
${rollUp.facts.map(f => f.content).join('\n')}

## Decisions
${rollUp.decisions.map(d => `${d.what}: ${d.why}`).join('\n')}

## Constraints
${rollUp.constraints.join('\n')}
`;
}
```

3. **Test QMD sidecar integration:**

```typescript
async function testQMDAvailability(): Promise<boolean> {
  try {
    // Check if QMD sidecar is running
    const response = await fetch('http://localhost:QMD_PORT/health');
    return response.ok;
  } catch (error) {
    logger.info('QMD sidecar not available, using built-in search');
    return false;
  }
}
```

---

## Summary: Context & Memory (Q15-Q27)

### Critical Findings

🔴 **Memory Flush = Opportunity** (Q16, Q18)
- Silent turn before compaction
- **This is when we create snapshot!**
- Must inject constraints to prevent paraphrasing

✅ **Clear Injection Mechanism** (Q15)
- Bootstrap files for first turn
- Transform modules for ongoing injection
- 150,000 char budget (generous)

✅ **Rich Search Capabilities** (Q20-Q27)
- Hybrid search (vector + BM25)
- Temporal decay (30-day half-life)
- MMR for diversity
- Fast indexing (1.5s latency)

✅ **Efficient Implementation** (Q27)
- 50,000 entry embedding cache
- QMD sidecar for performance
- Background indexing (non-blocking)

### Implementation Priorities

**High Priority:**
1. Hook into memory flush turn (Q16, Q18)
2. Write roll-ups to indexed directory (Q21, Q25)
3. Implement hybrid search (Q20, Q23)
4. Monitor compaction threshold (Q16, Q19)

**Medium Priority:**
5. Add temporal decay to search (Q24)
6. Implement BM25 search in SQLite (Q22)
7. Add MMR for diversity (Q23)
8. Monitor embedding cache (Q27)

**Low Priority:**
9. Parse session transcripts (Q17)
10. Test QMD sidecar (Q27)

### Integration Strategy

**Context Injection:**
- Use bootstrap files for initial snapshot
- Use transform modules for per-turn updates
- Hook into memory flush for pre-compaction snapshot

**Memory Search:**
- Write roll-ups to `memory/rollups/*.canonical.md`
- Use OpenClaw's memory_search for semantic queries
- Use our SQLite for structured queries
- Combine both for hybrid search

**Performance:**
- Leverage embedding cache (50,000 entries)
- Use QMD sidecar if available
- Monitor indexing lag (<2s target)

---

**Document Owner:** Project Team  
**Status:** Q15-Q27 Answered (27/100 total)  
**Next Batch:** Q28-Q40 (Session Management, Tool Execution, Agent Management)
