# ADR 0003 — Supabase Selection

> **Language:** English · [Português (Brasil)](../pt-br/ADR/0003-supabase-selection.md)

- **Status:** Accepted
- **Date:** 2025-01-20
- **Deciders:** Platform / Engineering Lead

---

## Context

Licitech needs:

- A relational system of record with strong consistency
- Multi-tenant isolation enforceable close to the data
- Authentication suitable for a SaaS product
- Object storage for artifacts, without standing up a separate stack on day one

The team preferred to buy undifferentiated heavy lifting (HA Postgres, auth, storage) and spend engineering on procurement domain flows.

---

## Decision

Adopt **Supabase** as the managed data and auth plane:

- **PostgreSQL** for canonical state
- **Supabase Auth** for identity
- **Storage** for blobs
- **Row Level Security** as mandatory tenancy control

Application code accesses data through repository abstractions, so the vendor does not leak across the entire codebase.

---

## Alternatives Considered

| Alternative | Why not (now) |
|---|---|
| **Self-managed Postgres + Keycloak + MinIO** | High ops load for the same primitives |
| **Firebase / Firestore** | Weaker relational model for procurement entities |
| **PlanetScale / serverless MySQL + custom auth** | RLS-style policy model less natural; more glue code |
| **AWS Amplify / AppSync** | More AWS coupling than we want |

---

## Trade-offs

| Benefit | Cost |
|---|---|
| Fast multi-tenant security baseline (RLS) | Vendor-feature coupling if overused (e.g. SDK leaking everywhere) |
| Managed backups / HA | Restore drills are still mandatory (trust, but verify) |
| Auth + DB + storage in one place | Provider regional / pricing constraints |
| Less undifferentiated ops | Needs an abstraction layer for future portability |

---

## Consequences

**Positive**

- Tenant isolation tested at the DB layer — defense in depth with API RBAC
- Auth flows ready earlier
- PITR / backup story in the provider's hands, with documented RPO/RTO

**Negative / Follow-ups**

- Forbid ad-hoc `service_role` use outside audited workers
- Keep SQL/policy migrations in version control
- Periodically validate the escape hatch: plain Postgres + external IdP remains viable

**Non-goals**

- Implementing business rules only in database triggers
- Exposing privileged keys in the browser

---

## Related

- [SECURITY_AND_COMPLIANCE.md](../SECURITY_AND_COMPLIANCE.md)
- [DISASTER_RECOVERY.md](../DISASTER_RECOVERY.md)
- [TECHNOLOGY_CHOICES.md](../TECHNOLOGY_CHOICES.md)
