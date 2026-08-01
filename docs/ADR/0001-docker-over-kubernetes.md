# ADR 0001 — Docker over Kubernetes

> **Language:** English · [Português (Brasil)](../pt-br/ADR/0001-docker-over-kubernetes.md)

- **Status:** Accepted
- **Date:** 2025-01-15
- **Deciders:** Platform / Engineering Lead

---

## Context

Licitech needed a production deploy model for a small number of long-running services (API, workers), plus managed data services. Team size and traffic profile did not yet justify a full Kubernetes control plane. At the same time, we needed reproducible builds, health-check-based restarts, and horizontal scale without drama.

Constraints:

- Prefer operational simplicity over maximum theoretical elasticity
- Keep a clean exit path if orchestration needs to grow
- Do not couple business logic to a specific orchestrator API

---

## Decision

Package all compute workloads as **Docker images** and run them with **Compose-compatible orchestration on Dokploy** (or an equivalent Docker-centric PaaS). Defer Kubernetes until explicit exit criteria are met.

---

## Alternatives Considered

| Alternative | Why not (now) |
|---|---|
| **Managed Kubernetes (EKS/GKE/AKS)** | Control-plane cost + cognitive load disproportionate to the number of services |
| **Nomad** | Smaller ecosystem for the team's skill set |
| **Serverless containers only** | Poor fit for long-running, queue-oriented workers with a constant poll loop |
| **Bare VM + systemd** | Little reproducibility; configuration drift risk |

---

## Trade-offs

| Benefit | Cost |
|---|---|
| Fast path to production | More manual / simple autoscaling early on |
| Easy parity with local (`compose`) | Less rich service-mesh / policy ecosystem |
| Portable images | The team still needs to harden hosts and networks |
| Low ops overhead | Multi-region scheduling becomes DIY work |

---

## Consequences

**Positive**

- Engineers reason in images and health checks — a skill that transfers to K8s later
- Deploys stay digest-pinned and CI-driven
- Failure domains map cleanly to containers

**Negative / Follow-ups**

- Document exit criteria (see below)
- Invest in Compose/network documentation and host hardening
- Revisit this ADR when queue depth or service count force HPA-like behavior day to day

**Exit criteria toward Kubernetes**

1. Sustained need for multi-service autoscaling policies
2. Multi-host scheduling beyond the comfort of the current PaaS
3. Compliance / network-policy requirements the current model does not cover
4. Operational cost of *not* having K8s exceeds the cost of running it

---

## Related

- [ADR 0002 — Vercel Edge](0002-vercel-edge.md)
- [SCALABILITY.md](../SCALABILITY.md)
- [ROADMAP.md](../../ROADMAP.md)
