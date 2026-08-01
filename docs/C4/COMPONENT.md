# C4 — Components

> **Language:** English · [Português (Brasil)](../pt-br/C4/COMPONENT.md)

> **Level 3.** Main blocks inside the API and Workers. Conceptual — not a dump of the source tree.

---

## Purpose

Explain the **separation of responsibilities** internally, without proprietary module names or business algorithms.

---

## API components

```mermaid
flowchart TB
    subgraph API["API Application"]
        HW[HTTP Interface]
        MW[Auth Middleware]
        VAL[Validation]
        APP[Application Services]
        REPO[Repositories]
        QCLIENT[Queue Publisher]
        OBS[Logging / Metrics Hooks]
    end

    HW --> MW --> VAL --> APP
    APP --> REPO
    APP --> QCLIENT
    HW --> OBS
    APP --> OBS

    REPO --> DB[(PostgreSQL)]
    QCLIENT --> REDIS[(Redis)]
    MW --> AUTH[Auth Provider]
```

| Component | Responsibility |
|---|---|
| HTTP Interface | Routing, status codes, serialization |
| Auth Middleware | JWT validation, principal extraction |
| Validation | Schema checks; reject malformed input early |
| Application Services | Use-case orchestration |
| Repositories | Persistence abstraction |
| Queue Publisher | Enqueue jobs with idempotency keys |
| Observability Hooks | Correlation IDs, RED metrics |

---

## Worker components

```mermaid
flowchart TB
    subgraph Worker["Worker Application"]
        CONS[Queue Consumer]
        DISP[Job Dispatcher]
        H1[Ingest Handler]
        H2[Enrich Handler]
        H3[Notify Handler]
        IDEM[Idempotency Guard]
        RETRY[Retry / DLQ Policy]
        REPO2[Repositories]
    end

    CONS --> DISP
    DISP --> IDEM
    IDEM --> H1 & H2 & H3
    H1 & H2 & H3 --> REPO2
    H1 & H2 & H3 --> RETRY
    REPO2 --> DB[(PostgreSQL)]
    CONS --> REDIS[(Redis)]
```

| Component | Responsibility |
|---|---|
| Queue Consumer | Lease jobs; heartbeat; ACK/NACK |
| Job Dispatcher | Routing by job type |
| Handlers | Near-pure use cases for each job family |
| Idempotency Guard | Skip duplicate side effects |
| Retry / DLQ Policy | Backoff; dead-letter; alert |
| Repositories | Same persistence boundary as the API |

---

## Frontend components (simplified)

```mermaid
flowchart LR
    subgraph Web["Web Application"]
        PAGES[Pages / Layouts]
        FEAT[Feature Modules]
        API_CLIENT[API Client]
        AUTH_UI[Auth Session Bridge]
    end

    PAGES --> FEAT
    FEAT --> API_CLIENT
    AUTH_UI --> API_CLIENT
    API_CLIENT -->|HTTPS| API[API Application]
```

---

## Cross-component design rules

1. **No SQL in HTTP handlers** — repositories only
2. **No HTTP in workers** for core domain writes — handlers call repositories/services
3. **Idempotency** is mandatory on every job handler
4. **AuthZ checks** happen in services *and* in RLS
5. **Secrets** never travel in job payloads when avoidable — fetch at runtime

---

## Related

- [CONTAINER.md](CONTAINER.md)
- [ARCHITECTURE.md](../ARCHITECTURE.md)
- [SECURITY_AND_COMPLIANCE.md](../SECURITY_AND_COMPLIANCE.md)
