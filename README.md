# Licitech — Architecture Showcase

> **Language:** English · [Português (Brasil)](README-pt-br.md)
>
> The English version of this repository was translated with the assistance of **Gemini 3.6 Flash**.

Architecture, DevOps, and security documentation behind [Licitech](https://www.licitech.com.br/) — a production SaaS for searching and managing public procurement (Brazilian government tenders).

[![Python](https://img.shields.io/badge/Python-3.11+-3776AB?style=for-the-badge&logo=python&logoColor=white)](https://www.python.org/)
[![TypeScript](https://img.shields.io/badge/TypeScript-5.x-3178C6?style=for-the-badge&logo=typescript&logoColor=white)](https://www.typescriptlang.org/)
[![Docker](https://img.shields.io/badge/Docker-Containers-2496ED?style=for-the-badge&logo=docker&logoColor=white)](https://www.docker.com/)
[![PostgreSQL](https://img.shields.io/badge/PostgreSQL-15+-4169E1?style=for-the-badge&logo=postgresql&logoColor=white)](https://www.postgresql.org/)
[![Supabase](https://img.shields.io/badge/Supabase-BaaS-3FCF8E?style=for-the-badge&logo=supabase&logoColor=white)](https://supabase.com/)
[![Vercel](https://img.shields.io/badge/Vercel-Edge-000000?style=for-the-badge&logo=vercel&logoColor=white)](https://vercel.com/)
[![Dokploy](https://img.shields.io/badge/Dokploy-Deploy-0EA5E9?style=for-the-badge)](https://dokploy.com/)
[![GitHub Actions](https://img.shields.io/badge/GitHub_Actions-CI%2FCD-2088FF?style=for-the-badge&logo=githubactions&logoColor=white)](https://github.com/features/actions)
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow?style=for-the-badge)](LICENSE)

---

> [!IMPORTANT]
> **This repository does not contain production source code.**
> Licitech is a commercial product. What you will find here is sanitized architecture documentation — decisions, C4 diagrams, ADRs, and configuration templates — without proprietary business logic, real schemas, credentials, or live endpoints.

---

## About the product

[Licitech](https://www.licitech.com.br/) is a platform for companies that sell to the government. In practice, it covers the operational cycle of public tenders:

- **Search** for notices by state (UF), modality, and keywords
- **Alerts** (WhatsApp and email) when something matches the supplier profile
- **Position monitoring** and chat during reverse auctions (Compras.gov.br)
- **Bidding robot** on ComprasNet and BLL
- **Lifecycle management** — quotes, proposals, and follow-up

Data comes from public procurement sources (including PNCP / Compras.gov.br). The team uses the platform daily; this repository documents how the system is built underneath.

Live product: **[licitech.com.br](https://www.licitech.com.br/)**

### Search

Filters by keywords, state (UF), and modality — public tenders aggregated in one place.

[![Licitech tender search](docs/imagens/tela_1.png)](https://www.licitech.com.br/)

### Bidding robot

Live disputes on Compras.gov.br: position, best bid, and automated strategy without staying glued to the portal.

[![Licitech Compras.gov.br bidding robot](docs/imagens/tela_2.png)](https://www.licitech.com.br/)

---

## Why this repository exists

You cannot open-source a commercial SaaS. It still makes sense to show how the engineering was designed: service boundaries, security, CI/CD, scalability, and real trade-offs.

The material below describes that foundation — useful for anyone evaluating the technical work behind the product (including academic review processes).

---

## Engineering highlights

### Architecture

- Clear boundaries between edge UI, API, workers, and the data plane
- **Stateless** application tier — durable state in PostgreSQL / object storage
- Async processing via **queue** (ingestion and long-running jobs)
- Fault isolation through containers
- Documented with **C4** and **ADRs** — see [`docs/`](docs/)

### Security

- Layered defense: edge → reverse proxy → app → RLS → secrets
- JWT auth, least privilege, and Row Level Security
- Dependency and image scanning in the pipeline; SBOM
- **STRIDE** threat model — [`docs/THREAT_MODEL.md`](docs/THREAT_MODEL.md)

### Infrastructure

- **Docker** + **Dokploy** on the backend
- **Vercel** for the Next.js frontend
- **Supabase / PostgreSQL** on the data plane
- **Nginx** with TLS on the host
- Templates in [`config-templates/`](config-templates/)

### DevOps

- **GitHub Actions**: lint → test → security scan → build → deploy
- SHA-versioned images; health-gated promotion
- Explicit rollback; secrets separated by environment

### Scalability and reliability

- Horizontal scale-out of API and workers; connection pooling
- **RPO / RTO** targets and restore runbooks
- A worker crash should not take down the API
- Documented path toward OpenTelemetry and, if needed, Kubernetes

---

## Diagrams

### Overall architecture

```mermaid
graph TD
    subgraph Clients["Clients"]
        U[Browser / SPA]
        M[Mobile Web]
    end

    subgraph Edge["Edge Plane — Vercel"]
        FE[Next.js Frontend]
        CDN[CDN / Static Assets]
        ER[Edge Middleware]
    end

    subgraph Compute["Compute Host — Dokploy + Docker"]
        NX[Nginx Reverse Proxy]
        API[API Service]
        W1[Worker Pool A]
        W2[Worker Pool B]
        Q[(Redis Queue)]
    end

    subgraph Data["Data Plane — Supabase"]
        PG[(PostgreSQL)]
        AUTH[Auth Service]
        STOR[Object Storage]
    end

    subgraph Obs["Observability"]
        LOG[Structured Logs]
        MET[Metrics]
        ALRT[Alerting]
    end

    U --> CDN
    M --> CDN
    CDN --> FE
    FE --> ER
    ER -->|HTTPS API calls| NX
    NX --> API
    API --> Q
    Q --> W1
    Q --> W2
    API --> PG
    API --> AUTH
    W1 --> PG
    W2 --> STOR
    API --> LOG
    W1 --> MET
    API --> ALRT
```

### Deployment

```mermaid
flowchart LR
    subgraph Dev["Developer Workstation"]
        CODE[Source Control]
    end

    subgraph CI["GitHub Actions"]
        LINT[Lint & Typecheck]
        TEST[Unit / Integration Tests]
        SEC[SAST + Dependency Scan]
        BUILD[Docker Build]
        ISCAN[Image Scan]
        ART[Artifact Registry]
    end

    subgraph Prod["Production"]
        VER[Vercel — Frontend]
        DOK[Dokploy — Backend Host]
        SUP[Supabase — Data]
    end

    CODE --> LINT --> TEST --> SEC --> BUILD --> ISCAN --> ART
    ART -->|Promote image| DOK
    CODE -->|Frontend deploy| VER
    DOK --> SUP
    VER -->|API HTTPS| DOK
```

### CI/CD pipeline

```mermaid
flowchart TD
    A[Push / PR] --> B[Checkout]
    B --> C[Install Dependencies]
    C --> D[Lint]
    C --> E[Unit Tests]
    C --> F[Security Scan]
    C --> G[Dependency Scan]
    D --> H{Gates Pass?}
    E --> H
    F --> H
    G --> H
    H -->|No| X[Fail Pipeline]
    H -->|Yes| I[Docker Build]
    I --> J[Image Scan]
    J --> K[Push Artifact]
    K --> L[Deploy — Staging]
    L --> M[Health Check]
    M -->|Healthy| N[Promote — Production]
    M -->|Unhealthy| R[Rollback]
    N --> O[Post-deploy Probe]
    O -->|Fail| R
```

### Security boundaries

```mermaid
flowchart LR
    subgraph Public["Untrusted — Internet"]
        ATT[Any Client]
    end

    subgraph EdgeTrust["Trust Boundary — Edge"]
        WAF[Edge Protections]
        FE2[Frontend Origin]
    end

    subgraph AppTrust["Trust Boundary — Application"]
        TLS[TLS Termination]
        API2[Authenticated API]
        JWT[JWT Validation]
    end

    subgraph DataTrust["Trust Boundary — Data"]
        RLS[Row Level Security]
        DB[(PostgreSQL)]
        SECRETS[Secrets Store]
    end

    ATT --> WAF --> FE2
    FE2 -->|mTLS / HTTPS| TLS --> API2 --> JWT
    JWT --> RLS --> DB
    API2 -.->|runtime inject| SECRETS
```

### Request lifecycle

```mermaid
sequenceDiagram
    autonumber
    actor User
    participant Edge as Vercel Edge
    participant Proxy as Nginx
    participant API as API Service
    participant Auth as Auth / JWT
    participant DB as PostgreSQL
    participant Q as Queue
    participant W as Worker

    User->>Edge: HTTPS request
    Edge->>Edge: Auth session / middleware
    Edge->>Proxy: API call (TLS)
    Proxy->>API: Forward (internal network)
    API->>Auth: Validate JWT / claims
    Auth-->>API: Principal + roles
    API->>DB: Authorized query (RLS)
    DB-->>API: Result set
    alt Async work required
        API->>Q: Enqueue job
        Q->>W: Deliver job
        W->>DB: Persist side effects
        W-->>Q: ACK / retry
    end
    API-->>Proxy: JSON response
    Proxy-->>Edge: HTTPS response
    Edge-->>User: Rendered UI / data
```

---

## Engineering decisions

| Decision | Why | Trade-off |
|---|---|---|
| **Docker instead of Kubernetes** | Faster path to production; ops fits the team | Less native HPA / mesh — for now |
| **Dokploy** | Deploy, health, and rollback without building a custom PaaS | Platform coupling; exit via Docker images |
| **Vercel for the frontend** | CDN, preview deploys, low ops for Next.js | Long-running jobs stay on the backend |
| **PostgreSQL** | Integrity, RLS, mature ecosystem | Indexes and pooling need care |
| **Supabase** | Managed Postgres + Auth + Storage with RLS | Vendor boundary — abstracted via repositories |
| **GitHub Actions** | Close to the code; mature security actions | Runner minutes and concurrency |
| **Stateless backend** | Simple horizontal scale | State lives in DB / cache |
| **Queue for heavy work** | Keeps the request path light | Eventual consistency; idempotency required |

Details: [`docs/ADR/`](docs/ADR/) · [`docs/TECHNOLOGY_CHOICES.md`](docs/TECHNOLOGY_CHOICES.md)

---

## Repository navigation

| Path | Contents |
|---|---|
| [`docs/ARCHITECTURE.md`](docs/ARCHITECTURE.md) | Design, boundaries, flows, HA |
| [`docs/DEVOPS_AND_CICD.md`](docs/DEVOPS_AND_CICD.md) | Pipelines, versioning, deploy, rollback |
| [`docs/SECURITY_AND_COMPLIANCE.md`](docs/SECURITY_AND_COMPLIANCE.md) | AuthZ, OWASP, supply chain, LGPD |
| [`docs/OBSERVABILITY.md`](docs/OBSERVABILITY.md) | Logs, metrics, tracing, incidents |
| [`docs/SCALABILITY.md`](docs/SCALABILITY.md) | Horizontal scale, pools, cache, K8s path |
| [`docs/PERFORMANCE.md`](docs/PERFORMANCE.md) | Edge, CDN, DB, async |
| [`docs/DISASTER_RECOVERY.md`](docs/DISASTER_RECOVERY.md) | Backups, RPO/RTO |
| [`docs/THREAT_MODEL.md`](docs/THREAT_MODEL.md) | STRIDE |
| [`docs/TECHNOLOGY_CHOICES.md`](docs/TECHNOLOGY_CHOICES.md) | Why each technology |
| [`docs/ADR/`](docs/ADR/) | Architecture Decision Records |
| [`docs/C4/`](docs/C4/) | Context, Container, Component |
| [`config-templates/`](config-templates/) | Compose, Nginx, Prometheus, env |
| [`ROADMAP.md`](ROADMAP.md) | Next steps |

---

## Roadmap

Near term: OpenTelemetry, canary releases, Secrets Manager, and Terraform. Later: Kubernetes, blue/green, and multi-region DR — if the metrics ask for it. See [`ROADMAP.md`](ROADMAP.md).

---

## Author

**Alisson Oliveira** — Software Engineer / Data Scientist

- GitHub: [@alissonf216](https://github.com/alissonf216)
- LinkedIn: [linkedin.com/in/alissonfranclin](https://www.linkedin.com/in/alissonfranclin/)
- Product: [licitech.com.br](https://www.licitech.com.br/)

---

## License

[MIT License](LICENSE). Documentation and templates for educational and portfolio purposes.
