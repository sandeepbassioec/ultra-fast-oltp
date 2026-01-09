# TOGAF Phase A — Architecture Vision

> Folder: `ea/`
> Purpose: Establish the Architecture Vision for the Ultra-Fast OLTP Engine. This document sets scope, stakeholders, drivers, constraints, and success criteria. It is aligned with the locked decisions in `State-continuation.md`.

---

## A.1 Introduction

This document defines the **Architecture Vision** for a next-generation, ultra-fast OLTP execution engine built on .NET, following TOGAF, DDD, C4, and ArchiMate. The engine is **memory-first, log-first**, with **engine-level ACID**, and uses SQL databases strictly as **projection/audit stores**.

---

## A.2 Business Drivers

* **Latency**: Microsecond-level command execution for transactional workloads.
* **Correctness**: Financial-grade guarantees (atomicity, consistency, isolation, durability) at the engine/protocol level.
* **Scale**: Support billions of historical records and thousands of concurrent users via sharding and partitioning.
* **Reliability**: Deterministic recovery, failover, and auditability.
* **Developer Velocity (without compromise)**: Extensible in-memory layouts and pipelines without sacrificing safety, testability, or performance.

---

## A.3 Stakeholders & Concerns

* **Chief Architect**: Architectural integrity, TOGAF/DDD/C4/ArchiMate compliance.
* **Engineering Lead**: Performance, correctness, maintainability.
* **SRE/Operations**: Observability, failover, recovery, operability.
* **Security/Compliance**: Audit trails, encryption, tamper evidence.
* **Developers (Platform Consumers)**: Clear APIs, extensibility points, testability.

---

## A.4 Scope

### In Scope

* Engine-level transaction processing (ACID at protocol/log level)
* Durable append-only logs, snapshots, replay
* In-memory execution engine (single-writer per shard)
* SQL projection layer (async)
* Sharding, replication, failover
* Observability, admin operations

### Out of Scope (Phase-1)

* Global secondary indexes across shards
* Full consensus frameworks beyond log replication
* Embedded analytics inside the execution engine

---

## A.5 Architecture Principles (Derived from Locked Decisions)

* **Architecture First**: TOGAF, DDD, C4, ArchiMate are mandatory.
* **Performance as a Feature**: No convenience over latency/throughput.
* **Correctness by Design**: Engine-level ACID; SQL is not the source of truth.
* **Testability by Default**: Deterministic replay, failover, and recovery tests.
* **Extensible but Safe**: Custom in-memory layouts allowed only if determinism and durability are preserved.

---

## A.6 High-Level Solution Concept

```
Clients → Command Gateway → Shard Router
                           ↓
                    In-Memory Engine (per shard)
                           ↓
                    Durable Log (replicated)
                           ↓
                 Background Projection → SQL DBs
```

* **Execution**: Memory-first, single-writer per shard.
* **Durability**: Log-first (WAL), snapshots, replay.
* **Queries**: SQL projections only.

---

## A.7 Risks & Mitigations

* **Complexity** → Phased delivery; strong modularity.
* **Data Loss Risk** → Log-first durability; replication; fsync policies.
* **Cross-Shard Transactions** → Sagas with idempotency and compensation.
* **Operational Burden** → Built-in admin APIs, metrics, and tooling.

---

## A.8 Success Metrics

* P99 command latency: microseconds (in-memory path)
* Zero data loss under crash tests
* Recovery time: seconds via snapshot + replay
* Linear throughput with shard scaling
* 100% test coverage on core engine components

---

## A.9 Next Steps (TOGAF)

* Phase B: Business Architecture
* Phase C: Data & Application Architecture
* Phase D: Technology Architecture

---

*This document is governed by the Locked Decisions in `State-continuation.md`. Changes must be logged per project Change Log policy.*
