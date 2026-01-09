# TOGAF Phase C — Data Architecture (Engine-Level)

> Folder: `ea/`  
> Format: **Markdown only**  
> Purpose: Define the physical, logical, and operational data architecture for the Ultra-Fast OLTP Engine. This document is implementation-grade: file formats, memory layouts, recovery semantics, cross-shard protocols, tiering, and verification.  
> Governance: Constrained by the **Locked Decisions** in `State-continuation.md`.

---

## C-D.1 Architectural Objectives
- **Authoritative Source of Truth:** Durable append-only logs (not SQL)
- **Deterministic Recovery:** Snapshot + replay with verifiable state
- **Performance:** Sequential I/O, cache-friendly in-memory layouts
- **Correctness:** Engine-level ACID via protocol and log
- **Scalability:** Shard/partition aware at every data layer
- **Observability:** Checksums, offsets, watermarks for auditing

---

## C-D.2 Data Domains & Ownership

### D1 — Command & Transaction Metadata
- CommandId, CorrelationId, CausationId, TenantId
- Idempotency keys, version, schema id
- Ownership: **Execution Engine**

### D2 — Durable Log (Authoritative)
- Immutable, ordered, segmented
- Ownership: **Durability Subsystem**

### D3 — In-Memory Hot State
- Aggregate/state-machine data structures
- Ownership: **Shard Executor**

### D4 — Snapshots
- Point-in-time images bound to log offsets
- Ownership: **Snapshot Manager**

### D5 — Projections / Read Models (SQL)
- Denormalized, query-optimized
- Ownership: **Projection Subsystem** (derived)

---

## C-D.3 Source of Truth & Invariants
- **Truth:** Durable Log (D2)
- **Execution:** In-Memory State (D3)
- **Query/Audit:** SQL Projections (D5)  
**Invariant:** Core state is **never** rebuilt from SQL. Recovery uses **Snapshots + Log Replay** only.

---

## C-D.4 Information Lifecycle

### Write Path
Client → Command Gateway → Durable Log (fsync/replicated)  
→ In-Memory Execution → Background Projection → SQL

### Recovery Path
Start → Load Snapshot → Replay Log → Rebuild In-Memory State → Resume

### Rebuild (SQL) Path
Drop Projections → Replay Log → Recreate Projections

---

## C-D.5 Physical Storage: Durable Log

### C-D.5.1 Segment File Layout
<segment-file>  
Header:  
Magic(4) | Version(2) | SegmentId(8) | CreatedAt(8) | Flags(2)  
Records[]:  
[Offset:uint64]  
[RecordLength:uint32]  
[CRC:uint32]  
[Type:uint8]   // Prepare | Commit | Compensate | Data | SnapshotMarker  
[CommandId:uuid]  
[CorrelationId:uuid]  
[TenantId:bytes]  
[ShardKey:bytes]  
[SchemaId:uint16]  
[Payload:bytes]

**Properties**
- Append-only, sequential I/O
- Sparse side-index every N records: (Offset → FilePosition)
- Group-commit or per-record fsync (configurable)
- Optional encryption-at-rest; checksum per record

### C-D.5.2 Log Compaction & Retention
- **Time-based:** retain last X days
- **Key-compaction:** keep latest record per AggregateId
- **Archival:** sealed segments to object storage (WORM)

---

## C-D.6 Snapshots

### C-D.6.1 Snapshot File Layout
<snapshot-file>  
Header:  
Magic(4) | Version(2) | ShardId(4) | LogOffset(8) | CreatedAt(8)  
Body:  
[CodecId:uint16]  
[Compression:uint8]  
[Encrypted:uint8]  
[StateBytes:bytes]

- **Binding:** LogOffset anchors recovery point
- **Types:** Full, Incremental
- **Validation:** Snapshot CRC + optional Merkle hash of state

### C-D.6.2 Snapshot Scheduling
- Count-based (every N records)
- Time-based (every T minutes)
- Adaptive (memory pressure / replay SLA)

---

## C-D.7 In-Memory State Layouts (Pluggable)

### A) Aggregate Map (Default)
Dictionary<AggregateId, AggregateState>

### B) Columnar In-Memory
struct ColumnStore {  
AggregateId[] ids;  
long[] version;  
decimal[] balance;  
int[] status;  
}

### C) Pre-Joined / Indexed Layouts
Dictionary<Key, RowPointer>  
RowPointer → contiguous memory block

### D) Graph / Adjacency (Workflows)
Dictionary<NodeId, EdgeList>

---

## C-D.8 Cross-Shard Data Protocol (Saga)

### Records
Prepare(TxId, ShardId, AggregateId, Delta)  
Commit(TxId)  
Compensate(TxId)

**Rules**
- Idempotent per (TxId, ShardId)
- Exactly-once within shard
- Recovery: Prepare without Commit ⇒ Compensate

---

## C-D.9 Consistency Windows & Projection Lag
- **Strong:** In-memory state (authoritative)
- **Eventual:** SQL projections
- **Monotonicity:** Projections advance strictly by log offset
- **Watermarks:** ShardOffset, ProjectionOffset

---

## C-D.10 SQL Projections (Canonical Schemas)

### History / Ledger (Append-Only)
Ledger(ShardId, AggregateId, TxId, Type, Amount, OccurredAt)  
Partitioned by (ShardId, OccurredAt)

### Current State (Upsert)
CurrentState(ShardId, AggregateId, Balance, Status, Version)  
Clustered on (ShardId, AggregateId)

**Indexing**
- Hot-path composite indexes
- Columnstore for historical scans

---

## C-D.11 Tiering: Hot / Warm / Cold
- **Hot:** In-memory + recent log segments (NVMe)
- **Warm:** SQL primary
- **Cold:** Archived log segments (object storage)

---

## C-D.12 Rebuild, Verification, and Auditing

### Full Rebuild
Drop Projections → For each shard: Load Snapshot → Replay Log → Emit Projections

### Consistency Verification
Compare(InMemoryChecksum, SQLAggregateChecksum)  
Assert(ProjectionOffset == LogOffset)

### Audit
- Immutable log with offset-based trace
- Reproducible state from snapshot+log

---

## C-D.13 ACID at Data Layer
- **Atomicity:** Prepare/Commit/Compensate records
- **Consistency:** Invariants in state machine
- **Isolation:** Single-writer per shard
- **Durability:** fsync/replicated append before ACK

---

## C-D.14 Risks & Mitigations
- **Log Growth:** Compaction, tiering
- **Replay Time:** Frequent snapshots; parallel per shard
- **Projection Drift:** Periodic rebuild & checksums

---

## C-D.15 Deliverables
- Physical log & snapshot schemas
- In-memory layout catalog
- SQL projection schemas & indexing rules
- Saga protocol specification
- Tiering, rebuild, and audit procedures

---

## C-D.16 Deep Internals — Replication, Memory, Indexing, Evolution

### C-D.16.1 Log Replication & Consensus
**Topology:** Per-shard leader/follower with epoch-based leadership.  
**Record Pipeline:** Client → Leader.Append(Prepare) → fsync/group-commit → Replicate to Followers → Majority ACK → Leader.Append(Commit) → ACK  
**Replica Frame:** Epoch | LeaderId | ShardId | BaseOffset | Count | CRC | Records[]  
**Semantics:** Write quorum for Prepare; Commit after quorum; idempotence via (TxId, Offset); leader change requires highest (Epoch, Offset)  
**Failures:** Follower lag → catch-up; split-brain → epoch fencing

### C-D.16.2 Memory Ownership, Pooling & GC Strategy
**Principles:** No allocations on hot path; shard-local arenas  
**Design:** IMemoryArena.Allocate(size); Reset()  
- Object pools, pinned buffers, epoch-based reclamation  
**GC Mitigation:** Struct-of-arrays, Span/Memory

### C-D.16.3 Secondary Indexing (In-Memory)
**Types:** Hash, B+Tree, Bitmap, Bloom  
**Maintenance:** On Apply → update primary state → update secondary indexes  
**Constraints:** Deterministic, idempotent

### C-D.16.4 Compression & Encoding
- Log: LZ4
- Snapshots: ZSTD + delta
- In-memory: dictionary encoding, bit-packing

### C-D.16.5 Schema Evolution & Migrations
**Versioning:** SchemaId per record  
**Rules:** Backward-compatible readers; forward via defaults  
**Paths:** Lazy (on write), Eager (offline replay)

### C-D.16.6 Compaction & Garbage Collection
- Key-compaction, time-compaction, tombstones with TTL

### C-D.16.7 Integrity, Verification & Forensics
- Merkle trees over segments; end-to-end checksums; deterministic codecs

---

## C-D.17 Lock & Governance of Data Architecture

### C-D.17.1 Lock Statement
Sections **C-D.1 through C-D.16** are **LOCKED**. All subsequent design must conform. Changes require amendment with impact analysis and Change Log entry.

### C-D.17.2 Change Control
- Proposals: What, Where, Why, Risk, Rollback
- No breaking changes without migration + replay validation

### C-D.17.3 Acceptance Criteria
- Deterministic recovery (replay tests)
- Exactly-once per shard
- Zero allocations on hot path

---

## C-D.18 Data Architecture Change Log
- **2026-01-09**
  - **Added:** C-D.16 Deep Internals — for implementation-grade durability, performance, correctness
  - **Added:** C-D.17 Lock & Governance — to freeze Data Architecture before Application Architecture

---

*End of Data Architecture. Proceed to Application Architecture only after this section is accepted and locked.*
