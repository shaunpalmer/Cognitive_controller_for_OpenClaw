# Snapshot Schema Analysis: Research Impact

**Date:** 2026-03-03  
**Source:** Snapshot.json  
**Status:** Analysis Complete

---

## Current Schema Review

Your Snapshot.json is well-structured and aligns with most of our research findings. However, based on the 78 questions we researched, here are the key updates needed:

---

## Required Changes

### 1. Handshake Structure (CRITICAL)

**Current:**
```json
{
  "schema_version": "1.0",
  "snapshot_id": "uuid",
  "created_at": "2026-02-28T00:00:00Z",
  "expires_at": "2026-02-28T02:00:00Z"
}
```

**Research Finding (Q5, Q47):**
Requirements specify handshake metadata should include `source_event_ids` for traceability.

**Recommended:**
```json
{
  "handshake": {
    "snapshot_id": "uuid",
    "schema_version": "1.0",
    "created_at": "2026-02-28T00:00:00Z",
    "expires_at": "2026-02-28T02:00:00Z",
    "source_event_ids": ["evt_100", "evt_150", "evt_200"],
    "trigger": "memory_flush|scheduled|manual|file_change"
  }
}
```

**Why:** 
- Groups metadata logically
- Adds source event traceability
- Adds trigger type for debugging
- Matches requirements R5 exactly

---

### 2. Field Naming: "facts" → "recent_facts"

**Current:**
```json
"facts": [...]
```

**Research Finding (Q5):**
Requirements R5 specifies field name as `recent_facts` to emphasize bounded, recent nature.

**Recommended:**
```json
"recent_facts": [...]
```

**Why:**
- Matches requirements exactly
- Emphasizes bounded nature (max 20 entries)
- Clarifies these are recent, not all facts

---

### 3. Tool State Simplification

**Current:**
```json
"tool_state": {
  "gateway_healthy": true,
  "last_heartbeat_at": "2026-02-28T00:00:00Z",
  "known_tools": ["tool_a", "tool_b"],
  "notes": "short optional"
}
```

**Research Finding (Q5, Q32-Q37):**
Requirements R5 specifies simpler structure: available, unavailable, health.

**Recommended:**
```json
"tool_state": {
  "available": ["tool_a", "tool_b", "tool_c"],
  "unavailable": ["tool_d"],
  "health": "healthy|degraded|unknown",
  "last_check_at": "2026-02-28T00:00:00Z"
}
```

**Why:**
- Matches requirements R5
- Simpler structure
- Clear availability status
- Aligns with OpenClaw's tool execution model (Q32-Q37)

---

### 4. Remove "routing" Section (Out of Scope)

**Current:**
```json
"routing": {
  "policy": "safe",
  "suggested_model": "small|medium|large",
  "escalation_reason": "string"
}
```

**Research Finding:**
Elastic resolver (routing, escalation) is explicitly out of scope for Phase 0 & 1.

**Recommended:**
Remove entirely or move to `extensions` for future use.

**Why:**
- Out of scope for Phase 0/1
- Elastic resolver is Phase 2+
- Keep schema focused on current phase

---

### 5. Token Budget in Meta

**Current:**
```json
"meta": {
  "domain_hint": "personal|professional|system",
  "token_budget": 500,
  "event_range": { "from": "evt_100", "to": "evt_200" }
}
```

**Research Finding (Q45, Q18):**
Token budget should be validated using tiktoken, and actual token count should be tracked.

**Recommended:**
```json
"meta": {
  "domain_hint": "personal|professional|system",
  "token_budget": 500,
  "token_count_actual": 487,
  "token_estimation_model": "cl100k_base",
  "event_range": { "from": "evt_100", "to": "evt_200" },
  "agent_id": "agent-123",
  "model": "gpt-4"
}
```

**Why:**
- Track actual vs budget for monitoring
- Record which tokenizer was used
- Add agent_id for per-agent isolation (Q40)
- Add model for token estimation accuracy (Q45)

---

### 6. Hard Constraints Enhancement

**Current:**
```json
"hard_constraints": [
  {
    "text": "VERBATIM STRING - never summarize",
    "source": { "type": "user|system|tool", "id": "evt_001" }
  }
]
```

**Research Finding (Q8, Q19):**
Good structure, but add constraint_id for tracking and updates.

**Recommended:**
```json
"hard_constraints": [
  {
    "constraint_id": "const_001",
    "text": "VERBATIM STRING - never summarize",
    "source": { "type": "user|system|tool", "id": "evt_001" },
    "added_at": "2026-02-28T00:00:00Z",
    "immutable": true
  }
]
```

**Why:**
- Unique ID for tracking
- Timestamp for audit trail
- Immutable flag emphasizes verbatim requirement

---

### 7. Add Confidence Decay Tracking

**Current:**
Threads have confidence but no decay tracking.

**Research Finding (Q10):**
Requirements R10 specifies confidence decay: 0.05 per hour of inactivity.

**Recommended:**
```json
"active_threads": [
  {
    "thread_id": "string",
    "title": "string",
    "next_step": "string",
    "status": "active|stalled|done",
    "confidence": 0.75,
    "confidence_last_updated": "2026-02-28T00:00:00Z",
    "last_event_at": "2026-02-28T00:00:00Z",
    "evidence_ids": ["evt_123", "evt_456"],
    "blocked_by": []
  }
]
```

**Why:**
- Track when confidence was last updated
- Enable decay calculation
- Separate from last_event_at (events don't always update confidence)

---

## Updated Schema (Recommended)

Here's the complete updated schema incorporating all research findings:

```json
{
  "handshake": {
    "snapshot_id": "snap_20260228_000000_abc123",
    "schema_version": "1.0",
    "created_at": "2026-02-28T00:00:00Z",
    "expires_at": "2026-02-28T02:00:00Z",
    "source_event_ids": ["evt_100", "evt_150", "evt_200"],
    "trigger": "memory_flush"
  },

  "objective_now": "Complete user authentication implementation",

  "active_threads": [
    {
      "thread_id": "thread_001",
      "title": "Implement OAuth2 flow",
      "next_step": "Test token refresh logic",
      "status": "active",
      "confidence": 0.75,
      "confidence_last_updated": "2026-02-28T00:00:00Z",
      "last_event_at": "2026-02-28T00:00:00Z",
      "evidence_ids": ["evt_123", "evt_456"],
      "blocked_by": []
    }
  ],

  "recent_facts": [
    {
      "fact_id": "fct_001",
      "text": "User prefers PostgreSQL over MySQL for production database.",
      "type": "decision",
      "confidence": 1.0,
      "source": { "type": "user", "id": "evt_123" },
      "ts": "2026-02-28T00:00:00Z",
      "tags": ["professional", "database"]
    }
  ],

  "hard_constraints": [
    {
      "constraint_id": "const_001",
      "text": "All API endpoints must use HTTPS in production",
      "source": { "type": "user", "id": "evt_001" },
      "added_at": "2026-02-28T00:00:00Z",
      "immutable": true
    }
  ],

  "decisions": [
    {
      "decision_id": "dec_001",
      "what": "Use JWT for authentication tokens",
      "why": "Industry standard, stateless, scalable",
      "how": "Implement using jsonwebtoken library with RS256 signing",
      "when": "v1.0 release",
      "who": "system",
      "inactive": false,
      "revisit_if": [
        "Security audit recommends alternative",
        "Performance issues with token validation"
      ],
      "source": { "type": "system", "id": "evt_222" },
      "ts": "2026-02-28T00:00:00Z"
    }
  ],

  "open_questions": [
    {
      "question_id": "q_001",
      "text": "Should we support OAuth providers beyond Google and GitHub?",
      "priority": "medium",
      "critical": false,
      "blocker": false,
      "source": { "type": "system", "id": "evt_123" },
      "ts": "2026-02-28T00:00:00Z"
    }
  ],

  "tool_state": {
    "available": ["file_read", "file_write", "web_search"],
    "unavailable": ["database_query"],
    "health": "healthy",
    "last_check_at": "2026-02-28T00:00:00Z"
  },

  "last_actions": [
    {
      "action_id": "act_001",
      "text": "Created user authentication endpoint",
      "outcome": "success",
      "source": { "type": "tool", "id": "evt_999" },
      "ts": "2026-02-28T00:00:00Z"
    }
  ],

  "meta": {
    "agent_id": "agent-123",
    "model": "gpt-4",
    "domain_hint": "professional",
    "token_budget": 500,
    "token_count_actual": 487,
    "token_estimation_model": "cl100k_base",
    "event_range": { "from": "evt_100", "to": "evt_200" }
  },

  "extensions": {
    "routing": {
      "policy": "safe",
      "suggested_model": "medium",
      "escalation_reason": null
    }
  }
}
```

---

## Key Changes Summary

1. ✅ **Grouped metadata into `handshake`** - Better organization, adds traceability
2. ✅ **Renamed `facts` → `recent_facts`** - Matches requirements exactly
3. ✅ **Simplified `tool_state`** - Clearer structure, matches requirements
4. ✅ **Moved `routing` to `extensions`** - Out of scope for Phase 0/1
5. ✅ **Enhanced `meta`** - Added agent_id, model, actual token count
6. ✅ **Added `constraint_id` and `immutable`** - Better tracking
7. ✅ **Added `confidence_last_updated`** - Enable decay calculation
8. ✅ **Added `trigger` to handshake** - Debug which event triggered snapshot
9. ✅ **Added `action_id` and `outcome`** - Better action tracking

---

## Validation Against Requirements

### R5: Snapshot Structure and Schema ✅
- 9 fields: objective_now, active_threads, recent_facts, hard_constraints, decisions, open_questions, tool_state, last_actions, handshake
- Handshake includes created_at, expires_at, source_event_ids
- All fields present and correctly structured

### R8: Atomic Facts and Verbatim Constraints ✅
- Facts have fact_id, timestamp, source_type, source_id, content (text), tags
- Constraints marked as immutable
- Source traceability maintained

### R6: Thread Management ✅
- Thread fields: thread_id, title, next_step, confidence, status, blocked_by
- Status: active, stalled, done
- Confidence: 0.0-1.0
- Blocked_by array with valid values

### R10: Confidence Gradient Scoring ✅
- Confidence field present
- confidence_last_updated enables decay calculation
- Ready for 0.05/hour decay implementation

---

## Token Budget Considerations

**Research Finding (Q45, Q18):**
- Use tiktoken for estimation
- Target: 300-500 tokens
- Must validate before storage

**Current Schema Size:**
With example data, this schema is approximately 450 tokens (estimated).

**Recommendations:**
1. Monitor actual token counts in production
2. Trim lowest-priority items if budget exceeded
3. Prioritize: hard_constraints > active_threads > recent_facts > decisions > open_questions > last_actions

---

## Implementation Notes

### 1. Schema Validation
```typescript
import Ajv from 'ajv';

const ajv = new Ajv();
const snapshotSchema = require('./Snapshot.json');
const validate = ajv.compile(snapshotSchema);

function validateSnapshot(snapshot: Snapshot): boolean {
  const valid = validate(snapshot);
  if (!valid) {
    logger.error('Snapshot validation failed:', validate.errors);
  }
  return valid;
}
```

### 2. Token Budget Enforcement
```typescript
import { encoding_for_model } from 'tiktoken';

function enforceTokenBudget(snapshot: Snapshot, budget: number): Snapshot {
  const encoder = encoding_for_model('gpt-4');
  let tokens = encoder.encode(JSON.stringify(snapshot)).length;
  
  while (tokens > budget) {
    // Trim lowest priority items
    if (snapshot.last_actions.length > 2) {
      snapshot.last_actions.pop();
    } else if (snapshot.open_questions.length > 1) {
      snapshot.open_questions.pop();
    } else if (snapshot.recent_facts.length > 5) {
      snapshot.recent_facts.pop();
    } else {
      break; // Can't trim further
    }
    
    tokens = encoder.encode(JSON.stringify(snapshot)).length;
  }
  
  snapshot.meta.token_count_actual = tokens;
  return snapshot;
}
```

### 3. Per-Agent Isolation
```typescript
function getSnapshotPath(agentId: string, snapshotId: string): string {
  return path.join(
    os.homedir(),
    '.openclaw',
    'agents',
    agentId,
    '.cognitive-controller',
    'snapshots',
    `${snapshotId}.json`
  );
}
```

---

## Next Steps

1. **Update Snapshot.json** with recommended changes
2. **Create JSON Schema** for validation (snapshot-schema.json)
3. **Update TypeScript types** to match new schema
4. **Implement validation** in snapshot assembly
5. **Test token budgeting** with real data
6. **Update design.md** to reference new schema

---

## Backward Compatibility

If you have existing snapshots, implement migration:

```typescript
function migrateSnapshotV1toV1_1(oldSnapshot: any): Snapshot {
  return {
    handshake: {
      snapshot_id: oldSnapshot.snapshot_id,
      schema_version: '1.1',
      created_at: oldSnapshot.created_at,
      expires_at: oldSnapshot.expires_at,
      source_event_ids: [],
      trigger: 'migration'
    },
    objective_now: oldSnapshot.objective_now,
    active_threads: oldSnapshot.active_threads.map(t => ({
      ...t,
      confidence_last_updated: t.last_event_at
    })),
    recent_facts: oldSnapshot.facts || oldSnapshot.recent_facts,
    hard_constraints: oldSnapshot.hard_constraints.map((c, i) => ({
      constraint_id: `const_${i}`,
      text: c.text,
      source: c.source,
      added_at: oldSnapshot.created_at,
      immutable: true
    })),
    decisions: oldSnapshot.decisions,
    open_questions: oldSnapshot.open_questions,
    tool_state: {
      available: oldSnapshot.tool_state.known_tools || [],
      unavailable: [],
      health: oldSnapshot.tool_state.gateway_healthy ? 'healthy' : 'degraded',
      last_check_at: oldSnapshot.tool_state.last_heartbeat_at
    },
    last_actions: oldSnapshot.last_actions.map((a, i) => ({
      action_id: `act_${i}`,
      text: a.text,
      outcome: 'unknown',
      source: a.source,
      ts: a.ts
    })),
    meta: {
      ...oldSnapshot.meta,
      agent_id: 'unknown',
      model: 'unknown',
      token_count_actual: 0,
      token_estimation_model: 'cl100k_base'
    },
    extensions: {
      routing: oldSnapshot.routing
    }
  };
}
```

---

**Conclusion:**

Your Snapshot.json is solid! The recommended changes are mostly organizational (grouping handshake, renaming facts) and additive (tracking fields). The core structure aligns well with our research findings. The main updates ensure compliance with requirements R5, R8, R6, R10 and integrate learnings from Q40 (per-agent isolation), Q45 (token estimation), and Q47 (memory flush triggers).

