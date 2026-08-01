# Arquitetura

> **Idioma:** [English](../ARCHITECTURE.md) · Português (Brasil)

> Documentação de arquitetura sanitizada: sem schemas proprietários, endpoints reais ou detalhes de implementação.

---

## 1. Contexto do sistema

A Licitech é um SaaS multi-tenant que agrega e organiza **sinais de compras públicas** (editais, avisos, referências legislativas) e os disponibiliza em fluxos web autenticados.

```mermaid
C4Context
    title Contexto do Sistema — Licitech

    Person(user, "Analista de Compras", "Acompanha editais e fluxos")
    Person(admin, "Admin do Tenant", "Gerencia usuários e preferências")

    System(licitech, "Plataforma Licitech", "SaaS de inteligência em compras públicas")

    System_Ext(sources, "Fontes de Dados Públicas", "Portais governamentais e APIs de dados abertos")
    System_Ext(email, "Provedor de Notificações", "E-mail transacional / webhooks")
    System_Ext(idp, "Identidade (Supabase Auth)", "Provedor de autenticação")

    Rel(user, licitech, "Usa", "HTTPS")
    Rel(admin, licitech, "Administra", "HTTPS")
    Rel(licitech, sources, "Ingere (com rate limit)", "HTTPS")
    Rel(licitech, email, "Envia notificações", "HTTPS")
    Rel(licitech, idp, "Valida sessões", "HTTPS")
```

> [!NOTE]
> Adaptadores de fontes externas são tratados como **I/O não confiável**. Todo conteúdo ingerido é validado, normalizado e armazenado sob políticas por tenant antes de chegar ao usuário.

---

## 2. Fronteiras de serviço

| Fronteira | Responsabilidade | Isolamento de falha |
|---|---|---|
| **Edge UI** | Renderização, roteamento no client, gates de auth no edge | UI degradada sem travar workers |
| **API** | Commands/queries síncronos, authZ | Reinicia independente dos workers |
| **Workers** | Ingestão, enriquecimento, notificações | Crash loops não derrubam a API |
| **Queue** | Buffer e semântica de retry | Back-pressure protege as fontes upstream |
| **Data plane** | Estado durável, RLS, storage | PITR / backups independentes do compute |

```mermaid
graph TB
    subgraph Edge["Fronteira Edge"]
        FE[Frontend]
    end
    subgraph App["Fronteira de Aplicação"]
        API[API]
        W[Workers]
        Q[Queue]
    end
    subgraph Data["Fronteira de Dados"]
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

## 3. Componentes do backend

| Componente | Papel | Estado |
|---|---|---|
| **Processo da API** | Superfície REST/JSON (ou RPC); validação; orquestração | Stateless |
| **Auth middleware** | Verificação de JWT, extração de claims, vínculo ao tenant | Stateless |
| **Domain services** | Orquestração de casos de uso (sem SQL direto nos controllers) | Stateless |
| **Camada de repositório** | Abstração de persistência | Stateless |
| **Worker runners** | Consomem jobs; handlers idempotentes | Stateless |
| **Scheduler / poller** | Enfileira janelas periódicas de ingestão | Stateless |

> [!TIP]
> Controllers ficam magros. Regras de domínio ficam nos services. Persistência é trocável atrás dos repositórios — assim Supabase/Postgres fica como detalhe de implementação do data plane.

---

## 4. Arquitetura de workers

Workers são **processos com escala horizontal** consumindo de uma queue compartilhada.

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
    H3 --> EXT[Provedor de Notificações]

    H1 -->|retry / DLQ| Q
    H2 -->|retry / DLQ| Q
```

**Regras de design**

1. **Idempotency keys** em todo job — a gente assume entrega at-least-once
2. **Retries limitados** com exponential backoff + jitter
3. Caminho de **dead-letter** para poison messages depois do máximo de tentativas
4. **Visibility timeout** para workers que crasham não deixarem jobs órfãos
5. **Sem memória mutável compartilhada** entre workers

---

## 5. Fluxo de dados

```mermaid
flowchart LR
    SRC[Fontes Públicas] -->|pull| ING[Worker de Ingestão]
    ING -->|normaliza / valida| NORM[Registros Normalizados]
    NORM -->|upsert| DB[(PostgreSQL)]
    DB -->|read| API[API]
    API -->|JWT + RLS| TENANT[Resultados por Tenant]
    TENANT --> UI[Frontend]
```

**Princípios**

- O caminho de escrita é orientado a append/upsert; atualizações são auditáveis quando necessário
- O caminho de leitura é filtrado por RLS no banco — filtros na aplicação são a segunda linha, não a única
- Payloads grandes (anexos, snapshots brutos) vão para object storage; o DB guarda referências e metadados

---

## 6. Fluxo de eventos

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
    alt Regras de matching do usuário batem
        W->>Q: enqueue(notify)
        Q->>N: deliver(notify_job)
        N->>N: send channel message
    end
    W-->>Q: ACK
```

Eventos são **commands numa queue**, não um event bus completo — escolhido pela simplicidade operacional na escala atual. Veja o [roadmap de ADRs](../../ROADMAP-pt-br.md) para quando faria sentido reconsiderar um backbone de eventos mais rico.

---

## 7. Processamento da queue

| Preocupação | Abordagem |
|---|---|
| Entrega | At-least-once |
| Ordenação | Por partição / por chave de entidade quando necessário; senão best-effort |
| Back-pressure | Métricas de profundidade da queue → escala workers / descarta jobs não críticos |
| Poison messages | Máximo de retries → dead-letter + alerta |
| Observabilidade | Correlation de job ID nos logs: API → queue → worker |

---

## 8. Design stateless

Todos os processos de aplicação são **stateless**:

- Sem session store em processo
- Sem sticky sessions no load balancer
- Configuração via environment (injetada em runtime)
- Escala horizontal = número de réplicas

Estado durável:

| Store | Conteúdo |
|---|---|
| PostgreSQL | Entidades canônicas, tenancy, atributos de authz |
| Redis | Queue / cache de curta duração / contadores de rate limit |
| Object storage | Blobs e artefatos brutos |

---

## 9. Isolamento de falhas

```mermaid
graph TD
    subgraph FD1["Domínio de Falha: Edge"]
        V[Deploy Vercel]
    end
    subgraph FD2["Domínio de Falha: API"]
        A1[Réplica API 1]
        A2[Réplica API 2]
    end
    subgraph FD3["Domínio de Falha: Workers"]
        W1[Pool de Workers]
    end
    subgraph FD4["Domínio de Falha: Dados"]
        PG[(PostgreSQL)]
        R[(Redis)]
    end

    V -.->|independente| FD2
    FD2 -.->|independente| FD3
    FD2 --> FD4
    FD3 --> FD4
```

| Falha | Raio de impacto | Mitigação |
|---|---|---|
| Crash de um container da API | Requests em voo naquela réplica | Restart policy + multi-réplica |
| Saturação do pool de workers | Jobs assíncronos atrasados | Scale out; descarta trabalho de baixa prioridade |
| Redis indisponível | Queue pausa | Alertas; modo degradado (só caminhos sync críticos, se definidos) |
| Postgres indisponível | Outage total de leitura/escrita | Fail closed; restore de backup / PITR |

---

## 10. Rate limiting

| Camada | Mecanismo | Propósito |
|---|---|---|
| Edge | Regras da plataforma / WAF | Abuso volumétrico |
| API | Orçamentos por token / IP / tenant | Uso justo e controle de custo |
| Workers | Limites de educação por fonte | Proteger APIs públicas upstream |
| Database | Caps de pool | Evitar connection storms |

Quando o limite estoura, a gente devolve erros explícitos e amigáveis a retry (ex.: `429` com semântica de `Retry-After` quando aplicável).

---

## 11. Estratégia de retry

```mermaid
flowchart TD
    START[Tentativa N] --> OK{Sucesso?}
    OK -->|Sim| DONE[ACK / Commit]
    OK -->|Não| RETRYABLE{Erro retryable?}
    RETRYABLE -->|Não| FAIL[Falha + alerta / DLQ]
    RETRYABLE -->|Sim| MAX{N < max?}
    MAX -->|Sim| BACKOFF[Exponential backoff + jitter]
    BACKOFF --> START
    MAX -->|Não| DLQ[Dead-letter + page]
```

- Distinguir **transitório** (timeouts, 503) de **permanente** (validação, 404)
- Limitar tentativas totais; nunca retry infinito no hot path
- Idempotency tokens evitam side effects duplicados

---

## 12. Alta disponibilidade

| Camada | Postura de HA |
|---|---|
| Frontend | Rede edge multi-AZ (Vercel) |
| API | N ≥ 2 réplicas atrás de reverse proxy |
| Workers | N ≥ 2; a queue absorve outages curtos |
| PostgreSQL | HA gerenciado / failover automático (provedor) |
| Redis | Modo de persistência adequado à durabilidade da queue |

Endpoints de health (`/health/live`, `/health/ready`) controlam tráfego e restarts da orquestração.

---

## 13. Escala horizontal

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

Gatilhos de escala (manual hoje; automatizado no roadmap):

- API: CPU / latência p95 / RPS
- Workers: profundidade da queue / lag / idade do job

---

## 14. Domínios de falha — resumo

| Domínio | Contém | Restart independente? |
|---|---|---|
| Edge | Deploy Next.js | Sim |
| Proxy | Nginx | Sim |
| API | Containers da aplicação | Sim |
| Workers | Consumidores de jobs | Sim |
| Cache/Queue | Redis | Sim (com risco de drain da queue) |
| Dados | PostgreSQL + storage | Orientado a restore |

---

## Documentos relacionados

- [C4 Context](C4/CONTEXT.md) · [Container](C4/CONTAINER.md) · [Component](C4/COMPONENT.md)
- [SCALABILITY.md](SCALABILITY.md) · [PERFORMANCE.md](PERFORMANCE.md)
- [THREAT_MODEL.md](THREAT_MODEL.md) · [ADR/](ADR/)
