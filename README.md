# Licitera — Architecture Showcase

Documentação de arquitetura, DevOps e segurança do Licitera — um SaaS de inteligência em compras públicas.

[![Python](https://img.shields.io/badge/Python-3.11+-3776AB?style=for-the-badge&logo=python&logoColor=white)](https://www.python.org/)
[![TypeScript](https://img.shields.io/badge/TypeScript-5.x-3178C6?style=for-the-badge&logo=typescript&logoColor=white)](https://www.typescriptlang.org/)
[![Docker](https://img.shields.io/badge/Docker-Containers-2496ED?style=for-the-badge&logo=docker&logoColor=white)](https://www.docker.com/)
[![PostgreSQL](https://img.shields.io/badge/PostgreSQL-15+-4169E1?style=for-the-badge&logo=postgresql&logoColor=white)](https://www.postgresql.org/)
[![Supabase](https://img.shields.io/badge/Supabase-BaaS-3FCF8E?style=for-the-badge&logo=supabase&logoColor=white)](https://supabase.com/)
[![Vercel](https://img.shields.io/badge/Vercel-Edge-000000?style=for-the-badge&logo=vercel&logoColor=white)](https://vercel.com/)
[![Dokploy](https://img.shields.io/badge/Dokploy-Deploy-0EA5E9?style=for-the-badge)](https://dokploy.com/)
[![GitHub Actions](https://img.shields.io/badge/GitHub_Actions-CI%2FCD-2088FF?style=for-the-badge&logo=githubactions&logoColor=white)](https://github.com/features/actions)
[![Security](https://img.shields.io/badge/Security-Defense_in_Depth-DC2626?style=for-the-badge&logo=shield&logoColor=white)](docs/SECURITY_AND_COMPLIANCE.md)
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow?style=for-the-badge)](LICENSE)
[![Mermaid](https://img.shields.io/badge/Diagrams-Mermaid-FF3670?style=for-the-badge&logo=mermaid&logoColor=white)](https://mermaid.js.org/)
[![Architecture](https://img.shields.io/badge/Architecture-C4_%2B_ADR-6366F1?style=for-the-badge)](docs/ARCHITECTURE.md)
[![DevOps](https://img.shields.io/badge/DevOps-SRE_Practices-059669?style=for-the-badge)](docs/DEVOPS_AND_CICD.md)

---

> [!IMPORTANT]
> **Este repositório é um portfólio de arquitetura — não o código de produção.**
> Aqui estão documentação sanitizada, registros de decisão (ADRs), modelos C4 e *templates* de configuração.
> Não há lógica de negócio proprietária, credenciais, schemas reais nem endpoints ao vivo.

---

## O que é o Licitera

O **Licitera** é um SaaS em produção para **inteligência em compras públicas**: monitoramento de editais, processamento de dados legislativos e de avisos, e automação de fluxos para times que precisam de sinais estruturados e em tempo hábil a partir de fontes públicas.

Este repositório mostra o **sistema de engenharia por trás** — como os serviços se separam, como os deploys ficam seguros, como a segurança é empilhada em camadas, e quais trade-offs fizemos em cada fase.

| Quem lê | O que encontra |
|---|---|
| Gestores / entrevistas Staff+ | Qualidade de decisão, fluência em trade-offs, pensamento operacional |
| Revisores de segurança | Modelo STRIDE, defesa em profundidade, controles de supply chain |
| Pares de Platform / SRE | CI/CD, postura de HA, metas de DR, estratégia de observabilidade |
| Arquitetos | Visões C4, ADRs, escalabilidade e domínios de falha |

---

## Destaques de engenharia

### Arquitetura

- Fronteiras claras entre UI na edge, API, workers e o plano de dados
- Camada de aplicação **stateless** — o estado durável fica no PostgreSQL / object storage
- Processamento assíncrono baseado em **fila** para ingestão e jobs longos
- Isolamento de falha por containers e reinícios independentes
- Documentado com **C4** e **ADRs** — veja [`docs/`](docs/)

### Segurança

- **Defesa em profundidade**: edge → reverse proxy → app → RLS → isolamento de secrets
- Auth com JWT, autorização de menor privilégio e Row Level Security
- Controles de supply chain: scan de dependências, scan de imagem, pipeline orientado a SBOM
- Containers endurecidos, segmentação de rede, mitigações do OWASP Top 10
- Threat model completo em **STRIDE** — [`docs/THREAT_MODEL.md`](docs/THREAT_MODEL.md)

### Infraestrutura

- **Docker** como unidade de deploy; **Dokploy** para orquestração no host
- **Vercel Edge** para o frontend Next.js (CDN + edge runtime)
- **Supabase / PostgreSQL** como plano de dados gerenciado
- **Nginx** como reverse proxy com terminação TLS no host de compute
- Templates sanitizados em [`config-templates/`](config-templates/)

### DevOps

- **GitHub Actions** com gates de lint → teste → security scan → build → deploy
- Versionamento imutável de imagens e retenção de artefatos
- Promoção com health check e caminho explícito de rollback
- Promoção entre ambientes sem compartilhar secrets entre estágios

### Escalabilidade

- Scale-out horizontal de réplicas de API e workers
- Connection pooling e pools de workers sensíveis à profundidade da fila
- Cache na edge e na aplicação
- Caminho documentado rumo ao Kubernetes quando a pressão operacional justificar

### Confiabilidade

- Health checks, restart policies e rolling updates
- Metas de **RPO / RTO** e runbooks de backup/restore
- Domínios de falha pensados para que um worker cair não derrube a API
- Base de observabilidade com caminho claro para OpenTelemetry

---

## Diagramas Mermaid

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

| Decisão | Por quê | Trade-off principal |
|---|---|---|
| **Docker em vez de Kubernetes** | Caminho mais rápido até produção; carga operacional cabe no time | Menos HPA / service mesh nativos — até precisar |
| **Dokploy em vez de orquestração caseira** | Deploys declarativos, health e rollback sem montar um PaaS próprio | Acoplamento à plataforma; saída via imagens Docker padrão |
| **Vercel Edge no frontend** | CDN global, preview deploys, pouco ops no Next.js | Backend fica em compute dedicado para jobs longos |
| **PostgreSQL** | Integridade relacional, RLS, ecossistema maduro | Exige cuidado com índices e connection pool |
| **Supabase** | Postgres + Auth + Storage gerenciados, com RLS | Fronteira de vendor — abstraímos via repositories |
| **GitHub Actions** | Nativo ao controle de versão; ecossistema de actions de segurança | Planejar minutos de runner e concorrência |
| **Rede Docker** | Redes bridge explícitas; alcance mínimo entre serviços | Políticas de rede precisam ser documentadas e testadas |
| **Backend stateless** | Scale horizontal simples; recuperação limpa | Sessão/estado vão para DB / cache |
| **Processamento por fila** | Isola trabalho lento/IO do caminho da request | Consistência eventual; idempotência obrigatória |

Detalhes: [`docs/ADR/`](docs/ADR/) · [`docs/TECHNOLOGY_CHOICES.md`](docs/TECHNOLOGY_CHOICES.md)

---

## Navegação do repositório

| Caminho | O que tem |
|---|---|
| [`docs/ARCHITECTURE.md`](docs/ARCHITECTURE.md) | Design do sistema, fronteiras, fluxos, HA |
| [`docs/DEVOPS_AND_CICD.md`](docs/DEVOPS_AND_CICD.md) | Pipelines, versionamento, deploy, rollback |
| [`docs/SECURITY_AND_COMPLIANCE.md`](docs/SECURITY_AND_COMPLIANCE.md) | AuthZ, OWASP, supply chain, LGPD |
| [`docs/OBSERVABILITY.md`](docs/OBSERVABILITY.md) | Logs, métricas, tracing, resposta a incidentes |
| [`docs/SCALABILITY.md`](docs/SCALABILITY.md) | Scale horizontal, pools, cache, caminho K8s |
| [`docs/PERFORMANCE.md`](docs/PERFORMANCE.md) | Edge, CDN, DB, otimização assíncrona |
| [`docs/DISASTER_RECOVERY.md`](docs/DISASTER_RECOVERY.md) | Backups, RPO/RTO, continuidade |
| [`docs/THREAT_MODEL.md`](docs/THREAT_MODEL.md) | STRIDE: assets, atores, mitigações |
| [`docs/TECHNOLOGY_CHOICES.md`](docs/TECHNOLOGY_CHOICES.md) | Por que cada tecnologia foi escolhida |
| [`docs/ADR/`](docs/ADR/) | Architecture Decision Records |
| [`docs/C4/`](docs/C4/) | Visões Context, Container e Component |
| [`config-templates/`](config-templates/) | Compose, Nginx, Prometheus e env sanitizados |
| [`ROADMAP.md`](ROADMAP.md) | Evolução de curto e longo prazo |

---

## Roadmap

No curto prazo: **OpenTelemetry**, **canary releases**, **Secrets Manager** e **Terraform**. Mais à frente, avaliamos **Kubernetes**, **blue/green** e **DR multi-região** quando as métricas justificarem o custo operacional. Detalhe em [`ROADMAP.md`](ROADMAP.md).

---

## Autor

**Alisson Oliveira** — engenheiro de software construindo sistemas de inteligência em compras públicas em produção.

- GitHub: [@alissonf216](https://github.com/alissonf216)
- LinkedIn: *[adicionar URL do perfil]*

---

## Licença

Publicado sob a [MIT License](LICENSE). Documentação e templates são para fins educacionais e de portfólio.
