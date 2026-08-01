# Licitech — Architecture Showcase

> **Idioma:** [English](README.md) · Português (Brasil)

Documentação de arquitetura, DevOps e segurança por trás do [Licitech](https://www.licitech.com.br/) — um SaaS em produção para busca e gestão de licitações públicas.

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
> **Este repositório não contém o código de produção.**
> O Licitech é um produto comercial. Aqui está a documentação sanitizada da arquitetura — decisões, diagramas C4, ADRs e templates de configuração — sem lógica de negócio, schemas reais, credenciais ou endpoints vivos.

---

## Sobre o produto

O [Licitech](https://www.licitech.com.br/) é uma plataforma para empresas que vendem ao governo. Na prática, ela cobre o ciclo operacional de licitações:

- **Busca** de editais por UF, modalidade e palavras-chave
- **Alertas** (WhatsApp e e-mail) quando algo bate com o perfil do fornecedor
- **Monitor de posição** e chat em pregões (Compras.gov.br)
- **Robô de lances** no ComprasNet e BLL
- **Gestão** do ciclo — orçamento, proposta e acompanhamento

Os dados vêm de fontes públicas de compras (incluindo PNCP / Compras.gov.br). O time usa a plataforma no dia a dia; este repositório documenta como o sistema foi construído por baixo.

Produto em produção: **[licitech.com.br](https://www.licitech.com.br/)**

### Buscador

Filtros por palavras-chave, UF e modalidade — editais agregados de fontes públicas num só lugar.

[![Buscador de licitações no Licitech](docs/imagens/tela_1.png)](https://www.licitech.com.br/)

### Robô de lances

Disputas ao vivo no Compras.gov.br: posição, melhor lance e estratégia automática sem ficar grudado no portal.

[![Robô de lances Compras.gov.br no Licitech](docs/imagens/tela_2.png)](https://www.licitech.com.br/)

---

## Por que este repositório existe

Não dá para abrir o código-fonte de um SaaS comercial. Mesmo assim, faz sentido mostrar como a engenharia foi pensada: fronteiras de serviço, segurança, CI/CD, escalabilidade e trade-offs reais.

O material abaixo descreve essa base — útil para quem avalia o trabalho técnico por trás do produto (incluindo processos acadêmicos).

---

## Destaques de engenharia

### Arquitetura

- Fronteiras claras entre UI na edge, API, workers e o plano de dados
- Aplicação **stateless** — estado durável no PostgreSQL / object storage
- Processamento assíncrono por **fila** (ingestão e jobs longos)
- Isolamento de falha por containers
- Documentado com **C4** e **ADRs** — veja [`docs/pt-br/`](docs/pt-br/)

### Segurança

- Defesa em camadas: edge → reverse proxy → app → RLS → secrets
- Auth com JWT, menor privilégio e Row Level Security
- Scan de dependências e imagens no pipeline; SBOM
- Threat model em **STRIDE** — [`docs/pt-br/THREAT_MODEL.md`](docs/pt-br/THREAT_MODEL.md)

### Infraestrutura

- **Docker** + **Dokploy** no backend
- **Vercel** para o frontend Next.js
- **Supabase / PostgreSQL** no plano de dados
- **Nginx** com TLS no host
- Templates em [`config-templates/`](config-templates/)

### DevOps

- **GitHub Actions**: lint → teste → security scan → build → deploy
- Imagens versionadas por SHA; promoção com health check
- Rollback explícito; secrets separados por ambiente

### Escalabilidade e confiabilidade

- Scale-out de API e workers; connection pooling
- Metas de **RPO / RTO** e runbooks de restore
- Um worker cair não deve derrubar a API
- Caminho documentado para OpenTelemetry e, se precisar, Kubernetes

---

## Diagramas

### Arquitetura geral

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

### Deploy

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

### Pipeline de CI/CD

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

### Fronteiras de segurança

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

### Ciclo de vida de uma request

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

## Decisões de engenharia

| Decisão | Por quê | Trade-off |
|---|---|---|
| **Docker em vez de Kubernetes** | Chegar em produção mais rápido; ops cabe no time | Menos HPA / mesh nativos — por enquanto |
| **Dokploy** | Deploy, health e rollback sem montar PaaS próprio | Acoplamento à plataforma; saída via imagem Docker |
| **Vercel no frontend** | CDN, preview deploys, pouco ops no Next.js | Jobs longos ficam no backend |
| **PostgreSQL** | Integridade, RLS, ecossistema maduro | Índices e pool precisam de cuidado |
| **Supabase** | Postgres + Auth + Storage com RLS | Vendor — abstraímos via repositories |
| **GitHub Actions** | Perto do código; actions de segurança maduras | Minutos de runner e concorrência |
| **Backend stateless** | Scale horizontal simples | Estado vai para DB / cache |
| **Fila para trabalho pesado** | Request path fica leve | Consistência eventual; idempotência |

Detalhes: [`docs/pt-br/ADR/`](docs/pt-br/ADR/) · [`docs/pt-br/TECHNOLOGY_CHOICES.md`](docs/pt-br/TECHNOLOGY_CHOICES.md)

---

## Navegação

| Caminho | Conteúdo |
|---|---|
| [`docs/pt-br/ARCHITECTURE.md`](docs/pt-br/ARCHITECTURE.md) | Design, fronteiras, fluxos, HA |
| [`docs/pt-br/DEVOPS_AND_CICD.md`](docs/pt-br/DEVOPS_AND_CICD.md) | Pipelines, versionamento, deploy, rollback |
| [`docs/pt-br/SECURITY_AND_COMPLIANCE.md`](docs/pt-br/SECURITY_AND_COMPLIANCE.md) | AuthZ, OWASP, supply chain, LGPD |
| [`docs/pt-br/OBSERVABILITY.md`](docs/pt-br/OBSERVABILITY.md) | Logs, métricas, tracing, incidentes |
| [`docs/pt-br/SCALABILITY.md`](docs/pt-br/SCALABILITY.md) | Scale horizontal, pools, cache, caminho K8s |
| [`docs/pt-br/PERFORMANCE.md`](docs/pt-br/PERFORMANCE.md) | Edge, CDN, DB, async |
| [`docs/pt-br/DISASTER_RECOVERY.md`](docs/pt-br/DISASTER_RECOVERY.md) | Backups, RPO/RTO |
| [`docs/pt-br/THREAT_MODEL.md`](docs/pt-br/THREAT_MODEL.md) | STRIDE |
| [`docs/pt-br/TECHNOLOGY_CHOICES.md`](docs/pt-br/TECHNOLOGY_CHOICES.md) | Por que cada tecnologia |
| [`docs/pt-br/ADR/`](docs/pt-br/ADR/) | Architecture Decision Records |
| [`docs/pt-br/C4/`](docs/pt-br/C4/) | Context, Container, Component |
| [`config-templates/`](config-templates/) | Compose, Nginx, Prometheus, env |
| [`ROADMAP-pt-br.md`](ROADMAP-pt-br.md) | Próximos passos |

---

## Roadmap

Curto prazo: OpenTelemetry, canary releases, Secrets Manager e Terraform. Depois: Kubernetes, blue/green e DR multi-região — se as métricas pedirem. Ver [`ROADMAP-pt-br.md`](ROADMAP-pt-br.md).

---

## Autor

**Alisson Oliveira** — Engenheiro de Software / Cientista de Dados

- GitHub: [@alissonf216](https://github.com/alissonf216)
- LinkedIn: [linkedin.com/in/alissonfranclin](https://www.linkedin.com/in/alissonfranclin/)
- Produto: [licitech.com.br](https://www.licitech.com.br/)

---

## Licença

[MIT License](LICENSE). Documentação e templates para fins educacionais e de portfólio.
