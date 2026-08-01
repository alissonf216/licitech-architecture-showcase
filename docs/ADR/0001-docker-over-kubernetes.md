# ADR 0001 — Docker em vez de Kubernetes

- **Status:** Accepted
- **Date:** 2025-01-15
- **Deciders:** Platform / Engineering Lead

---

## Context

A Licitera precisava de um modelo de deploy em produção para poucos serviços de longa duração (API, workers), além de serviços de dados gerenciados. O tamanho do time e o perfil de tráfego ainda não justificavam um control plane completo de Kubernetes. Ao mesmo tempo, precisávamos de builds reproduzíveis, restart baseado em health check e escala horizontal sem drama.

Restrições:

- Preferir simplicidade operacional a elasticidade teórica máxima
- Manter uma saída limpa se a orquestração precisar crescer
- Não acoplar a lógica de negócio à API de um orquestrador específico

---

## Decision

Empacotar todos os workloads de compute como **imagens Docker** e rodá-los com orquestração **compatível com Compose no Dokploy** (ou PaaS equivalente centrado em Docker). Adiar Kubernetes até critérios de saída explícitos serem atingidos.

---

## Alternatives Considered

| Alternativa | Por que não (agora) |
|---|---|
| **Managed Kubernetes (EKS/GKE/AKS)** | Custo do control plane + carga cognitiva desproporcionais à quantidade de serviços |
| **Nomad** | Ecossistema menor para o skill set do time |
| **Só containers serverless** | Encaixa mal em workers de longa duração, orientados a fila, com poll loop constante |
| **VM bare + systemd** | Pouca reproduzibilidade; risco de drift de configuração |

---

## Trade-offs

| Benefício | Custo |
|---|---|
| Caminho rápido até produção | Autoscaling mais manual / simples no início |
| Paridade fácil com o ambiente local (`compose`) | Ecossistema de service mesh / policies menos rico |
| Imagens portáveis | O time ainda precisa endurecer hosts e redes |
| Pouca sobrecarga de ops | Scheduling multi-região vira trabalho caseiro |

---

## Consequences

**Positivo**

- Engenheiros raciocinam em imagens e health checks — skill que transfere para K8s depois
- Deploys continuam pinados por digest e guiados por CI
- Domínios de falha mapeiam bem para containers

**Negativo / Follow-ups**

- Documentar critérios de saída (veja abaixo)
- Investir em documentação de Compose/rede e hardening de host
- Revisar este ADR quando profundidade de fila ou contagem de serviços forçarem comportamento tipo HPA no dia a dia

**Critérios de saída rumo ao Kubernetes**

1. Necessidade sustentada de policies de autoscaling multi-serviço
2. Scheduling multi-host além do conforto do PaaS atual
3. Requisitos de compliance / network policy que o modelo atual não cobre
4. Custo operacional de *não* ter K8s supera o custo de rodá-lo

---

## Related

- [ADR 0002 — Vercel Edge](0002-vercel-edge.md)
- [SCALABILITY.md](../SCALABILITY.md)
- [ROADMAP.md](../../ROADMAP.md)
