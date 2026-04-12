---
name: event-driven-architecture
description: Use when designing or troubleshooting event-driven systems - guides architectural decisions, scaling analysis, integration patterns, and infrastructure setup
---

# Event-Driven Architecture Skill

**Generic patterns for any event-driven system.**

---

## Choose Your Workflow

| Situation | Workflow | Deliverable |
|-----------|----------|-------------|
| Planning a new event-driven feature | [Architecture Design](#1-architecture-design) | 1-page Architecture Decision Document |
| Someone claims there's a bottleneck | [Scaling Analysis](#2-scaling-analysis) | Bottleneck Analysis Report |
| Integrating APIs or consumer groups | [Integration Patterns](#3-integration-patterns) | Tested integration code |
| Adding/upgrading infrastructure | [Infrastructure Setup](#4-infrastructure-setup) | infrastructure-versions.md |

---

## 1. Architecture Design

**Outcome:** Design a scalable, redundant, and recoverable event-driven architecture. Choose the most appropriate model (event streams, message bus, message queue, etc.) and document all key decisions for alignment before implementation.

**When:** Planning or redesigning event-driven systems before code

**Patterns & Principles:**
- Broker type: Event streams for compliance/audit needs; queues for ephemeral work only
- Work distribution: Generic topics + consumer groups (never topic sharding) - enables horizontal scaling
- Worker architecture: Stateless instances, scale by adding containers
- Durability: Write-through pattern required when state triggers side effects - persist first, then trigger, then cache (optional)
- Ordering guarantees: Preserve order per entity (partition key = entity ID), not globally - global ordering creates bottlenecks. Document which events require order.
- Idempotency: Consumers must handle duplicate messages (network retries, broker rebalancing). Use request IDs or checksums to detect duplicates.
- API design: Endpoints should reflect data model access patterns, not database schema

**Reference:** [event-driven-architecture-design.md](event-driven-architecture-design.md)

---

## 2. Scaling Analysis

**Outcome:** Identify real bottlenecks through measurement; separate facts from assumptions. Document scaling ceilings per component and prevent costly premature optimization.

**When:** Measuring bottlenecks before declaring architectural constraints

**Patterns & Principles:**
- Measure before deciding: Collect traffic distribution (read vs write, concurrent users, message sizes) - don't assume bottlenecks exist
- Test systematically: Measure lock contention, database throughput, worker processing speed - measure actual impact, not assumptions
- Classify accurately: Per-resource bottlenecks vs global; different mitigations apply to each
- Document scaling limits: Know your ceiling before declaring architectural constraints; measure before optimizing

**Reference:** [event-driven-scaling-analysis.md](event-driven-scaling-analysis.md)

---

## 3. Integration Patterns

**Outcome:** Implement reliable external API and consumer group integration with proper error handling, defensive parsing, and correct consumer semantics. Prevent runtime failures and data inconsistencies.

**When:** Integrating external APIs, SDKs, consumer groups, or parsing responses

**Patterns & Principles:**
- SDK method choice: Match to use case - chat API for structured messaging with roles/context; generate for simple completion only
- Consumer identity discipline: durable_name per instance (for resumption), deliver_group shared (for group membership) - never use same value; enables per-instance tracking
- Defensive parsing saves debugging: External APIs are non-deterministic - wrap in try-catch, use safe field access (.get), always provide sensible fallbacks
- Configuration-schema alignment: Examples must match database constraints exactly (e.g., "medium" not "med"); misalignment causes constraint violations
- Idempotency by default: Treat all messages as potentially duplicated. Implement via request ID deduplication (check if ID already processed) or checksums. Acknowledge only after dedup check passes.
- Error handling & retries: Use exponential backoff (200ms → 400ms → 800ms) with max retries before routing to dead-letter queue. Don't retry on permanent errors (validation failures, constraint violations); do retry on transient errors (timeouts, service unavailable).
- Publisher/Consumer pattern: Publisher stateless (generic topic, no routing); consumer acknowledges only after successful processing

**Reference:** [event-driven-integration.md](event-driven-integration.md)

---

## 4. Infrastructure Setup

**Outcome:** Establish stable, versioned infrastructure that prevents cascade failures from version mismatches. Support architecture decisions with properly configured services and compatibility verification.

**When:** Adding or upgrading observability/infrastructure (Jaeger, Loki, Prometheus, OpenTelemetry)

**Patterns & Principles:**
- Pin versions explicitly, never `latest`: Verify image:tag correspondence actually matches version - cached images can be stale; prevents cascade failures
- Read migration guides first: Major version upgrades have breaking changes - v1→v2 transitions are often architectural rewrites, not incremental; requires config rewrite
- Verify SDK/exporter/instrumentation compatibility: Pin all to same minor version range - version mismatches cascade quickly (e.g., SDK incompatible with exporter causes import errors)
- Test locally before deploying: Validate config syntax; simplify single-instance configs (remove HA features, disable WAL, use inmemory kvstore) - enterprise configs add unnecessary complexity

**Checklist:**
- [ ] Pin all versions explicitly (no `latest` tags)
- [ ] Verify image name matches version (jaegertracing/all-in-one ≠ jaegertracing/jaeger)
- [ ] Read breaking changes for major versions
- [ ] Check OpenTelemetry compatibility (SDK + exporter + instrumentation same minor version)
- [ ] Test config locally (docker-compose config, kubectl --dry-run, terraform validate)
- [ ] Simplify single-instance configs (remove HA features, use inmemory kvstore, disable WAL)
- [ ] Document in infrastructure-versions.md

**Reference:** [infrastructure-version-management.md](infrastructure-version-management.md)

---

## Key Principles

1. **Measure before deciding** — Don't assume bottlenecks
2. **Durability before effects** — Persist state before triggering side effects
3. **Stateless workers scale** — Instances not threads (horizontal, not vertical)
4. **Handle duplicates, not exactly-once** — At-least-once + idempotency beats impossible exactly-once
5. **Order per entity, never globally** — Global ordering kills scalability; partition by entity ID
6. **Explicit versions prevent cascades** — No `latest` tags
7. **Defensive parsing saves debugging** — Always have fallbacks
8. **Backpressure matters** — If consumers can't keep up, broker queues fill; add circuit breakers or scale workers
9. **Match access patterns to endpoints** — Don't optimize for schema, optimize for queries
10. **Document the why** — Write down reasoning, not just what

---

## Common Questions

**Should I use event streams or queues?**
Event streams for compliance/audit needs; queues for ephemeral work only. → See Architecture Design

**How do I set up consumer groups?**
Unique durable_name per instance, shared deliver_group across all. → See Integration Patterns

**Can I copy the enterprise infrastructure config?**
No. Find "single-instance" example and remove HA features for MVP. → See Infrastructure Setup

**How do I identify real bottlenecks?**
Measure first. Test hypotheses with actual data. Don't assume. → See Scaling Analysis

**Can I achieve exactly-once semantics?**
No. Distributed systems guarantee at-most-once (message lost) or at-least-once (message duplicated). Choose at-least-once and handle duplicates with idempotency (request IDs, deduplication). → See Integration Patterns

**How do I preserve message order?**
Order per entity (use entity ID as partition key), never globally. Global ordering creates bottlenecks. Document which events require ordering. → See Architecture Design

---

## Context-Specific Architectures

These specializations extend the generic patterns for specific domains where additional constraints or patterns apply:

- **Healthcare/Multi-Source Systems:** If you're building a system that deduplicates entities across multiple external sources (Medicare, hospital systems, labs, pharmacies), see [domains/healthcare/event-driven-architecture-healthcare-systems.md](domains/healthcare/event-driven-architecture-healthcare-systems.md) for patterns on identity resolution, per-entity bottlenecks, and compliance-driven architecture.

---

## All Reference Documents Are Language-Agnostic

Examples provided in: Python, JavaScript/Node.js, Go, Java
