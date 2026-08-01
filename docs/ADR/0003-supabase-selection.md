# ADR 0003 — Escolha do Supabase

- **Status:** Accepted
- **Date:** 2025-01-20
- **Deciders:** Platform / Engineering Lead

---

## Context

A Licitera precisa de:

- Um sistema de registro relacional com consistência forte
- Isolamento multi-tenant aplicável perto dos dados
- Autenticação adequada para um produto SaaS
- Object storage para artefatos, sem levantar um stack separado no dia um

O time preferiu comprar o trabalho pesado indiferenciado (HA Postgres, auth, storage) e gastar engenharia nos fluxos de domínio de procurement.

---

## Decision

Adotar o **Supabase** como plano gerenciado de dados e auth:

- **PostgreSQL** para o estado canônico
- **Supabase Auth** para identidade
- **Storage** para blobs
- **Row Level Security** como controle obrigatório de tenancy

O código da aplicação acessa dados por abstrações de repositório, para o vendor não vazar pelo codebase inteiro.

---

## Alternatives Considered

| Alternativa | Por que não (agora) |
|---|---|
| **Postgres self-managed + Keycloak + MinIO** | Carga de ops alta para as mesmas primitivas |
| **Firebase / Firestore** | Modelo relacional mais fraco para entidades de procurement |
| **PlanetScale / serverless MySQL + auth custom** | Modelo de policy estilo RLS menos natural; mais glue code |
| **AWS Amplify / AppSync** | Mais acoplamento AWS do que queremos |

---

## Trade-offs

| Benefício | Custo |
|---|---|
| Baseline rápido de segurança multi-tenant (RLS) | Acoplamento a features do vendor se exagerar (ex.: SDK vazando em todo lugar) |
| Backups / HA gerenciados | Drills de restore ainda são obrigatórios (confie, mas verifique) |
| Auth + DB + storage no mesmo lugar | Restrições regionais / de preço do provider |
| Menos ops indiferenciada | Precisa de camada de abstração para portabilidade futura |

---

## Consequences

**Positivo**

- Isolamento de tenant testado na camada de DB — defense in depth com RBAC na API
- Auth flows prontos mais cedo
- História de PITR / backup nas mãos do provider, com RPO/RTO documentados

**Negativo / Follow-ups**

- Proibir uso ad-hoc de `service_role` fora de workers auditados
- Manter migrations de SQL/policy no version control
- Validar periodicamente a escape hatch: Postgres puro + IdP externo continua viável

**Non-goals**

- Implementar regras de negócio só em triggers de banco
- Expor chaves privilegiadas no browser

---

## Related

- [SECURITY_AND_COMPLIANCE.md](../SECURITY_AND_COMPLIANCE.md)
- [DISASTER_RECOVERY.md](../DISASTER_RECOVERY.md)
- [TECHNOLOGY_CHOICES.md](../TECHNOLOGY_CHOICES.md)
