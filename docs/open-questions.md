# Cognitive Controller — Open Questions (Parking Lot)

> These are intentionally unanswered. They exist so we can make decisions later without losing the thread.

## Snapshot Contract & Validation
- Should the snapshot contract be **strictly JSON Schema validated** at runtime (reject invalid snapshots), or **warn + continue**?
- Should `facts[]` allow **inferences** at all, or be **100% source-verifiable** in v1?
- Do we want additional domain hints beyond `personal|professional|system` (e.g., `project:<name>`), or keep that purely as `tags[]`?

## Threads & Continuity
- Should `active_threads[]` be capped at **5** or **7** threads in v1?
- Should we add `blocked_by` enums to threads (e.g., `user_input`, `missing_credential`, `rate_limit`, `external_dependency`)?
- What is the default snapshot expiry window: **48 hours**, **7 days**, or **14 days**?

## Facts: What Counts as “Truth”
- Should `recent_facts[]` be strictly **verifiable events** (tool outputs, user statements), or can it include **inferred context**?
- What are the accepted `source.type` values in v1 (`tool|user|system|db|inference`)—any others needed?

## Decisions: Structure & Retention
- For `decisions.when`, do we prefer **absolute timestamps** (strict) or **phase-based timing** (e.g., “after 1 week of telemetry”)?
- For `decisions.who`, should the allowed values be `user|system|operator`, or also support `agent:<name>` for multi-agent setups?
- Do we want a dedicated `principles[]` section (e.g., “reliability > surprises”) so the model doesn’t have to infer guiding rules from decisions?

## Consolidation & Roll-up Policy
- Should consolidation run on a fixed schedule (**cron**, e.g., every 30 min) or via **idle detection** (no events for X minutes)?
- What’s the emergency fallback if consolidation fails: auto-archive to a **raw dump** after 48 hours, or something else?
- What is the long-term boundary between what lives in **SQLite** vs what gets written to **Markdown roll-ups**?

## Routing & Escalation
- What is the default routing policy: always start small and escalate, or start medium for better continuity?
- What are the “revisit” triggers for escalating model size (e.g., low confidence, repeated failures, stalled threads)?
