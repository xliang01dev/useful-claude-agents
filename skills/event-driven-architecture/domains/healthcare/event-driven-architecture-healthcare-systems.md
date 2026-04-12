# Event-Driven Architecture: Healthcare Systems Specialization

**Companion to:** [SKILL.md](SKILL.md) - event-driven-architecture skill

This document extends the generic event-driven architecture skill with patterns specific to **multi-source healthcare systems** where patient/entity data arrives from multiple sources (claims, encounters, labs, pharmacies, etc.) and must be deduplicated and unified.

**Applies to:** Systems that require identity resolution across multiple external data sources mapping to one canonical entity.

---

## When to Use This Document

Use this specialization when your system has:
- **Multiple data sources** (Medicare, hospital systems, lab services, pharmacy networks, etc.)
- **Identity deduplication** (same patient from different sources must map to one canonical ID)
- **Atomic identity resolution** (INSERT OR UPDATE to unify records across sources)
- **High volume, multi-source ingest** (thousands of messages per second from heterogeneous sources)

---

## Extension 1: Identity Resolution Pattern

### The Pattern (Multi-Source Deduplication)

**Core Principle:** Single source of truth for identity across sources.

**What it solves:**
- Multiple external IDs for the same entity (Patient MED-123 in Medicare, Patient LAB-456 in hospital lab system)
- Need to map all external IDs to one canonical entity (UUID)
- Requiring atomicity: INSERT external ID → UPDATE canonical ID (no race conditions)

**Implementation:**

```
Primary identity table (source of truth):
  - CREATE TABLE entities (
      entity_id UUID PRIMARY KEY,
      source_id TEXT NOT NULL,        -- which system (Medicare, Hospital, Lab)
      external_id TEXT NOT NULL,      -- patient ID from that system
      created_at TIMESTAMP,
      UNIQUE(source_id, external_id)
    )
  
  - Atomic upsert on (source_id, external_id):
    INSERT INTO entities (entity_id, source_id, external_id) 
    VALUES (gen_random_uuid(), $source, $external_id)
    ON CONFLICT (source_id, external_id) 
    DO UPDATE SET ... RETURNING entity_id
```

**Read replicas:** All SELECT queries (identity lookup) go to read replicas; only UPSERT goes to primary.

### Probabilistic Matching Pre-Filter

External systems often have inconsistent ID quality. Use multi-stage matching:

1. **Exact match:** (source_id, external_id) → use canonical_id if found
2. **Probabilistic:** Match on demographics (name+DOB or insurance_id)
   - Score >0.95 → accept; 0.80-0.95 → manual review; <0.80 → new ID
3. **Temporal windowing:** MRN reused after 3+ years → create new ID

Fallback: New canonical_id on uncertain matches. Merging is manual but safe. Auto-merge on second appearance with strong match.

Use when: Integrating legacy systems (Medicare, hospital archives) with inconsistent IDs.

### Why NOT to Shard Identity

**Anti-pattern (per-source sharding):**
```
❌ Shard by source:
   - Medicare shard: MED-123 → uuid-x
   - Hospital shard: MED-123 → uuid-y (different entity!)
   - Lab shard: MED-123 → uuid-z (different entity!)

Result: Same patient has 3 canonical IDs → identity resolution broken
```

**Why it breaks:** Identity mapping is a **global invariant**. Sharding violates it by allowing the same external ID to map to different canonical IDs in different shards.

### Contention Analysis for Healthcare

**Lock behavior on atomic upsert:**
- Lock scope: **Per (source, external_id) tuple** — very fine-grained
- Lock hold time: **Sub-millisecond** (just state update, no expensive computation)
- Contention: **Per-source, not global**

**Example traffic:**
```
Medicare source:    100 new patients/day → ~1 per sec
Hospital source:    50 new patients/day  → <1 per sec
Lab source:         30 new patients/day  → <1 per sec
─────────────────────────────────────────────────
Total:             180 new patients/day  → ~2 per sec

Global upsert throughput available: ~1,000/sec
Actual new registrations: ~2/sec

Bottleneck?  NO — only 0.2% of capacity used
```

**When it COULD become a bottleneck:**
- Bulk import: 10,000 new patients all at once
- Same source flooding: 500 new patients/sec from one source
- Under these conditions: contention becomes visible, but it's **per-source serialization, not global**

### Bulk Import Hazards

Enterprise go-lives, claims backfill, and master syncs create temporary spikes (5 years of traffic in 1 hour).

**Mitigation:**
1. Use separate storage for bulk load (temporary, high-throughput)
2. Two-phase: Bulk load to temporary storage → merge to canonical (off-peak)
3. Monitor identity insertion ratio; alert if 5x baseline
4. Test bulk scenarios before go-live

---

### HL7 v2 vs FHIR: Format Fragility

Healthcare data arrives in multiple formats with parsing inconsistencies:
- **HL7 v2** (70% of integrations): Field separators, segment order, encoding all vary
- **FHIR** (modern): Better structured but identifier systems and extensions vary

**Defensive parsing pattern:**
1. Try-catch all field access; route missing IDs to dead-letter queue
2. Normalize to canonical form: extract ID, name, DOB, SSN with safe fallbacks
3. Handle malformed/missing: probabilistic match on demographics, manual review queue for errors

Ignoring fragility → silent dedup failures, lost records, duplicate IDs.

---

**Scaling ceiling:**
```
Per-source throughput: ~1,000 upserts/sec (typical Postgres row lock)
With 10 sources:     ~10,000 upserts/sec global
If any source exceeds 1,000/sec sustained, add write acceleration or queue dedupe work
```

---

## Extension 2: Healthcare-Specific Bottleneck Analysis

### Traffic Distribution in Healthcare Systems

**Key insight:** Healthcare event distribution is **heavily skewed toward updates.**

**Typical distribution:**
```
99% of events: Updates to existing entities
  - Claims for existing patients
  - Lab results for existing patients
  - Encounter records for existing patients
  - Prescription fills for existing patients

<1% of events: New entity registration
  - First-time patient record creation
  - New provider enrollment
  - New facility record

Example (100,000 events/day):
  99,000 events: Update existing records
  1,000 events:  Create new entities
```

### Why This Matters for Scaling

**Atomic upsert bottleneck ONLY exists if new entity creation is high:**

```
If new registration = 0.5% of traffic:
  ✓ Atomic upsert lock contention is acceptable
  ✓ Don't shard identity tables
  ✓ Measure actual throughput before declaring bottleneck

If new registration = 20% of traffic:
  ⚠️ Contention becomes measurable
  ⚠️ May need queue-based dedupe + batch upsert
  ⚠️ Or write-acceleration layer (memcached + async flush)
```

### Per-Entity Bottlenecks (Debounce Pattern)

**Healthcare-specific:** Reconciliation debounce on per-patient state.

**Pattern:**
```
Debounce table:
  - Row per patient (fine-grained lock)
  - Tracks: last_reconciliation_triggered, hold_until_timestamp
  - Lock semantics: FOR UPDATE during state check → decision → enqueue work
  
Behavior under high per-patient throughput:
  - Patient A: 1,000 events/sec → serialized by row lock
  - Patient B: 100 events/sec → independent, no contention with A
  - Patient C: 50 events/sec → independent
  - Global throughput: 1,150 events/sec (sum of per-patient)
```

**This is acceptable because:**
- Lock hold time: milliseconds (state update, not expensive computation)
- Expensive work (actual reconciliation) happens **outside the lock**
- Different patients don't contend (per-patient, not global)

**Scaling ceiling:**
```
Per-patient throughput: Limited by lock serialization
  - If patient gets 1,000 events/sec:
    - Each event acquires lock, updates state, releases lock
    - Throughput ≈ 1,000 events/sec per patient (depends on lock hold time)

Global throughput: Sum across all patients
  - If 100,000 patients, avg 10 events/sec each = 1M events/sec global
  - If 10 patients, avg 100 events/sec each = 1K events/sec global
  - Scaling is linear with patient count (fine-grained locks)
```

---

## Extension 3: Healthcare-Specific Architecture Decisions

### Identity Resolution vs Event Sourcing

**Healthcare systems need both:**

```
Event sourcing: All raw events immutable (claims, encounters, labs)
  - HIPAA audit trail requirement
  - Replay capability for investigations
  - Source-of-truth for what happened

Identity resolution: Deduplicate entities across sources
  - Map external IDs to canonical entities
  - Enable cross-source queries ("all records for patient X")
  - Clean data for analytics/reporting
```

**Pattern:**
```
Raw event streams (source of truth):
  - Store 100% of events from all sources
  - Keyed by (source_id, external_id, timestamp)
  - Immutable

Deduplicated entity streams (derived):
  - Enrich events with canonical_entity_id
  - Keyed by (canonical_entity_id, timestamp)
  - Derived from identity table lookups
```

### Conflict Resolution Strategies

When sources disagree (insurance denies claim, lab corrects result, hospital updates demographics):

**Pattern:** Store source-of-truth per field: `{value, source_id, timestamp}`

- Same source, newer timestamp → auto-update
- Different sources → manual review
- Time-based fallback: newer wins (unless domain rule overrides)

### Temporal Anchoring

Patient IDs change over time. Handle three scenarios:

1. **ID migration:** Link old canonical_id → new UUID (preserve ordering)
2. **Late corrections:** Use event timestamp, not arrival time, for identity lookup
3. **MRN reuse:** Track valid_from/valid_until per source+external_id per entity

**Pattern:** Store validity window per identity mapping. When looking up identity, check if event timestamp falls within valid range. Support entity merging and supersession tracking for ID migrations.

### Compliance-Driven Architecture

**Event streams are required, not optional:**

Reasons:
1. **HIPAA audit trails:** Must prove who accessed what, when
2. **Regulatory queries:** "Show all records for this patient from 2023-2024"
3. **Incident investigation:** "Replay all events for this patient to debug the reconciliation error"
4. **Data lineage:** Track which claims led to which reconciliation decision
5. **Temporal queries:** "Who was this patient on date X?" (identity changes over time)

→ Event streams (not queues) are the only pattern that satisfies these requirements.

---

## Extension 4: Key Principles for Healthcare Systems

1. **Single source of truth for identity** (not just single source of code, but single canonical ID mapping)
2. **Probabilistic matching before canonical assignment** (clean external IDs are rare; plan for demographic fallbacks)
3. **Per-entity bottlenecks are acceptable** if different entities don't contend
4. **New entity registration is rare** (<1% typical); don't over-optimize for it; BUT test bulk imports separately
5. **HL7/FHIR parsing fragility is unavoidable** (use defensive parsing, canonical normalization, dead-letter queues)
6. **Conflict resolution needs domain expertise** (auto-resolve same-source conflicts; queue different-source conflicts for review)
7. **Temporal anchoring matters** (store valid_from/valid_until; queries use event_timestamp, not arrival_timestamp)
8. **Event streams are required** (not optional) for audit trails, replay, and temporal queries
9. **Durability must precede effects** (reconciliation can't be triggered until state persisted)

---

## When to Remove/Relax These Patterns

**Stop using identity resolution pattern if:**
- You have a **single authoritative source** (no external sources to dedupe)
- Your system uses **immutable IDs from source** (no need to map)
- You're building a **single-tenant system** (no cross-tenant deduplication)

**Stop using event streams if:**
- You have **no audit/compliance requirements** (simple CRUD systems OK with queues)
- You don't need **replay capability** (state can be rebuilt from current DB)
- You don't need **temporal queries** ("state on date X")

---

## Reference Documents

For healthcare-specific deep dives, see:
- [event-driven-architecture-design.md](event-driven-architecture-design.md) — Architecture Design section (Identity Resolution)
- [event-driven-scaling-analysis.md](event-driven-scaling-analysis.md) — Scaling Analysis (debounce lock patterns, 99% update ratio)
- [learning-summary.md](../../learning-summary.md) — Original healthcare learnings (Section 4-6, 10-11)

---

## When to Move Back to Generic Skill

If you're building a **non-healthcare multi-source system** (e.g., e-commerce with marketplace sellers, SaaS with customer integrations) and you need identity deduplication, these patterns still apply but are **generalized:**

- Replace "patient" with "entity"
- Replace "source system" with "external tenant/partner"
- Same identity mapping invariant
- Same per-entity bottleneck analysis
- Same event sourcing for audit trails (if required by compliance)

The patterns are domain-agnostic; only the examples and compliance drivers are healthcare-specific.

