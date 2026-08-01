# ADR 0002 — Vercel Edge for the Frontend

> **Language:** English · [Português (Brasil)](../pt-br/ADR/0002-vercel-edge.md)

- **Status:** Accepted
- **Date:** 2025-01-15
- **Deciders:** Platform / Engineering Lead

---

## Context

Licitech's UI is a Next.js application that needs:

- Low global latency for marketing pages and authenticated app shells
- Preview deployments per pull request
- Near-zero ops for TLS, CDN, and static asset delivery

Ingestion workloads and long-running workers do not fit ephemeral edge functions.

---

## Decision

Deploy the **frontend on Vercel's Edge Network**. Keep the **API and workers on Dockerized compute** (Dokploy), talking over public HTTPS with JWT-authenticated requests.

---

## Alternatives Considered

| Alternative | Why not (now) |
|---|---|
| **Self-host Next.js on the same VM** | Loses global CDN and preview DX; mixes failure domains |
| **Cloudflare Pages** | Viable; the team already standardized on Vercel + Next.js tooling |
| **Netlify** | Similar trade-offs; weaker fit for the Next.js workflows we already use |
| **Everything as edge SSR for APIs** | Timeouts / execution limits clash with heavy server-side work |

---

## Trade-offs

| Benefit | Cost |
|---|---|
| First-class CDN and previews | Separate deploy pipelines (frontend vs backend) |
| Near-zero TLS/CDN ops | Vendor coupling on frontend hosting |
| Clear separation from workers | Cross-origin / cookie design must be intentional |
| Scales with the platform | Cold-start / platform limits on edge functions if misused |

---

## Consequences

**Positive**

- Frontend failure does not take down workers, and vice versa
- Marketing pages can hit strong performance budgets
- Backend evolves independently (image tags)

**Negative / Follow-ups**

- Rule: no heavy jobs in Next.js route handlers
- Document CORS / cookie / CSRF strategy across origins
- Monitor Vercel usage limits and budget

**Non-goals**

- Moving Redis consumers or long-running automations to Vercel
- Storing secrets only on the edge without backend validation

---

## Related

- [ADR 0001 — Docker over Kubernetes](0001-docker-over-kubernetes.md)
- [PERFORMANCE.md](../PERFORMANCE.md)
- [ARCHITECTURE.md](../ARCHITECTURE.md)
