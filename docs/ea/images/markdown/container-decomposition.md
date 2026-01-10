```mermaid
flowchart TB
  classDef core fill:#e8f5e9,stroke:#2e7d32,color:#1b5e20;
  classDef io fill:#e3f2fd,stroke:#1e88e5,color:#0d47a1;
  classDef data fill:#fff3e0,stroke:#ef6c00,color:#e65100;
  classDef ops fill:#f3e5f5,stroke:#6a1b9a,color:#4a148c;

  subgraph Edge["Edge / API Layer"]:::io
    Gateway["Command Gateway"]
    QueryAPI["Query API"]
  end

  subgraph Core["Execution Core"]:::core
    Router["Shard Router"]
    ShardExec["Shard Executor(s)"]
    TxMgr["Transaction Protocol"]
    MemArena["Memory Arenas"]
  end

  subgraph Durability["Durability & Recovery"]:::data
    LogWriter["Durable Log Writer"]
    SnapshotMgr["Snapshot Manager"]
    Replay["Replay Engine"]
  end

  subgraph ReadSide["Read / Projection"]:::data
    Projector["Projection Dispatcher"]
    SQL["SQL Databases"]
  end

  subgraph Ops["Observability & Admin"]:::ops
    Metrics["Metrics / Tracing"]
    Admin["Admin API"]
  end

  Gateway --> Router
  Router --> ShardExec
  ShardExec --> TxMgr
  TxMgr --> LogWriter
  ShardExec --> MemArena
  LogWriter --> SnapshotMgr
  LogWriter --> Replay
  LogWriter --> Projector
  Projector --> SQL
  Admin --> Gateway
  Admin --> Router
  Admin --> Projector
  Metrics --- Gateway
  Metrics --- ShardExec
  Metrics --- Projector
```
