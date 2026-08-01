# ADR 0002 — Vercel Edge para o Frontend

- **Status:** Accepted
- **Date:** 2025-01-15
- **Deciders:** Platform / Engineering Lead

---

## Context

A UI da Licitera é uma aplicação Next.js que precisa de:

- Baixa latência global para páginas de marketing e shells autenticados do app
- Preview deployments por pull request
- Quase zero ops para TLS, CDN e entrega de assets estáticos

Workloads de ingestão e workers de longa duração não combinam com edge functions efêmeras.

---

## Decision

Deployar o **frontend na Edge Network da Vercel**. Manter a **API e os workers em compute Dockerizado** (Dokploy), falando por HTTPS público com requests autenticados via JWT.

---

## Alternatives Considered

| Alternativa | Por que não (agora) |
|---|---|
| **Self-host Next.js na mesma VM** | Perde CDN global e DX de preview; mistura domínios de falha |
| **Cloudflare Pages** | Viável; o time já padronizou no tooling Vercel + Next.js |
| **Netlify** | Trade-offs parecidos; encaixa pior nos workflows Next.js que já usamos |
| **Tudo em edge SSR para APIs** | Timeouts / limites de execução batem de frente com trabalho pesado de servidor |

---

## Trade-offs

| Benefício | Custo |
|---|---|
| CDN e preview de primeira | Pipelines de deploy separados (frontend vs backend) |
| Quase zero ops de TLS/CDN | Acoplamento de vendor no hosting do frontend |
| Separação clara dos workers | Design de cross-origin / cookies precisa ser intencional |
| Escala com a plataforma | Cold-start / limites da plataforma em edge functions se forem mal usadas |

---

## Consequences

**Positivo**

- Falha no frontend não derruba workers, e vice-versa
- Páginas de marketing conseguem bater budgets de performance fortes
- Backend evolui sozinho (tags de imagem)

**Negativo / Follow-ups**

- Regra: nada de jobs pesados em route handlers do Next.js
- Documentar estratégia de CORS / cookie / CSRF entre origins
- Monitorar limites de uso e orçamento da Vercel

**Non-goals**

- Mover consumers Redis ou automações de longa duração para a Vercel
- Guardar secrets só na edge sem validação no backend

---

## Related

- [ADR 0001 — Docker em vez de Kubernetes](0001-docker-over-kubernetes.md)
- [PERFORMANCE.md](../PERFORMANCE.md)
- [ARCHITECTURE.md](../ARCHITECTURE.md)
