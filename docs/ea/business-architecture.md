# TOGAF Phase B — Business Architecture

> Folder: `ea/`
> Purpose: Define business capabilities, value streams, stakeholders, and business requirements for the Ultra-Fast OLTP Engine. This phase translates the Architecture Vision into business structure and outcomes.
> Governance: Constrained by the **Locked Decisions** in `State-continuation.md`.

---

## B.1 Business Scope

The platform provides a **general-purpose, financial-grade transactional runtime** for organizations that require:

* Ultra-low latency transaction execution
* Deterministic correctness and auditability
* Horizontal scalability across shards and data centers
* Developer extensibility without loss of safety

**Primary Consumers:** Platform engineers, product teams building workflows, financial/ledger systems, large-scale operational backends.

---

## B.2 Business Capabilities

### Core Capabilities

1. **Transaction Execution**

   * Accept and execute commands as engine-level transactions.
   * Enforce ACID semantics at the protocol/log level.

2. **Durability & Audit**

   * Append-only immutable log as source of truth.
   * Full replay and historical audit.

3. **State Management**

   * In-memory state machines per shard.
   * Snapshotting and deterministic recovery.

4. **Scalability & Partitioning**

   * Sharding, routing, and horizontal scale-out.
   * Cross-shard coordination via sagas.

5. **Query & Reporting Enablement**

   * Asynchronous projection to SQL stores.
   * Support analytical and operational queries.

6. **Observability & Operations**

   * Metrics, tracing, health, admin control.
   * Failover, restart, rebuild, and maintenance workflows.

### Supporting Capabilities

* **Security & Compliance** (encryption, access control, audit)
* **Developer Enablement** (SDKs, extension points, documentation)
* **Governance** (change management, configuration, versioning)

---

## B.3 Value Streams

### VS1: Execute a Transaction

1. Client submits command
2. Gateway authenticates & validates
3. Engine logs intent (durable)
4. In-memory execution (deterministic)
5. Client receives acknowledgment
6. Background projection updates SQL

**Outcome:** Ultra-fast, correct transaction with full audit.

### VS2: Recover From Failure

1. Node restarts
2. Load snapshot
3. Replay log from checkpoint
4. Rebuild in-memory state
5. Resume processing

**Outcome:** Zero data loss, deterministic recovery.

### VS3: Scale the Platform

1. Add shard/node
2. Update routing rules
3. Rebalance partitions
4. Continue processing without downtime

**Outcome:** Linear throughput growth with scale.

---

## B.4 Stakeholders & Concerns

* **Business Owners**: Reliability, compliance, scalability
* **Platform Architects**: Architectural integrity, patterns, governance
* **Developers**: Extensibility, clarity, testing support
* **Operations/SRE**: Observability, failover, operability
* **Security/Compliance**: Audit, encryption, data integrity

---

## B.5 Business Requirements

### Functional

* Support engine-level transactions with ACID semantics
* Support sharding and cross-shard coordination
* Provide SQL projections for queries and reporting
* Enable deterministic recovery and replay
* Provide administrative operations (pause, snapshot, rebuild)

### Non-Functional

* Microsecond-level execution latency (hot path)
* Zero data loss on crash
* Horizontal scalability to billions of records
* Full auditability
* High testability and maintainability

---

## B.6 Business Rules & Policies

* **Source of Truth Policy**: Durable log and engine protocols define correctness; SQL is not authoritative.
* **Change Governance**: All architectural changes must follow documented change-log discipline.
* **Quality Policy**: No compromise on code quality, architecture, performance, or testing.
* **Compliance Policy**: Auditability and data protection are mandatory.

---

## B.7 Business-to-IT Alignment

This Business Architecture mandates:

* Engine-managed transactions (not DB-managed)
* Protocol-based ACID
* Partitioned execution model
* Projection-based querying

These constraints directly drive the Data, Application, and Technology architectures in Phases C and D.

---

## B.8 Gaps & Risks

* **Skill Gap**: Requires advanced systems engineering expertise
* **Adoption Risk**: Learning curve for teams new to log-first architectures
* **Operational Complexity**: Sharding and replay require strong tooling

**Mitigation:** Phased delivery, strong documentation, automated operations.

---

## B.9 Phase B Deliverables

* Business Capability Map
* Value Stream Diagrams
* Stakeholder Matrix
* Business Requirements Catalogue
* Business Policies

---

## B.10 Next Steps (TOGAF)

* Phase C: Data Architecture
* Phase C: Application Architecture

---

*All content is governed by `State-continuation.md` (Locked Decisions & Change Log).*
