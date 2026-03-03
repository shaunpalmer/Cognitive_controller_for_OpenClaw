# 🧠 Cognitive-Controller for OpenClaw  
### *The "Control Tower" for OpenClaw: Observe → Rank → Snapshot → Escalate*

---

## 🧠 What Is This?

The **Cognitive-Controller** is an event-driven cognitive layer designed to sit on top of the OpenClaw gateway.

Standard agents often suffer from:

- **Context drift** in long-running sessions  
- Becoming **reactive zombies** that respond but don’t reason  
- Losing high-signal information in long chat histories  

This controller solves that by maintaining a **bounded, high-signal working set** of *what matters right now*.

It transforms a raw stream of events into a:

- Coherent  
- Prioritized  
- Recoverable  
- Operator-visible workflow  

---

## 🛠 Why It Exists

OpenClaw is excellent at execution — but complex workflows require more than a rolling chat history.

The Cognitive-Controller provides:

### 🔄 Continuity  
Carry forward critical context without hitting token limits.

### 📦 Bounded Memory  
Prevent “Markdown sprawl” by prioritizing signal over noise.

### ♻ Recoverability  
Maintain **1–10 safe rollback points per thread**.

### 🔍 Operator Trust  
Expose system health via:
- Heartbeats  
- Explicit feedback loops  
- Progress LEDs  

---

## 🏗 Core Architecture

The Cognitive-Controller runs as a **standard TypeScript plugin** and requires **no modifications to the OpenClaw core**.

### Components

| Component | Function |
|------------|----------|
| **Observer** | Captures messages, tool results, and state transitions |
| **Reranker** | Selects Top-K active threads and builds a ~500 token resume block |
| **Snapshots** | Maintains a bounded cache (1–10 recovery points per thread) |
| **Elastic Resolver** | Handles fail-fast policies, tactic swaps, and model escalation |

---

## 🔁 The Core Loop

1. **Capture**  
   Write inbound/outbound events to an append-only SQLite store.

2. **Route**  
   Categorize by domain (system, personal, professional) and topic.

3. **Score**  
   Update thread state based on:
   - Confidence  
   - Urgency  
   - Progress  

4. **Rerank**  
   Filter for the Top-K threads (default ≤ 8).

5. **Snapshot**  
   Generate a compact resume block for the LLM.

6. **Inject**  
   Feed the context envelope into the next agent call.

---

## 💾 Technical Specifications

**Language:** TypeScript (Native OpenClaw compatibility)  
**Storage:** SQLite  
- Append-only event logs  
- Bounded snapshot cache  

**Heartbeat:**  
- Event-driven  
- Triggered by real activity  
- No synthetic timers  

**UI Elements:**  
- Minimal toast notifications  
- 3-state LED indicator:
  - 🟢 Green — Healthy / Progressing  
  - 🟠 Amber — Stalled / Needs Attention  
  - 🔴 Red — Failure / Escalation  

---

## 🗺 Roadmap

### Phase 0 — Foundations 🏗️
- [ ] Plugin skeleton & Windows-safe deployment sync  
- [ ] SQLite schema migrations  
- [ ] Observer capture hooks & heartbeat implementation  

---

### Phase 1 — Working Memory 🧠
- [ ] Reranker & Resume Block builder  
- [ ] Bounded Snapshot store (max 10/thread)  
- [ ] Cache management endpoints & UI toasts  

---

### Phase 2 — Elastic Resolver ⚡
- [ ] Fail-fast triage & tactic swap policies  
- [ ] Confidence/Urgency scoring with decay algorithms  
- [ ] Tiered model escalation (stronger model on stall)  

---

### Phase 3 — External Integration (Optional) 📝
- [ ] Obsidian vault indexing (tags + links)  
- [ ] Long-term memory export/import formats  

---

## 🤝 Contributing

This project is **architecture-first**.

We prioritize:

- Structural integrity  
- Clear contracts  
- Predictable behavior  
- Bounded complexity  

Before contributing, review the core contracts:

- `docs/architecture.md`  
- `docs/hook-contracts.md`  
- `docs/storage-schema.md`  

---

## 🎯 Design Philosophy

> Signal over noise.  
> Memory with boundaries.  
> Escalation with intent.  
> Recovery by design.

The Cognitive-Controller is not just a plugin —  
it is the stability layer that keeps autonomous systems coherent over time.
