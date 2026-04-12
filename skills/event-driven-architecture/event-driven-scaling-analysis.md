# Event-Driven Scaling Analysis

**Reference document for:** [SKILL.md](SKILL.md) - event-driven-architecture skill

Use this skill when you suspect a bottleneck or are considering major architectural changes (sharding, replication) to solve performance issues. This skill prevents premature optimization by measuring actual traffic patterns and identifying real vs imagined bottlenecks.

## When to Use

- Someone proposes sharding to "solve" a bottleneck
- You're planning to refactor identity resolution or worker architecture
- Considering Redis, caching, or other optimization strategies
- Claims of "medium production bottleneck" lack measurement data
- Need to justify scaling ceiling or architectural constraint

## Process: Measure Before Deciding

### Step 1: Challenge the Assumption

**Question to ask:** "What's the evidence this is a bottleneck?"

Common assumptions that are *wrong without measurement*:
- "Atomic upsert on identity table will serialize at scale"
- "Row locks on debounce table will limit throughput"
- "Per-entity sharding is necessary"
- "This needs Redis optimization"

**Action:** Ask for metrics or estimates before accepting the problem statement.

---

### Step 2: Understand Traffic Distribution

**Collect (or estimate):**

1. **Event types & ratios**
   ```
   Total events per day: ___
   - Updates to existing records: ___ (%)
   - New record creation: ___ (%)
   - Other (deletes, corrections): ___ (%)
   ```

   **Key insight:** In most systems, >95% of events are updates.
   New record creation is often <1% of traffic.

2. **Hotspots & skew**
   ```
   - Do certain entities get disproportionate traffic? (Pareto distribution)
   - Is lock contention per-entity or global?
   - Example: Patient 123 gets 1000 events/hour; typical patient gets 10 events/hour
   ```

3. **Peak vs average**
   ```
   Average events/sec: ___
   Peak events/sec: ___
   Peak duration: ___ minutes
   ```

4. **Database capabilities (baseline)**
   ```
   Postgres max writes/sec: ~1000 inserts/sec (typical single instance)
   Row lock duration: <1ms (typical atomic upsert)
   Concurrent locks held: number before contention
   ```

**Deliverable:** A simple table or chart showing traffic distribution.

---

### Step 3: Identify the Real Bottleneck

**Test hypotheses in order:**

#### Hypothesis 1: Atomic Upsert Lock Contention (Identity Table)

**Testing approach (database-agnostic):**

1. **Measure lock hold time** on atomic upsert operation
   - Benchmark: INSERT-or-UPDATE on indexed columns (source_id, external_id)
   - Expected: <1ms per operation (sub-millisecond)
   - Method: Time 5,000 sequential operations; calculate P50, P99 latency

2. **Measure concurrent throughput**
   - Benchmark: 5,000 concurrent upserts on same table
   - Expected: Complete in ~100ms (i.e., throughput ~50,000 ops/sec)
   - If takes >1 second: you have contention

3. **Identify contention scope**
   - Test 1: All 5,000 inserts on same (source_id, external_id) → measures per-row lock
   - Test 2: 5,000 inserts on different (source_id, external_id) → measures table-level contention
   - Per-row lock much faster than global? → It's per-entity, acceptable
   - Both slow? → It's table-level, bigger problem

**Expected results:**
- Per-row lock hold time: <1ms
- 5,000 inserts same row: ~5-10ms total (mostly lock queue)
- 5,000 inserts different rows: <1ms total (parallel)
- Not a global bottleneck if per-row only

**Database-specific syntax (examples):**

PostgreSQL:
```sql
-- Atomic upsert with lock measurement
EXPLAIN ANALYZE
INSERT INTO records (source_id, external_id, canonical_id) 
VALUES ($1, $2, gen_random_uuid())
ON CONFLICT (source_id, external_id) 
DO UPDATE SET updated_at = NOW();

-- Measure contention with concurrent connections
-- See: pg_locks, lock_timeout
```

MySQL:
```sql
-- Atomic upsert
INSERT INTO records (source_id, external_id, canonical_id)
VALUES ($1, $2, UUID())
ON DUPLICATE KEY UPDATE updated_at = NOW();

-- Monitor with: SHOW PROCESSLIST, innodb_locks
```

MongoDB:
```javascript
// Atomic upsert with lock behavior
db.records.updateOne(
  {source_id: $1, external_id: $2},
  {$set: {canonical_id: ObjectId(), updated_at: new Date()}},
  {upsert: true}
);
// Monitor with: db.currentOp(), db.serverStatus()
```

**When it's a real bottleneck:**
- Per-row lock hold time: >10ms
- 1,000 concurrent inserts on same row: take >10 seconds
- AND new records are >10% of your overall traffic
- AND you can't reduce contention (e.g., with write replicas for reads)

**When it's NOT a bottleneck:**
- New record creation: <1% of traffic (typical systems: ~1%)
- Lock contention: only per-row/per-entity, not global
- Per-row latency <1ms means other operations don't stall
- Other concurrent ops (queries, updates on different rows) proceed normally

**Verdict:** If new record creation is <1% AND lock time is sub-millisecond, **not a real bottleneck**. Optimization is premature. Monitor with observability hooks; revisit if traffic patterns change.

#### Hypothesis 2: Debounce State Lock Contention (Per-Entity)

**Testing approach (database-agnostic):**

1. **Measure lock hold time** on state row
   - Lock a row, do minimal state update (change a few fields)
   - Expected hold time: milliseconds (not seconds)
   - Release lock
   - Benchmark: repeat 1,000 times for same entity; measure average/P99

2. **Measure per-entity throughput**
   - All 1,000 operations target same entity
   - Measure: Do they serialize (1,000 × lock_hold_time) or parallelize?
   - If serialized: throughput = 1 / lock_hold_time ops/sec
   - If parallel: check if your concurrency model supports it

3. **Verify other entities aren't affected**
   - While locking entity A, try operations on entity B
   - Expected: Entity B operations proceed unaffected
   - If entity B stalls: it's a global lock (worse)
   - If entity B is fine: it's per-entity (acceptable)

**Database-specific syntax (examples):**

PostgreSQL (row-level locking):
```sql
BEGIN;
  SELECT * FROM debounce WHERE entity_id = $1 FOR UPDATE;
  -- State update (milliseconds)
  UPDATE debounce SET last_update = NOW() WHERE entity_id = $1;
COMMIT;
```

MySQL (row-level locking):
```sql
SET SESSION innodb_lock_wait_timeout = 2;
BEGIN;
  SELECT * FROM debounce WHERE entity_id = ? FOR UPDATE;
  UPDATE debounce SET last_update = NOW() WHERE entity_id = ?;
COMMIT;
```

MongoDB (document lock):
```javascript
db.debounce.updateOne(
  {entity_id: entityId},
  {$set: {last_update: new Date()}},
  {writeConcern: {j: true}}  // Wait for journal
);
```

Redis (distributed lock):
```python
# Using redis-py
with redis_client.lock(f"debounce:{entity_id}", timeout=10):
    # State update (should be milliseconds)
    state = redis_client.get(f"state:{entity_id}")
    redis_client.set(f"state:{entity_id}", new_state)
```

**Expected results:**
- Lock hold time: milliseconds (5-100ms for state update, not seconds)
- Per-entity throughput: capped by serialization (one operation at a time)
- Different entities: independent locks (don't contend)
- Expensive work: should happen OUTSIDE lock (after acquiring state, release lock, then do work)

**Scaling ceiling with per-entity locking:**
```
If lock_hold_time = 10ms:
  Per-entity throughput = 1 / 0.01s = 100 ops/sec per entity
  Global throughput = 100 ops/sec × 1,000 entities = 100,000 ops/sec

If all traffic targets 1 entity:
  Throughput = 100 ops/sec (bottleneck)

If traffic distributed across 1,000 entities:
  Throughput = 100,000 ops/sec (no bottleneck)
```

**Verdict:** Per-entity lock is acceptable if:
- Lock hold time is milliseconds (5-100ms), not seconds
- Expensive work (reconciliation, computation, I/O) happens OUTSIDE the lock
- Different entities have independent locks (don't contend with each other)
- Your traffic is distributed across multiple entities (typical case)

Document as: 
```
"Debounce bottleneck: Per-entity lock serializes to X ops/sec per entity.
 If traffic distributed across N entities, global throughput = X × N ops/sec.
 If all traffic to 1 entity, throughput limited to X ops/sec.
 Current distribution: [provide traffic stats]"
```

#### Hypothesis 3: Database Write Throughput (Real Bottleneck)

**Testing approach (database-agnostic):**

1. **Measure baseline write throughput**
   - Insert N records into a single table (no conflicts, simple data)
   - Measure: How many records written per second?
   - Typical results:
     - Single-instance database: 500-2,000 writes/sec (depends on hardware, durability settings)
     - With read replicas: Same as single instance (replicas don't improve write throughput)
     - With async writes: Higher (but you lose durability guarantees)

2. **Test with realistic data size**
   - Use typical record size (not 1 byte, not 1MB)
   - Include indexes and constraints
   - Measure end-to-end: INSERT completes → acknowledged by database

3. **Identify the bottleneck**
   - Is it CPU? (check CPU usage during benchmark)
   - Is it disk I/O? (write-ahead log, index updates)
   - Is it network? (replication lag if using replicas)
   - Is it lock contention? (if concurrent writes to same rows)

**Database-specific syntax (examples):**

PostgreSQL:
```sql
-- Simple write throughput test
\timing on
INSERT INTO records (source_id, external_id, data)
SELECT i, 'ext-'||i, 'test data'
FROM generate_series(1, 10000) i;
-- Note: Time reported; extrapolate to ops/sec
```

MySQL:
```sql
SET AUTOCOMMIT = 1;
INSERT INTO records (source_id, external_id, data)
SELECT i, CONCAT('ext-', i), 'test data'
FROM (SELECT 1 i UNION ALL SELECT 2 UNION ALL ... SELECT 10000) nums;
```

MongoDB:
```javascript
// Write throughput test
const docs = [];
for (let i = 1; i <= 10000; i++) {
  docs.push({source_id: Math.floor(i/100), external_id: `ext-${i}`, data: 'test'});
}
await db.collection('records').insertMany(docs);
```

**Expected baseline results:**
- Single-instance database: ~1,000 writes/sec (typical)
- Read replicas: No improvement to write throughput (replicas are for reads)
- Batch inserts: Better than single inserts (reduces transaction overhead)
- Async writes: Much higher throughput (but durability compromised)

**Verdict:** If your **peak write traffic** exceeds database write throughput, this is a real bottleneck (unlike lock contention, which is per-entity).

**When it's a real bottleneck:**
- Your measured write traffic: 5,000 ops/sec
- Database throughput: 1,000 ops/sec
- Gap: 4,000 ops/sec (writes queue up)
- Result: Increasing latency, growing queue, eventual failures

**Mitigations (in order of preference):**
1. **Read replicas** — If query-heavy, move SELECT to replicas; keep writes on primary
2. **Connection pooling** — Reduce overhead of establishing connections
3. **Batch inserts** — Reduce number of transactions (more records per transaction)
4. **Async durability** — Write to memory, defer disk durability (careful: data loss risk)
5. **Database upgrade** — Better hardware (more CPU, faster disk, more RAM)
6. **Horizontal scaling** — Sharding writes across multiple databases (complex; affects identity resolution)

Document as:
```
"Database write bottleneck: Current throughput ~1,000 ops/sec.
 Peak traffic: 2,000 ops/sec (2x over capacity).
 Mitigation: Implemented connection pooling (est. +20% throughput).
 Target: Monitor and upgrade hardware if peak exceeds 1,500 ops/sec sustained."
```

#### Hypothesis 4: Worker Processing Throughput

**How to test:**
```
- Measure: How long does each message take to process?
  - P50: ___ ms
  - P99: ___ ms
  
- Calculate: Max throughput = 1000 / P50_ms messages/sec
  - With N workers in parallel: N * (1000 / P50_ms)

- Compare: Is queue depth growing? Are workers keeping up?
```

**Verdict:** If worker throughput < event arrival rate, you need more workers (add instances, not threads).

---

### Step 4: Classify Bottleneck

For each identified bottleneck, classify it:

| Type | Scope | Acceptable | Mitigation |
|------|-------|-----------|-----------|
| **Per-entity lock** | Entity A doesn't block B | Yes, if lock time <10ms | Separate DBs per entity (expensive) |
| **Per-source lock** | Source A doesn't block B | Yes, if <1% of traffic new | Read replicas; accept serialization |
| **Global DB throughput** | All writes serialized | Maybe | Replication, sharding, hardware |
| **Worker speed** | All workers too slow | Fixable | Add instances (horizontal scale) |
| **Message broker** | Broker can't keep up | Rare | Upgrade broker config or hardware |

**Document:** Which bottlenecks are acceptable, which require mitigation.

---

### Step 5: Challenge Proposed Solutions

When someone proposes a solution (sharding, Redis, etc.), test it against your traffic distribution:

#### Example: Proposed Solution = "Shard identity table by source"

**Question:** "What % of traffic is new record creation?"
- If <1%: Sharding won't help; wastes complexity
- If >10%: Might help, but test the contention claim first

**Question:** "How many sources are you sharding?"
- If 3-5 sources: Sharding doesn't parallelize much
- If 100+ sources: Sharding helps, but cost/benefit analysis needed

**Question:** "Will you accept eventual consistency in identity mapping?"
- If no: Sharding breaks the invariant (same ID → multiple canonical IDs)
- If yes: You need distributed consensus (expensive, complex)

**Verdict:** Unless measurements show per-source lock contention is actually limiting throughput AND new records are >5% of traffic, sharding identity tables is premature optimization.

---

### Step 6: Scaling Ceiling Documentation

**Document for each potential bottleneck:**

```markdown
## Bottleneck: Atomic Upsert on records(source_id, external_id)

**Measured characteristics:**
- Lock hold time: <1ms
- Tested with: 5,000 concurrent inserts
- New record creation: 0.8% of traffic (80 events/day in 10,000 event/day system)

**Scaling ceiling:**
- Per-source: Can sustain 1,000 new records/sec per source (test shows)
- Global: Not a bottleneck; 10 sources × 10 new records/sec = 100 new records/sec < 1,000 ceiling

**When this becomes a real bottleneck:**
- If new record creation goes above 5% of traffic
- If single source gets >1,000 new records/sec sustained
- Then consider: read replicas for queries, or evaluate sharding with distributed consensus cost

**Recommended mitigation for MVP:**
- None needed; contention is acceptable
- Future: Monitor with observability hooks on lock acquisition time
```

---

## Common Scaling Myths & Reality

### Myth 1: "Atomic upsert will serialize all writes"
**Reality:** Lock is per-row, not global. Different sources/entities don't contend. Contention only occurs if you're hammering the same (source, identifier) tuple.

### Myth 2: "We need to shard to scale"
**Reality:** For identity tables, sharding breaks the invariant (single ID → multiple canonical IDs). Measure traffic first; most systems don't shard identity.

### Myth 3: "Row locks are slow"
**Reality:** Postgres row locks are sub-millisecond. If they're slow, the problem is what's inside the lock (expensive computation), not the lock itself.

### Myth 4: "Cache will solve all bottlenecks"
**Reality:** Cache helps read throughput. For write-heavy operations (identity upserts), cache doesn't help unless you accept eventual consistency (risky for identity).

### Myth 5: "We need to shard because we're growing"
**Reality:** Measure traffic distribution first. Growth often means more entities, not more throughput per entity. Per-entity lock contention stays the same.

---

## Deliverable: Bottleneck Analysis Report

After analysis, create a one-page report:

**Title:** Scaling Bottleneck Analysis — [System Name]

**Section 1: Traffic Distribution**
- New record creation: X% of traffic
- Update vs insert ratio
- Peak throughput: Y events/sec

**Section 2: Identified Bottlenecks**
- Bottleneck A: Per-entity lock, hold time <1ms, acceptable ✅
- Bottleneck B: DB write throughput, at 70% capacity, monitor ⚠️
- Bottleneck C: Worker processing, scaling linearly with instances ✅

**Section 3: Scaling Ceiling**
- Global throughput: Z events/sec (limited by: DB writes)
- Per-entity throughput: W events/sec (limited by: debounce lock)
- Required before sharding: (if applicable) >1000 new records/sec globally

**Section 4: Proposed Changes**
- ✅ Keep as-is (bottlenecks are acceptable)
- ⚠️ Monitor these metrics (Bottleneck B)
- ❌ Reject these proposals (sharding, unless metrics change)

**Section 5: Measurement Hooks**
- Add observability for: lock acquisition time, queue depth, worker processing time
- Alert thresholds: (if applicable)

---

## Questions to Always Ask

When someone claims there's a bottleneck:

1. **"What metric shows this is a bottleneck?"** (latency? throughput? error rate?)
2. **"What traffic pattern triggers it?"** (specific entity? all entities? specific operation?)
3. **"At what throughput does it occur?"** (100 events/sec? 10,000?)
4. **"Is this per-entity or global?"** (lock contention on same row vs serialization of all writes)
5. **"What % of traffic exercises this code path?"** (if <1%, probably not a bottleneck)
6. **"Have you measured it or estimated?"** (measure before deciding)

If you get "I think it might be a bottleneck" without data, this skill applies. Measure first.

