# Ultra-Fast OLTP Engine — State Continuation

> Purpose: Capture agreed requirements and open questions for a next‑gen, ultra‑fast OLTP execution engine (memory‑first, log‑first, SQL as ultimate store). This file persists our shared state so we can resume later.

---

## 0. Vision

Build a **.NET Core / C#** based transaction engine where **business logic executes in memory** for microsecond latency, **durability is guaranteed via a write‑ahead log**, and **SQL/DB is used as a projection/audit/query store**. The engine must be suitable for **financial‑grade correctness**, **billions of records**, and **thousands of concurrent users** through **partitioning/sharding**, **single‑writer per shard**, and **deterministic execution**.

---

## 1. Non‑Negotiable Principles

1. **Log‑First Durability**: Every command is appended to a durable log (fsync) before it mutates in‑memory state.
2. **Memory‑First Execution**: Domain logic runs in memory; DB is not in the hot path.
3. **Deterministic Processing**: Single‑writer per shard; no locks inside the execution engine.
4. **Eventual SQL**: Database is a projection store for queries, audit, and analytics.
5. **Crash Safety**: Recover by snapshot + log replay; never rebuild core state from SQL.
6. **Horizontal Scale**: Partitioned/sharded execution with linear scalability.

---

## 2. Comprehensive Requirements (Research-Informed)

Below is a **full-scale** requirement set distilled from patterns used by high-performance, event/log-based systems (single-writer engines, event stores, partitioned logs, virtual-actor runtimes).

### 2.1 Functional Requirements

**Command & Transaction Processing**

* Command API (sync/async) with correlation IDs, idempotency keys, and deterministic ordering.
* Per-shard sequencing (monotonic offset) and per-aggregate versioning.
* Validation pipeline: pre-validation, domain validation, authorization, policy checks.
* Hooks: before/after command execution; before/after projection.
* Exactly-once execution semantics within a shard; at-least-once option for projections.

**Durable Log (Source of Truth)**

* Append-only segmented log per shard (WAL-style) with checksum, compression, and encryption.
* fsync/flush policy options (sync, group-commit, async) with configurable durability SLA.
* Log compaction policies: none, key-compaction, time-based retention (cold archive).
* Log replication: leader/follower with configurable replication factor.
* Log replayer: deterministic replay with back-pressure and progress checkpoints.

**In-Memory Execution Engine**

* Single-writer state machine per shard (no internal locks).
* Pluggable state storage: hash map, columnar-in-memory, adjacency lists, pre-joined arrays.
* Pluggable command handlers: dispatch by command type, version, tenant.
* Support multiple state “domains” per shard (bounded contexts / modules).

**Snapshots**

* Snapshot creation schedule: time-based, count-based, or adaptive.
* Snapshot formats: binary, JSON, custom codecs; compression and encryption.
* Incremental snapshots (optional) and snapshot compaction.
* Snapshot restore with schema/version migrations.

**Query & Projection Layer (DB as Ultimate Store)**

* Projection writers (SQL) supporting bulk modes (batch insert/update, TVP, copy APIs).
* Projection rebuild: drop and regenerate from log.
* Multiple projection targets (SQL primary; optional secondary stores later).
* Read APIs: point reads, range reads, secondary-index queries via DB projections.

**Caches & Optimization**

* Hot-key caches (LFU/LRU) and admission control.
* Special caches for frequently used lookup/master data.
* Query pattern recording: capture query shapes, filters, hot paths; export telemetry.
* Auto-suggest projection/index candidates based on observed query patterns.

**Multi-Tenancy & Isolation**

* Tenant routing to shards; quotas per tenant.
* Data isolation: encryption keys per tenant (optional).

### 2.2 Non-Functional Requirements

**Latency/Throughput**

* Microsecond-level in-memory execution target.
* Bounded tail latency via single-writer design, back-pressure, and preallocation.

**Correctness**

* Deterministic ordering per shard.
* Idempotent command processing.
* Auditable history: immutable log + projection ledger tables.

**Reliability/Availability**

* Failover: promote follower to leader using replicated log.
* Self-healing: shard rebalancing, automatic restart, replay continuation.

**Scalability**

* Horizontal scale by adding shards/nodes.
* SQL scale via partitioned tables (by shard/time) and appropriate indexing.

**Security & Compliance**

* Tamper-evident logs (hash chaining optional).
* Encryption at rest (log/snapshots) and in transit.
* RBAC for admin operations.

**Operability**

* Metrics: per-shard queue depth, latency percentiles, replay rate, replication lag.
* Admin operations: pause/resume shard, force snapshot, rebuild projections, drain queues.
* Upgrade strategy: rolling upgrades with log format/schema versioning.

### 2.3 Developer Extensibility Requirements (Key Differentiator)

* Pluggable **state layout** (custom in-memory structures) per domain.
* Pluggable **snapshot codec** and **log serializer**.
* Custom join/access strategies inside memory (no DB joins), e.g. precomputed indexes.
* Extension points for scheduling policies, durability policies, cache policies.

---

## 3. Scale & Concurrency

### 3.1 Sharding / Partitioning

* **Shard router**: hash by business key (e.g., AccountId, OrderId).
* **One execution thread per shard** (no locks).
* One durable log per shard.

### 3.2 Cross‑Shard Operations

* **Saga / orchestration** (reserve → confirm → commit).
* Idempotent steps; no distributed SQL transactions.

### 3.3 Throughput Targets

* In‑memory exec: ~1–5 µs
* Log append: ~5–20 µs
* SQL projection: async, batch (thousands–millions rows/sec)

---

## 4. Reliability & Failover

### 4.1 Replication

* **Active–Passive or Active–Active** log replication.
* Secondary can **take over by replaying replicated log**.

### 4.2 Operational Safety

* Back‑pressure on shard queues.
* Circuit breakers for downstream (DB) outages.

---

## 5. Caching & Optimization (Advanced)

### 5.1 Realtime Cache

* In‑memory state is the primary cache for hot entities.

### 5.2 Frequency‑Based Caches

* LRU/LFU layers for most‑accessed keys or aggregates.

### 5.3 Query Pattern Recording

* Telemetry to record read patterns on projections.
* Auto‑suggest or auto‑build **specialized projections / indexes**.

---

## 6. Developer Extensibility (Your Key Ask)

### 6.1 Pluggable Snapshot / State Layout

* **Custom snapshot codecs** (binary/JSON/flat buffers).
* Ability for developers to **tune in‑memory data structures** for specific algorithms (e.g., pre‑joined arrays, adjacency lists, hash maps, columnar in‑memory layouts).

### 6.2 Execution Pipelines

* **Composable command pipelines** (pre‑validation → execute → post‑hooks).
* Option for **compiled/templated pipelines** for hot paths.

### 6.3 Custom Join / Access Strategies

* APIs to define **how state is organized and accessed** (e.g., by key, by range, by graph traversal) without involving DB joins.

---

## 7. Data Model

### 7.1 Hot vs Cold

* **Hot State** (memory): current balances, active workflows, open orders.
* **Cold State** (DB): ledger/history, audits, archives (billions of rows).

### 7.2 Versioning

* Schema/version metadata in log entries and snapshots.
* Forward/backward compatibility strategies.

---

## 8. Observability & Ops

* Structured metrics: per‑shard latency, queue depth, replay rate.
* Tracing for commands → log → memory → DB.
* Admin APIs: pause shard, snapshot now, replay, rebuild projection.

---

## 9. Security & Compliance (Financial‑Grade)

* Immutable audit log.
* Role‑based access for admin operations.
* Data encryption at rest (logs, snapshots) and in transit.

---

## 10. Out of Scope (for MVP)

* Full distributed consensus (Raft/Paxos) beyond log replication.
* Global secondary indexes across shards.
* Real‑time analytics inside the execution engine (use DB/columnar).

---

## 11. Open Questions (To Refine Next)

1. **Durable Log**: File‑based WAL, custom binary log, or Kafka‑style segment store?
2. **Snapshot Format**: Binary vs JSON vs FlatBuffers? Compression?
3. **Replication Mode**: Sync vs async; quorum requirements?
4. **Cross‑Shard Protocol**: Saga semantics vs two‑phase commands.
5. **Projection Stack**: EF only, or EF + bulk/TVP; optional ClickHouse/columnar?
6. **Dev APIs**: How much control to expose for custom in‑memory layouts without breaking safety?

---

## 12. Phased Delivery

### Phase‑1 (Single Node)

* One shard, durable log, in‑memory state, snapshots, async SQL projection.

### Phase‑2 (Scale Out)

* Multi‑shard routing, replication, failover, cross‑shard saga.

### Phase‑3 (Self‑Optimization)

* Query pattern recording, hot caches, auto‑projections.

---

## 13. Success Criteria

* Microsecond‑level command execution.
* Zero data loss on crash (log‑first).
* Recovery in seconds via snapshot + replay.
* Handles billions of historical records in DB while keeping hot state minimal.
* Extensible for developers to tune in‑memory algorithms and layouts.

---

*This document is the authoritative state for the project. Continue from here in future sessions.*

---

## 14. ACID Guarantees (Engine-Level)

### Atomicity

* Idempotent commands with correlation IDs.
* Multi-step protocol for cross-shard operations (prepare/commit/compensate).
* Durable intent logged before in-memory mutation.

### Consistency

* Domain invariants enforced in state machines (DDD aggregates).
* Versioned state; no side-effects outside engine during execution.

### Isolation

* Single-writer per shard ensures serializable execution within shard.
* Cross-shard flows coordinated via messages/sagas (no distributed DB transactions).

### Durability

* Write-ahead durable log (fsync or replicated commit) before ACK.
* Snapshots + deterministic replay for recovery.
* SQL projections are derivable; not the source of truth.

---

## 15. Transaction Model

* Native engine transactions supported at core (command boundaries = transaction scope).
* Optional grouping of commands into higher-level business transactions (sagas).
* Exactly-once semantics within shard; at-least-once for projections.

---

## 16. SQL/DB Integration

* Ultimate store for projections, audit, and analytics (SQL Server, Oracle, MySQL, PostgreSQL).
* Partitioned tables by shard/time; optional columnstore for history.
* Rebuild projections from log; never rebuild core state from SQL.

---

## 17. Document Set (Deliverables)

* Requirements Specification (this document sections 0–16).
* TOGAF: Architecture Vision, Business, Data, Application, Technology, Transition.
* DDD: Bounded Contexts, Aggregates, Ubiquitous Language.
* C4: Context, Container, Component, Code.
* ArchiMate: Business/Application/Technology layers and relations.
* System Design (HLD) and Low-Level Design (LLD).

---
## 18. Locked Decisions (Non-Negotiables)

The following principles are **locked** and must not be compromised at any stage of design or implementation:

1. **Code Quality First**
   - Clean code, SOLID principles, clear boundaries, meaningful naming.
   - No shortcuts or technical debt for speed of delivery.

2. **Architecture Above All**
   - TOGAF, DDD, C4, and ArchiMate compliance is mandatory.
   - Layered, modular, and evolvable architecture only.

3. **Performance Is a Primary Requirement**
   - Ultra-fast execution paths (memory-first, log-first).
   - No architectural decisions that trade correctness or simplicity for convenience at the cost of latency or throughput.

4. **Testing Is Non-Negotiable**
   - Unit tests, component tests, and contract tests required for all core modules.
   - Deterministic, repeatable test harnesses for recovery, replay, and failover.

5. **Functional & Non-Functional Requirements Are Binding**
   - All requirements documented in this file (sections 0–17) must be met.
   - Scalability, durability, reliability, security, and operability are first-class concerns.

6. **No Compromise for Timeline**
   - Delivery time (weeks/months) must not reduce quality, architecture, performance, or correctness.
   - Refactoring and redesign are acceptable to preserve principles.

7. **Engine-Level Transactions Are Mandatory**
   - Native transaction semantics (ACID at engine level) must be preserved even under sharding and failover.

8. **SQL Is Not the Source of Truth**
   - The durable log and engine protocols define correctness.
   - SQL remains a projection, audit, and analytics store only.

9. **Developer Extensibility Without Safety Loss**
   - Custom in-memory layouts, snapshot codecs, and execution pipelines are supported **only** if they preserve determinism, durability, and testability.

10. **Documentation Is Part of the System**
    - Architecture, invariants, protocols, and extension points must be documented alongside code.

> These decisions are **frozen** and may only be changed by explicit revision of this document.

---

## 19. Change Log

All material changes to this document must be recorded here. Follow EAVIP-style discipline: **what changed, where, and why**.

- **2026-01-09**
  - **Added:** *Section 18 – Locked Decisions (Non-Negotiables)*
    - **Where:** New section appended at the end of the document.
    - **Why:** To formally freeze quality, architecture, performance, testing, and ACID requirements as non-negotiable project constraints.

---
## Summary
- Added Section 18: Locked Decisions (Non-Negotiables)
- Added Section 19: Change Log (EAVIP-style)

## Rationale
Formally locked code quality, architecture, performance, testing, and ACID guarantees as non-negotiable constraints and introduced a mandatory change-log discipline.

*This file is now the canonical state for the project. Continue from here for all future design work.*
