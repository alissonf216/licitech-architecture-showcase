# C4 — Containers

> **Idioma:** [English](../../C4/CONTAINER.md) · Português (Brasil)

> **Nível 2.** Unidades que sobem e rodam de forma independente, e como elas se falam. Só nomes sanitizados.

---

## Para que serve

Descrever os principais **containers** no sentido C4 (não só Docker): aplicações e stores que dá para deployar e raciocinar separados.

---

## Diagrama de containers

```mermaid
flowchart TB
    USER["Web User"]

    subgraph Edge["Edge Plane"]
        WEB["Web Application<br/>Next.js on Vercel"]
    end

    subgraph Host["Compute Host"]
        PROXY["Reverse Proxy<br/>Nginx"]
        API["API Application<br/>Python"]
        WORKER["Worker Application<br/>Python"]
        QUEUE["Queue / Cache<br/>Redis"]
    end

    subgraph DataPlane["Managed Data Plane"]
        DB["Database<br/>PostgreSQL via Supabase"]
        AUTH["Auth Service<br/>Supabase Auth"]
        STORE["Object Storage<br/>Supabase Storage"]
    end

    USER -->|HTTPS| WEB
    WEB -->|HTTPS + JWT| PROXY
    PROXY --> API
    API --> QUEUE
    QUEUE --> WORKER
    API --> DB
    API --> AUTH
    WORKER --> DB
    WORKER --> STORE
    WORKER -->|HTTPS| EXT["Public Sources"]
```

---

## Catálogo de containers

| Container | Tecnologia | Responsabilidades |
|---|---|---|
| Web Application | Next.js / TypeScript | UI, edge middleware, chama a API |
| Reverse Proxy | Nginx | Término de TLS, roteamento, headers |
| API Application | Python | Commands/queries síncronos, authZ, enqueue |
| Worker Application | Python | Jobs assíncronos: ingest, enrich, notify |
| Queue / Cache | Redis | Broker de jobs, cache efêmero, contadores de rate |
| Database | PostgreSQL | System of record + RLS |
| Auth Service | Supabase Auth | Identidade, tokens |
| Object Storage | Supabase Storage | Blobs / artefatos |

---

## Resumo de comunicação

| De | Para | Protocolo | Auth |
|---|---|---|---|
| Web → Proxy/API | HTTPS | JWT / session |
| API → Redis | TCP interno | Isolamento de rede |
| API → Postgres | TLS | Credenciais de DB + contexto RLS |
| Worker → Sources | HTTPS | N/A (público) + allowlists |
| Worker → Storage | HTTPS | Credenciais de serviço |

---

## Mapeamento de deploy

```mermaid
flowchart LR
    subgraph Vercel
        WEB2[Web Application]
    end
    subgraph Dokploy Host
        PROXY2[Nginx]
        API2[API]
        W2[Workers]
        R2[Redis]
    end
    subgraph Supabase
        DB2[(Postgres)]
        A2[Auth]
        S2[Storage]
    end
    WEB2 --> PROXY2
    PROXY2 --> API2
    API2 --> R2
    R2 --> W2
    API2 --> DB2
    API2 --> A2
    W2 --> DB2
    W2 --> S2
```

---

## Relacionados

- [CONTEXT.md](CONTEXT.md)
- [COMPONENT.md](COMPONENT.md)
- [ADR 0001](../ADR/0001-docker-over-kubernetes.md)
- [ADR 0002](../ADR/0002-vercel-edge.md)
