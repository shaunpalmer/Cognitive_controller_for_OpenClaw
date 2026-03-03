This is a sophisticated architectural challenge. You are essentially moving from "Reactive Automation" to "Cognitive Persistence."

To solve the "Gateway as a single point of failure for context" and the "Big MD" sprawl, we need to treat memory as a **tiered data pipeline**.

I have drafted a technical blueprint for your **Cognitive Controller**. It includes the SQLite event-sourcing schema to "tap" the gateway stream, the categorical routing logic (Personal vs. Professional), and the logic for the "Roll-up" script that prevents amnesia and sprawl.

### Why this fits your "Big Picture"

1. **Solves the Zombie Problem:** By using the `snapshots` table, the agent doesn't wake up with a blank slate; it wakes up with a "Resume Block" injected into its first thought.
2. **Category Isolation:** By splitting **Personal** and **Professional** at the database level, you ensure your house-monitoring sub-agent doesn't get confused by your marketing campaign data.
3. **Markdown Discipline:** Instead of the agent writing messily to a file, the **Consolidation Loop** acts as a professional editor, only writing high-quality, summarized "Truths" to your Obsidian vault.

Does this schema feel like it handles the "tapping the stream" part of the gateway properly for your use case?Cognitive-controller for OpenClaw
Cognitive Controller for OpenClaw
Tagline: An event-driven cognitive layer for OpenClaw: observe → rank → snapshot → escalate.
What this project is
Cognitive Controller adds a control tower to OpenClaw’s gateway. It watches what the agent is doing, keeps a bounded working set of “what matters right now,” and feeds the agent high-signal context so it can continue a conversation or workflow without drifting, stalling, or forgetting.
Think of it as an event graph + signal flow layer:
Observer: captures meaningful activity (messages, tool results, state transitions)
Reranker: selects the top active threads and builds a compact “resume block”
Snapshots (cache): keeps 1–10 recovery points per thread
Elastic Resolver: fails fast, changes tactics, and escalates to stronger models only when needed
Why it exists
OpenClaw is excellent at event-driven execution, but long-running work needs:
Continuity (carry forward what matters)
Bounded memory (avoid Markdown sprawl)
Recoverability (safe rollbacks, not “clear everything”)
Operator trust (heartbeat + small feedback, no silent failures)
Cognitive Controller aims to make OpenClaw feel less like a reactive zombie loop and more like a coordinated system.

Technical overview (how it works)
Cognitive Controller runs as a standard OpenClaw plugin (TypeScript) and does not modify core.
Data model
Append-only events store: inbound/outbound/tool/state events
Threads store: domain/topic routing + confidence/urgency + last progress
Snapshots store: compact “resume blocks” and recovery points (bounded)
Core loop
Capture event → write events
Route → domain: system|personal|professional + topic
Update thread state → confidence/urgency/last_progress
Rerank → pick Top-K threads (e.g., <= 8)
Snapshot → create a compact resume block (token budget ~500)
Inject → send envelope/metadata back into the next agent call
UI controls (minimal)
Clear Cache: dropdown 1,2,3,4,5,10 (bounded)
Toast: Cleared N snapshots · R remaining
Heartbeat LED: event-driven, broad progress detection (green/amber/red)

Roadmap (initial)
Phase 0 — Foundations
Plugin skeleton + deploy sync script (Windows-safe, real dir under ~/.openclaw/extensions/<id>)
SQLite schema + migrations
Observer capture hooks + heartbeat timestamps
Phase 1 — Working memory
Reranker + resume block builder
Snapshot store (cap 10 per thread)
Cache clear endpoint + toast payload
Phase 2 — Elastic resolver
Fail-fast policy: triage once → plan swap → resolve tier → block/cooldown
Confidence/urgency scoring + decay
Phase 3 — Obsidian integration (optional)
Index vault notes (tags/links/keywords)
Provide a “memory export” format for Obsidian
Optional: pull curated long-term memory from Obsidian back into OpenClaw

Decisions (locked)
Language: TypeScript (matches OpenClaw)
Storage: SQLite (append-only events + bounded snapshots)
Cache: bounded (1–10), no “Clear All”
Heartbeat: event-driven from real activity (no fake timers)


Contributing
This repo is early-stage and architecture-first. If you’re contributing, start with:
docs/architecture.md
docs/hook-contracts.md
docs/storage-schema.md
Here is a critical audit of the current architectural plan.

1. The "Fan-Out" Latency Trap
You’ve proposed a secondary stream to capture events in a non-blocking way.

The Risk: In Node.js, "non-blocking" doesn't mean "zero-cost." If the gateway is under high load, the overhead of JSON serialization and SQLite writes for every event can cause pressure on the event loop.

Brutal Fix: You need a Write-Ahead Buffer. Don't write every heartbeat to SQLite immediately. Collect them in memory and flush them in chunks every 5–10 seconds. If the system crashes, you lose 5 seconds of logs, but the gateway doesn't lag.

2. The "Resume Block" Hallucination Risk
The snapshots table relies on a resume_block (compressed summary).

The Risk: LLMs are notorious for missing "negative constraints" (e.g., "Don't touch the production DB") when summarizing. If the Consolidation Loop compresses a warning into a generic "Fixed database issues," the agent wakes up and repeats the original mistake because the specific "Do Not" was lost in the summary.

Brutal Fix: The snapshot must include a Hard Constraints field that is never summarized or compressed—it must be a verbatim carry-forward of critical boundaries.

3. Category Drift (Personal vs. Professional)
You’re splitting the world into two folders.

The Risk: Life is messy. If you're fixing a lightbulb (Personal) but using a tool you developed for work (Professional), the Reranker will likely pull from the wrong "Semantic" store or create duplicates in both. Over time, your Obsidian vault will end up with "Pro-fixing-lights" and "Personal-fixing-lights."

Brutal Fix: You need a Cross-Pollination Index. Instead of hard-segregated folders, use a Tag-based retrieval system where "Professional" is just a high-weight filter, not a wall.

4. The "Sleep" Script is the Single Point of Failure
Your Consolidation Loop is the most complex part of the system.

The Risk: If the "Roll-up" script fails (e.g., SQLite lock or LLM error), your "Hot" memory grows indefinitely. Retrieval slows down, costs skyrocket, and you end up with exactly the "Big MD" sprawl you were trying to avoid, just hidden in a database.

Brutal Fix: You need an Emergency TTL (Time To Live). If an event is older than 48 hours and hasn't been consolidated, it should be auto-archived to a "Raw Dump" file regardless of whether the "Smart Consolidation" finished. Stability over Intelligence.

5. Semantic Search vs. Filename Retrieval
You mentioned QMD and Obsidian.

The Risk: If you rely on QMD for keyword search, the agent will miss context that uses different synonyms (e.g., "Auth" vs "Permissions").

Brutal Fix: Since you are building this for yourself, don't trust keywords. Use sqlite-vec or a local embeddings model. It's 2026; running a small vector index locally is cheap and prevents the "I know I wrote this but the agent can't find it" frustration.

The Verdict
The architecture is theoretically sound but operationally heavy. You are essentially building a custom DBMS for AI state. It will work beautifully for the first 100 sessions, but its success depends entirely on the Consolidation Loop's ability to be a "good editor."

If your "Sleep Script" isn't perfect, the system will eventually suffer from "Synthetic Dementia"—where it remembers its summaries of summaries, but forgets the actual facts.

What part of that "brutal" assessment strikes you as the biggest worry for your specific workflow?

Cognitive Controller: Technical Blueprint

1. The Gateway Tap (Stream Interception)

To prevent OpenClaw from waking up as a "zombie," the Controller acts as a side-car to the Gateway. It performs a fan-out of every message:

Primary Loop: Standard OpenClaw execution.

Secondary Stream: Non-blocking capture into the Event Store (SQLite). This captures the "Raw Truth" before any Markdown sprawl occurs.

2. Memory Tiering & Compression Model

Instead of one big file, we use a Decay & Promote strategy:

Tier

Storage

Strategy

Retention

Instant (Hot)

SQLite (In-Mem)

High Fidelity: Every tool call and heartbeat.

1 Session

Episodic (Warm)

SQLite (Disk)

Summary-Only: Grouped events (e.g., "The 3pm Debugging Session").

30 Days

Semantic (Cold)

Obsidian (MD)

Canonical Knowledge: Only verified facts and patterns.

Forever

3. Categorization & Routing

Every event is tagged at the Gateway level:

Personal: (e.g., ~/obsidian/personal/)

Professional: (e.g., ~/obsidian/pro/)

Reranker Logic: When the agent wakes up for a 'Professional' task, the 'Personal' tier is physically excluded from the context window. This kills "Cognitive Drift" instantly.

4. SQLite Schema: The "Brain State"

-- Event Log: The "Black Box" recorder
CREATE TABLE event_log (
    id TEXT PRIMARY KEY,
    timestamp DATETIME DEFAULT CURRENT_TIMESTAMP,
    category TEXT, -- 'Personal' | 'Professional'
    thread_id TEXT,
    payload JSON,
    importance_score REAL -- Used to decide what survives the Roll-up
);

-- Snapshots: The "Hard Resume" (Prevents Hallucination)
CREATE TABLE snapshots (
    thread_id TEXT PRIMARY KEY,
    resume_block TEXT, -- Compressed summary
    hard_constraints TEXT, -- VERBATIM rules that must NEVER be summarized
    active_goal TEXT
);


5. The Consolidation Loop (The "Sleep" Cycle)

To solve the "Big MD" problem, the system must perform "Synaptic Pruning":

Extraction: Once a day, the LLM reads the event_log and extracts "Learnings."

Promotion: If a Learning is high-importance (e.g., "Always use port 8080 for this API"), it is written to Obsidian.

Pruning: The event_log entries are deleted.

The Result: Your Markdown files stay small because they only contain Knowledge, not Activity Logs.

6. Wake-up Protocol (Anti-Zombie)

Load Snapshot: Get the hard_constraints and active_goal.

Semantic Recall: Use QMD to pull the 3 most relevant "Canonical Truths" from the Obsidian Vault.

Inject: Prepend to prompt.

Result: The agent wakes up with the wisdom of 3 months of work, but only 1,000 tokens of context.
