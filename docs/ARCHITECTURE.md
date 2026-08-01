# Architecture

> **Language:** English · [Português (Brasil)](pt-br/ARCHITECTURE.md)

> Sanitized architecture documentation: no proprietary schemas, real endpoints, or implementation details.

---

## 1. System context

Licitech is a multi-tenant SaaS that aggregates and organizes **public procurement signals** (notices, announcements, legislative references) and surfaces them through authenticated web flows.

```mermaid
C4Context
    title System Context — Licitech

    Person(user, "Procurement Analyst", "Tracks notices and workflows")
    Person(admin, "Tenant Admin", "Manages users and preferences")

    System(licitech, "Licitech Platform", "Public procurement intelligence SaaS")

    System_Ext(sources, "Public Data Sources", "Government portals and open-data APIs")
    System_Ext(email, "Notification Provider", "Transactional email / webhooks")
    System_Ext(idp, "Identity (Supabase Auth)", "Authentication provider")

    Rel(user, licitech, "Uses", "HTTPS")
    Rel(admin, licitech, "Administers", "HTTPS")
    Rel(licitech, sources, "Ingests (rate-limited)", "HTTPS")
    Rel(licitech, email, "Sends notifications", "HTTPS")
    Rel(licitech, idp, "Validates sessions", "HTTPS")
```

> [!NOTE]
> External source adapters are treated as **untrusted I/O**. All ingested content is validated, normalized, and stored under per-tenant policies before it reaches the user.

---

## 2. Service boundaries

| Boundary | Responsibility | Failure isolation |
|---|---|---|
| **Edge UI** | Rendering, client routing, edge auth gates | Degraded UI without stalling workers |
| **API** | Synchronous commands/queries, authZ | Restarts independently of workers |
| **Workers** | Ingestion, enrichment, notifications | Crash loops do not take down the API |
| **Queue** | Buffer and retry semantics | Back-pressure protects upstream sources |
| **Data plane** | Durable state, RLS, storage | PITR / backups independent of compute |

```mermaid
graph TB
    subgraph Edge["Edge Boundary"]
        FE[Frontend]
    end
    subgraph App["Application Boundary"]
        API[API]
        W[Workers]
        Q[Queue]
    end
    subgraph Data["Data Boundary"]
        DB[(PostgreSQL)]
        OBJ[Object Storage]
    end

    FE <-->|HTTPS / JWT| API
    API <--> Q
    Q --> W
    API <--> DB
    W <--> DB
    W <--> OBJ
```

---

## 3. Backend components

| Component | Role | State |
|---|---|---|
| **API process** | REST/JSON (or RPC) surface; validation; orchestration | Stateless |
| **Auth middleware** | JWT verification, claim extraction, tenant binding | Stateless |
| **Domain services** | Use-case orchestration (no direct SQL in controllers) | Stateless |
| **Repository layer** | Persistence abstraction | Stateless |
| **Worker runners** | Consume jobs; idempotent handlers | Stateless |
| **Scheduler / poller** | Enqueues periodic ingestion windows | Stateless |

> [!TIP]
> Controllers stay thin. Domain rules live in services. Persistence is swappable behind repositories — so Supabase/Postgres remains a data-plane implementation detail.

---

## 4. Worker architecture

Workers are **horizontally scalable processes** consuming from a shared queue.

```mermaid
flowchart TD
    API[API / Scheduler] -->|enqueue| Q[(Redis Queue)]
    Q --> WA[Worker A]
    Q --> WB[Worker B]
    Q --> WC[Worker N]

    WA --> H1[Handler: Ingest]
    WB --> H2[Handler: Enrich]
    WC --> H3[Handler: Notify]

    H1 --> DB[(PostgreSQL)]
    H2 --> DB
    H3 --> EXT[Notification Provider]

    H1 -->|retry / DLQ| Q
    H2 -->|retry / DLQ| Q
```

**Design rules**

1. **Idempotency keys** on every job — we assume at-least-once delivery
2. **Bounded retries** with exponential backoff + jitter
3. **Dead-letter** path for poison messages after max attempts
4. **Visibility timeout** so crashed workers do not leave orphaned jobs
5. **No shared mutable memory** across workers

---

## 5. Data flow

```mermaid
flowchart LR
    SRC[Public Sources] -->|pull| ING[Ingestion Worker]
    ING -->|normalize / validate| NORM[Normalized Records]
    NORM -->|upsert| DB[(PostgreSQL)]
    DB -->|read| API[API]
    API -->|JWT + RLS| TENANT[Per-Tenant Results]
    TENANT --> UI[Frontend]
```

**Principles**

- The write path is append/upsert-oriented; updates are auditable when needed
- The read path is filtered by RLS in the database — application filters are the second line, not the only one
- Large payloads (attachments, raw snapshots) go to object storage; the DB holds references and metadata

---

## 6. Event flow

```mermaid
sequenceDiagram
    participant S as Scheduler
    participant Q as Queue
    participant W as Worker
    participant DB as Database
    participant N as Notifier

    S->>Q: enqueue(ingest_window)
    Q->>W: deliver(job)
    W->>W: fetch + validate
    W->>DB: upsert records
    alt User matching rules hit
        W->>Q: enqueue(notify)
        Q->>N: deliver(notify_job)
        N->>N: send channel message
    end
    W-->>Q: ACK
```

Events are **commands on a queue**, not a full event bus — chosen for operational simplicity at the current scale. See the [ADR roadmap](../ROADMAP.md) for when a richer event backbone would be worth reconsidering.

---

## 7. Queue processing

| Concern | Approach |
|---|---|
| Delivery | At-least-once |
| Ordering | Per partition / per entity key when required; otherwise best-effort |
| Back-pressure | Queue-depth metrics → scale workers / drop non-critical jobs |
| Poison messages | Max retries → dead-letter + alert |
| Observability | Job ID correlation in logs: API → queue → worker |

---

## 8. Stateless design

All application processes are **stateless**:

- No in-process session store
- No sticky sessions at the load balancer
- Configuration via environment (injected at runtime)
- Horizontal scale = replica count

Durable state:

| Store | Contents |
|---|---|
| PostgreSQL | Canonical entities, tenancy, authz attributes |
| Redis | Queue / short-lived cache / rate-limit counters |
| Object storage | Blobs and raw artifacts |

---

## 9. Failure isolation

```mermaid
graph TD
    subgraph FD1["Failure Domain: Edge"]
        V[Vercel Deploy]
    end
    subgraph FD2["Failure Domain: API"]
        A1[API Replica 1]
        A2[API Replica 2]
    end
    subgraph FD3["Failure Domain: Workers"]
        W1[Worker Pool]
    end
    subgraph FD4["Failure Domain: Data"]
        PG[(PostgreSQL)]
        R[(Redis)]
    end

    V -.->|independent| FD2
    FD2 -.->|independent| FD3
    FD2 --> FD4
    FD3 --> FD4
```

| Failure | Blast radius | Mitigation |
|---|---|---|
| Single API container crash | In-flight requests on that replica | Restart policy + multi-replica |
| Worker pool saturation | Delayed async jobs | Scale out; drop low-priority work |
| Redis unavailable | Queue pauses | Alerts; degraded mode (sync-critical paths only, if defined) |
| Postgres unavailable | Full read/write outage | Fail closed; backup restore / PITR |

---

## 10. Rate limiting

| Layer | Mechanism | Purpose |
|---|---|---|
| Edge | Platform rules / WAF | Volumetric abuse |
| API | Per-token / IP / tenant budgets | Fair use and cost control |
| Workers | Per-source politeness limits | Protect upstream public APIs |
| Database | Pool caps | Avoid connection storms |

When a limit is exceeded, we return explicit, retry-friendly errors (e.g. `429` with `Retry-After` semantics when applicable).

---

## 11. Retry strategy

```mermaid
flowchart TD
    START[Attempt N] --> OK{Success?}
    OK -->|Yes| DONE[ACK / Commit]
    OK -->|No| RETRYABLE{Retryable error?}
    RETRYABLE -->|No| FAIL[Fail + alert / DLQ]
    RETRYABLE -->|Yes| MAX{N < max?}
    MAX -->|Yes| BACKOFF[Exponential backoff + jitter]
    BACKOFF --> START
    MAX -->|No| DLQ[Dead-letter + page]
```

- Distinguish **transient** (timeouts, 503) from **permanent** (validation, 404)
- Cap total attempts; never infinite retry on the hot path
- Idempotency tokens prevent duplicated side effects

---

## 12. High availability

| Layer | HA posture |
|---|---|
| Frontend | Multi-AZ edge network (Vercel) |
| API | N ≥ 2 replicas behind reverse proxy |
| Workers | N ≥ 2; the queue absorbs short outages |
| PostgreSQL | Managed HA / automatic failover (provider) |
| Redis | Persistence mode matched to queue durability needs |

Health endpoints (`/health/live`, `/health/ready`) control traffic and orchestrator restarts.

---

## 13. Horizontal scale

```mermaid
flowchart LR
    LB[Nginx / LB] --> A1[API]
    LB --> A2[API]
    LB --> A3[API]
    Q[(Queue)] --> W1[Worker]
    Q --> W2[Worker]
    Q --> W3[Worker]
    A1 & A2 & A3 --> PG[(PostgreSQL pooled)]
    W1 & W2 & W3 --> PG
```

Scale triggers (manual today; automated on the roadmap):

- API: CPU / p95 latency / RPS
- Workers: queue depth / lag / job age

---

## 14. Failure domains — summary

| Domain | Contains | Independent restart? |
|---|---|---|
| Edge | Next.js deploy | Yes |
| Proxy | Nginx | Yes |
| API | Application containers | Yes |
| Workers | Job consumers | Yes |
| Cache/Queue | Redis | Yes (with queue-drain risk) |
| Data | PostgreSQL + storage | Restore-oriented |

---

## Related documents

- [C4 Context](C4/CONTEXT.md) · [Container](C4/CONTAINER.md) · [Component](C4/COMPONENT.md)
- [SCALABILITY.md](SCALABILITY.md) · [PERFORMANCE.md](PERFORMANCE.md)
- [THREAT_MODEL.md](THREAT_MODEL.md) · [ADR/](ADR/)
