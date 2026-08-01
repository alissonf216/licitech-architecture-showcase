# Scalability

> Horizontal scale first. Kubernetes, mesh, and company only enter when operational metrics demand them.

---

## 1. Horizontal scale

```mermaid
flowchart TB
    subgraph Edge["Scales with the provider"]
        FE[Vercel Edge Network]
    end
    subgraph Compute["Scales by replica count"]
        API[API Replicas]
        W[Worker Replicas]
    end
    subgraph Data["Scales by plan / pattern"]
        PG[(PostgreSQL)]
        REDIS[(Redis)]
    end

    FE --> API
    API --> REDIS
    API --> PG
    W --> REDIS
    W --> PG
```

| Layer | Scale unit | Constraint |
|---|---|---|
| Frontend | Edge regions / ISR / CDN | Platform limits |
| API | Container replicas | DB pool budget |
| Workers | Container replicas | Queue throughput / upstream rate limits |
| Postgres | Vertical + read replicas (later) | Connection & IOPS |
| Redis | Vertical / clustered (later) | Memory & persistence mode |

---

## 2. Stateless services

API and workers **do not hold durable local state**. Scale-out does not require session affinity.

Implications:

- Any replica serves any request
- Rolling deploys stay simple
- Crash recovery = restart + reconsume

---

## 3. Worker pools

| Pool | Work type | Scale signal |
|---|---|---|
| Ingestion | Fetch & normalize | Queue depth / source windows |
| Enrichment | CPU / IO enrichment | Job age |
| Notification | Message fan-out | Queue depth |

Pools can be separate deploys sharing the same image with different command/env — isolates noisy workloads.

---

## 4. Autoscaling

**Today:** manual / scheduled replica adjustments via the orchestration UI.

**Target signals:**

```mermaid
flowchart LR
    QDEPTH[Queue depth] --> DEC{Scale decision}
    CPU[CPU saturation] --> DEC
    P95[API p95] --> DEC
    DEC -->|up| ADD[Add replicas]
    DEC -->|down| REM[Remove replicas]
```

Guards: cooldown periods, max replica caps (cost + DB pool), min replicas for HA.

---

## 5. Cache strategy

| Layer | What | TTL / invalidation |
|---|---|---|
| CDN / Edge | Static assets, cacheable pages | Platform + cache headers |
| Application | Hot read models | Short TTL + explicit bust on write |
| Redis | Rate limits, ephemeral computations | Key TTL |
| Database | Materialized views (selective) | Refresh jobs |

> [!WARNING]
> Cache only **tenant-safe** payloads. Never serve cross-tenant cached responses.

---

## 6. Connection pooling

- Cap the app-side pool so `replicas × pool ≤ DB max_connections` (with headroom)
- Prefer a pooler (e.g. managed pooler / PgBouncer-style) for many short queries
- Workers use pool budgets separate from the API

---

## 7. Queue processing at scale

- Partition work by entity key when order matters
- Batch when safe to reduce DB round-trips
- Apply source politeness limits independent of consumer concurrency
- Dead-letter monitoring prevents backlog rotting in silence

---

## 8. Database bottlenecks

| Symptom | Typical cause | Mitigation |
|---|---|---|
| Rising query p95 | Missing indexes / seq scans | Explain plans; index; rewrite |
| Pool wait | Too many replicas / large pools | Pooler; shrink pool; scale reads |
| Lock contention | Hot-row updates | Shard keys; serialize via queue |
| Storage growth | Unbounded history | Retention jobs; cold storage |

---

## 9. Future migration to Kubernetes

Kubernetes is a **possible** destination — not the default.

| Stay on Docker + Dokploy when… | Consider K8s when… |
|---|---|
| Replica count stays modest | Many services need declarative HPA |
| Single-region compute is enough | Multi-region scheduling is required |
| Ops team is small | Need standardized mesh / policy engines |
| Deploy pain is low | Deploy topology outgrows Compose comfort |

Exit criteria and trade-offs: [ADR 0001](ADR/0001-docker-over-kubernetes.md).

```mermaid
flowchart TD
    A[Measure pain] --> B{Ops complexity high?}
    B -->|No| C[Optimize current stack]
    B -->|Yes| D[ADR: K8s evaluation]
    D --> E[Pilot non-critical worker]
    E --> F{Net benefit?}
    F -->|Yes| G[Incremental migration]
    F -->|No| C
```

---

## Related documents

- [ARCHITECTURE.md](ARCHITECTURE.md)
- [PERFORMANCE.md](PERFORMANCE.md)
- [ROADMAP.md](../ROADMAP.md)
