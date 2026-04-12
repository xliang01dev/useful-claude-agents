# Event-Driven Architecture Design

**Reference document for:** [SKILL.md](SKILL.md) - event-driven-architecture skill

Use this skill when you're planning or designing an event-driven system before writing implementation code. This skill helps you make decisions about message brokers, consumer patterns, identity resolution, worker architecture, and scaling—preventing common pitfalls discovered through production experience.

## When to Use

- Starting a new event-driven feature or system
- Redesigning existing queue/pub-sub architecture
- Planning worker/consumer topology
- Deciding between event streams and queues
- Designing API for event-driven workflows

## Design Decision Tree

### 1. Broker & Delivery Semantics

**Question: What are your compliance/auditability requirements?**

- **High (regulatory, audit trails, replay needed):** Use event streams (JetStream, Kafka)
  - Immutable log of all events
  - Multiple subscribers possible
  - Temporal queries and replay capability
  - Aligns with compliance standards
  
- **Low (ephemeral work queue):** Traditional queue possible (RabbitMQ, SQS)
  - Work consumed once and discarded
  - Simpler infrastructure
  - Note: Still consider streams for observability

**Deliverable:** Document your choice and why (compliance requirements, replay capability, audit needs)

---

### 2. Message Distribution: Topic Routing vs Consumer Groups

**Decision: How do you want work distributed among workers?**

**Topic-Based Routing (DON'T DO THIS for stateless scaling):**
```
❌ Don't: pub to work.{entity_id}, expect per-entity ordering via topic sharding
  Problem: Requires stateful routing; doesn't scale horizontally
  Workers can't "own" specific topics
```

**Consumer Groups (DO THIS):**
```
✅ Do: Publish to generic topic; use queue groups for round-robin distribution
  Pattern:
  - Publisher: send to "work" topic (stateless)
  - Consumer: subscribe with deliver_group="batch-workers" (queue group)
  - Broker: automatically round-robins among subscribers
  
  Why: Stateless publisher and consumer; scales by adding instances
```

**Deliverable:** Diagram showing:
- Topic names (generic, not entity-sharded)
- Publisher behavior (stateless publish)
- Consumer group setup (deliver_group name)
- How new instances scale

---

### 3. Worker Architecture: Instances vs Threads

**Decision: How do you scale your workers?**

**Anti-pattern (Threads in single instance):**
```
❌ Max throughput = max threads per instance
  Doesn't scale across machines
  Defeats microservices architecture
```

**Correct (Stateless service instances):**
```
✅ Each worker is a stateless container/pod
  - Subscribe to queue group with deliver_group
  - Process work independently
  - Scale by adding/removing instances (not threads)
  - Auto-scale based on queue depth
```

**Requirements:**
- No in-memory state across messages (use external store)
- Each instance can start/stop without coordination
- Idempotent processing (same message processed twice = same outcome)

**Deliverable:** Document:
- How each instance is stateless
- What external store holds state (if needed)
- Auto-scaling trigger (queue depth, CPU, etc.)
- Idempotency strategy (deduplication, request IDs, etc.)

---

### 4. Identity Resolution: Single Source of Truth

**Decision: How do you map external identifiers to canonical IDs?**

**Anti-pattern (Sharded by source):**
```
❌ Source A → shard 0 (MED-123 → uuid-x)
   Source B → shard 1 (MED-123 → uuid-y)  ← Same ID, different canonical
   Source C → shard 2 (MED-123 → uuid-z)
   
   Problem: Multiple canonical IDs for one entity breaks identity resolution
```

**Correct (Single primary + read replicas):**
```
✅ Primary identity table (source of truth): all inserts/updates go here
   Read replicas: for scaling SELECT queries
   
   Invariant: One external ID → one canonical ID (always)
   
   Schema:
   - CREATE UNIQUE INDEX idx_external_id (source_id, external_identifier)
   - Atomic upsert: INSERT ... ON CONFLICT DO UPDATE
```

**Challenges & Trade-offs:**
- **Atomic upsert contention:** Postgres row locks on (source, identifier) tuple
  - Acceptable for MVP: lock time is milliseconds, different sources don't contend
  - Bottleneck is per-source (not global)
  - For production: measure actual traffic (new registration rarely >1% of traffic)
  - Mitigation: Read replicas for queries; primary for writes only

**Deliverable:** Document:
- Schema for identity table (unique constraints)
- Why no sharding on identity (invariant preservation)
- Upsert logic (atomic, or with explicit locking)
- Contention analysis (expected per-source throughput)

---

### 5. Consumer Group Identity & Observability

**Decision: How do you track which instance processed which message?**

**Anti-pattern (Same name for both):**
```
❌ durable_name: "batch-workers"
   deliver_group: "batch-workers"
   
   Problem: Can't distinguish which instance processed what
   Debugging is harder; no per-instance resume tracking
```

**Correct (Separate identity and group):**
```
✅ durable_name: "worker-1", "worker-2", "worker-3"  (per instance, unique)
   deliver_group: "batch-workers"  (shared, for group membership)
   
   Benefits:
   - Each instance tracked independently
   - Resume from per-instance position on restart
   - Better dead-letter handling
   - Observability: see which instance processed message N
```

**Deliverable:** Document:
- How durable_name is generated per instance (hostname, UUID, pod name)
- deliver_group name (shared for all instances)
- Why this matters for observability

---

### 6. Durability & Side Effects: Ordering Matters

**Decision: How do you safely trigger side effects from persisted state?**

**Anti-pattern (Write-behind cache):**
```
❌ Update Redis cache → queue job → async flush to Postgres
   
   Problem: Side effect (job enqueued) happens before durability
   If cache crashes: state lost, but job already triggered
   Result: Duplicate jobs or missing work
```

**Correct patterns:**

**Pattern A (Write-through, immediate side effect):**
```
✅ 1. Write state to Postgres (durable)
   2. Trigger side effect (within same transaction or immediately after)
   3. Update cache (optional, for performance)
   
   Invariant: Durability before effects
```

**Pattern B (Write-behind for cache only, no side effects):**
```
✅ Update cache → async flush to Postgres
   (Only if data is non-critical and doesn't trigger effects)
```

**Choose based on:**
- State triggers side effects? → Use write-through
- Pure caching (no effects)? → Write-behind OK
- Debounce table (triggers reconciliation)? → Write-through required

**Deliverable:** Document:
- Which state triggers which side effects
- Durability ordering (Postgres first, then effects)
- If using Redis/cache, whether it's safe to flush async

---

### 7. API Design: Data Model Drives Endpoint Strategy

**Decision: What endpoints do you expose for event-driven data?**

**Anti-pattern (Assume pagination always works):**
```
❌ GET /resource/{id}/timeline?page=1&limit=10
   
   Problem: Materialized view stores only latest per resource
   Pagination on "latest 1 row" is pointless
   Wastes complexity for O(1) query
```

**Correct (Separate endpoints for access patterns):**
```
✅ GET /resource/{id}/timeline/latest
     → Returns single row (O(1) lookup from materialized view)
     
   GET /resource/{id}/timeline/history?page=1
     → Returns paginated full history (if needed)
     → Optional; may require different backing store
```

**Guidelines:**
- Understand your data model first (what's stored, what's computed)
- Don't add pagination for queries that return 1 row
- Separate endpoints for different access patterns
- Return only what clients need (no internal deduplication hashes, no internal IDs)

**Deliverable:** Document:
- Data model for each endpoint (where does data come from?)
- Access pattern (latest? paginated history? aggregated?)
- Response fields (client-relevant only; no internal details)

---

## Pre-Implementation Checklist

Before you write code, you should have clear answers to:

- [ ] **Broker choice & reason** (event stream vs queue, JetStream vs alternatives)
- [ ] **Compliance requirements** (audit trails? replay? temporal queries?)
- [ ] **Topic & consumer group names** (documented, no per-entity sharding)
- [ ] **Worker architecture** (stateless instances, auto-scaling strategy)
- [ ] **Identity resolution** (primary table, upsert logic, contention analysis)
- [ ] **Consumer group identity** (durable_name per instance, deliver_group shared)
- [ ] **Durability ordering** (which state triggers effects, write-through required?)
- [ ] **API endpoints** (latest vs history, what fields returned)
- [ ] **Error handling & retries** (dead-letter queues, retry logic, idempotency)
- [ ] **Scaling assumptions challenged** (measure traffic distribution before declaring bottlenecks)
- [ ] **Bottleneck documented** (per-entity contention? global throughput limits?)

---

## Common Pitfalls & How to Avoid Them

### Pitfall 1: Assuming Topic Sharding Scales
**Problem:** Design per-entity topics expecting ordering and work distribution
**Why it fails:** Stateless publisher can't know which topics to route to; breaks horizontal scaling
**Solution:** Generic topics + consumer groups (queue group semantics)

### Pitfall 2: Sharding Identity Tables
**Problem:** Per-source shards to reduce upsert contention
**Why it fails:** Same external ID gets multiple canonical IDs; breaks identity resolution
**Solution:** Single primary + read replicas; accept per-source lock contention (it's not global)

### Pitfall 3: Threading Instead of Instances
**Problem:** Design workers as threads in one container
**Why it fails:** Max throughput = max threads; doesn't scale across machines
**Solution:** Stateless instances; scale by adding/removing containers

### Pitfall 4: Writing State Before Durability
**Problem:** Cache-first, async flush, trigger side effect
**Why it fails:** Effect happens before durability; if cache crashes, state lost but effect already done
**Solution:** Postgres first (durable), then trigger effect, then update cache (optional)

### Pitfall 5: Declaring Bottlenecks Before Measuring
**Problem:** Assume atomic upsert is a bottleneck; propose sharding
**Why it fails:** 99% of traffic is updates (existing records); new record creation <1%
**Solution:** Measure actual traffic distribution before declaring architectural constraints

### Pitfall 6: API Design from Database Schema
**Problem:** Include internal IDs and deduplication hashes in responses
**Why it fails:** Clients don't need internal details; couples API to implementation
**Solution:** Response models represent client view only (public ID, summary, timestamp)

---

## Output: Architecture Decision Document

After working through this skill, create a **one-page Architecture Decision Document** covering:

1. **Broker & Delivery** — Which broker, why (compliance needs), why not alternatives
2. **Scaling Model** — Stateless instances, consumer groups, auto-scaling trigger
3. **Identity Resolution** — Single primary table, upsert logic, contention ceiling
4. **Durability Ordering** — What triggers side effects, write-through required
5. **API Strategy** — Endpoint list with data models, response fields
6. **Bottleneck Analysis** — Known scaling ceilings, per-entity vs global
7. **Key Invariants** — Single source of truth for identity, durability before effects, stateless workers

This document prevents re-design mid-implementation when you discover the assumptions were wrong.

