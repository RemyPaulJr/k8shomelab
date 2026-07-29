# Architecture Overview

## High-Level Architecture

```
┌─────────────────────────────────────────────────────────────────────┐
│                         Internet                                     │
│                                                                      │
│  ┌──────────────┐     ┌──────────────────┐     ┌─────────────────┐  │
│  │  You (laptop) │────▶│  Cloudflare Edge │◀────│  yourdomain.com │  │
│  │  - k9s        │     │  - DNS           │     │  - ArgoCD UI    │  │
│  │  - git        │     │  - TLS (cert.)   │     │  - Grafana      │  │
│  │  - chat UI    │     │  - DDoS protect  │     │  - Apps         │  │
│  └──────────────┘     └────────┬─────────┘     └─────────────────┘  │
│                                │                                     │
│                        Cloudflare Tunnel (cloudflared)               │
│                                │                                     │
│                          (outbound tunnel, no open ports)            │
└────────────────────────────────┼─────────────────────────────────────┘
                                 │
                    ┌────────────┴────────────┐
                    │                         │
                    ▼                         ▼
          ┌──────────────────┐    ┌──────────────────┐
          │  Control Plane   │    │     Worker       │
          │  (HP EliteDesk)  │    │  (Qotom Mini PC) │
          │                  │    │                  │
          │  ┌────────────┐  │    │  ┌────────────┐  │
          │  │ k3s server │  │    │  │ k3s agent  │  │
          │  └────────────┘  │    │  └────────────┘  │
          │                  │    │                  │
          │  ┌────────────┐  │    │  ┌────────────┐  │
          │  │ ArgoCD     │  │    │  │ (workloads)│  │
          │  └────────────┘  │    │  └────────────┘  │
          │                  │    │                  │
          │  ┌────────────┐  │    │  ┌────────────┐  │
          │  │ Longhorn   │  │    │  │ Longhorn   │  │
          │  │ (storage)  │  │    │  │ (storage)  │  │
          │  └────────────┘  │    │  └────────────┘  │
          │                  │    │                  │
          │  ┌────────────┐  │    │  ┌────────────┐  │
          │  │ Grafana    │  │    │  │ Loki       │  │
          │  │ Prometheus │  │    │  │ Promtail   │  │
          │  └────────────┘  │    │  └────────────┘  │
          │                  │    │                  │
          │  ┌────────────┐  │    │  ┌────────────┐  │
          │  │ AI Agent   │  │    │  │ (lightwt.  │  │
          │  │ (planned)  │  │    │  │  apps)     │  │
          │  └────────────┘  │    │  └────────────┘  │
          └──────────────────┘    └──────────────────┘
```

## Network Flow

```
Public Traffic:
  Browser ──HTTPS──► Cloudflare ──Tunnel──► cloudflared pod ──► Ingress ──► Service ──► Pod

GitOps Flow:
  git push ──► GitHub ──webhook──► ArgoCD ──sync──► k3s cluster

AI Agent Flow:
  Chat ──► Agent pod ──► LLM API (Cloud) ──► GitHub API ──PR──► Repo ──► ArgoCD ──► k3s
                    └──► K8s API (read-only, monitoring)

Admin Flow:
  SSH ──► k3s master ──► kubectl / k9s
```

## Key Architectural Principles

1. **GitOps-first** — All persistent cluster state is defined in this repo. ArgoCD reconciles.
2. **No open ports** — Cloudflare Tunnel provides egress-only connectivity from cluster to internet.
3. **Single sign-on** — OAuth2 Proxy / Authentik in front of all exposed apps.
4. **AI-assisted but human-gated** — AI agents propose changes via PRs; you review and merge.
5. **Documentation as code** — Every design choice, error, and runbook is tracked in `docs/`.
