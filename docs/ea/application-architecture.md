# TOGAF Phase C — Application Architecture (Engine-Level)

> Folder: `ea/`  
> Format: **Markdown only (single continuous block)**  
> Diagrams: **Mermaid (renderable, color-styled)**  
> Purpose: Define the concrete, implementable application architecture for the Ultra-Fast OLTP Engine, strictly conforming to the locked Data Architecture. This is runtime-accurate: components, threading, data flow, ACID execution, failure handling, extensibility, and operability.

---

## A-A.1 Architectural Objectives
- **Deterministic execution:** Single-writer per shard, no locks in the hot path  
- **Engine-level ACID:** Protocol/log-based transactions, SQL never authoritative  
- **Memory-first, log-first:** In-memory execution with durable append before ACK  
- **Horizontal scale:** Shards as independent execution units  
- **Observability & Ops:** First-class metrics, tracing, and admin controls  
- **Extensibility without risk:** Pluggable state layouts, codecs, and projections under strict invariants  

---

## A-A.2 System Context (C4-Level)
![System Context - C4](images/mermaid-diagram-2026-01-10-115317.png "System Context C4")
[Alt text](docs/ea/images/mermaid-diagram-2026-01-10-115317.png)

