# k8shomelab

> Building a fully autonomous, AI-driven Kubernetes homelab — in public.
> Two mini PCs, one GitHub repo, and ArgoCD making it all tick.

## About

This is my homelab Kubernetes cluster running **k3s** on two mini PCs at home. The goal is a self-healing, AI-assisted cluster managed entirely through GitOps — deploy apps, heal failures, and make changes by talking to AI agents.

Everything is open source and built in public. [remyinthecloud.com](https://remyinthecloud.com) documents the journey.

## Hardware

| Node | Model | RAM | Storage | Role |
| ---- | ----- | --- | ------- | ---- |
| Master | HP EliteDesk G3 800 Mini | 64 GB | 256 GB SSD | Control plane + workloads |
| Worker | Qotom Mini PC | 8 GB | — | Worker node |

Both run **Debian 13 "Trixie"** with **k3s**.

## Stack

| Layer | Tool | Status |
| ----- | ---- | ------ |
| Orchestration | k3s | ✅ Deployed |
| Bootstrap | Ansible + Vault | ✅ Done |
| GitOps | ArgoCD | ✅ Deployed |
| TLS | cert-manager + Let's Encrypt | ✅ Deployed |
| Ingress | nginx-ingress | ✅ Deployed |
| External Access | Cloudflare Tunnel | 🔜 Planned |
| Storage | Longhorn | 🔜 Planned |
| Monitoring | Prometheus + Grafana | 🔜 Planned |
| AI Agents | In-cluster agent pod | 🔜 Planned |

## Architecture

```
Main PC ──► GitHub Repo ──► ArgoCD ──► k3s Cluster
                                          ├── HP (master)
                                          └── Qotom (worker)
```

External access via Cloudflare Tunnel, monitoring with Prometheus/Grafana, AI agents for self-healing and deploy-on-command. Full architecture diagram and roadmap in [docs/CLAUDE.md](docs/CLAUDE.md).

## Ecosystem

This homelab is part of a larger personal cloud ecosystem:

| Project | Description | Links |
| ------- | ----------- | ----- |
| **k8shomelab** | This repo — GitOps source of truth for the cluster | [GitHub](https://github.com/RemyPaulJr/k8shomelab) |
| **remyinthecloud.com** | Personal website + blog documenting the homelab journey. Built with Jekyll, hosted on S3, deployed via GitHub Actions. | [Site](https://remyinthecloud.com) · [Repo](https://github.com/RemyPaulJr/remyinthecloud.com) |

## Roadmap

| Phase | Status |
| ----- | ------ |
| 1 — Foundation (k3s + Ansible bootstrap) | ✅ Complete |
| 2 — GitOps + Infrastructure (ArgoCD, cert-manager, ingress, tunnel, storage) | 🔜 In Progress |
| 3 — Observability (Prometheus, Grafana, Loki) | 🔜 Planned |
| 4 — AI Agent Platform | 🔜 Planned |
| 5 — Autonomous Behaviors (self-healing, deploy-on-command) | 🔜 Planned |
| 6 — Enhancements | 🔜 Planned |

Full roadmap with checkboxes in [docs/CLAUDE.md](docs/CLAUDE.md).

## Connect

- **Website:** [remyinthecloud.com](https://remyinthecloud.com)
- **GitHub:** [@RemyPaulJr](https://github.com/RemyPaulJr)
