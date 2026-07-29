# k8shomelab — AI Context File

## Project Identity

This is a **homelab Kubernetes cluster** running **k3s** on two mini PCs. The goal is a fully GitOps-driven, AI-assisted autonomous homelab where applications are deployed via ArgoCD, monitored with Prometheus/Grafana, exposed externally via Cloudflare Tunnel + a custom domain, and managed through natural language interaction with AI agents.

Maintainer: Remy Paul

## Hardware

| Node     | Model                    | RAM   | Storage | Role                                  |
| -------- | ------------------------ | ----- | ------- | ------------------------------------- |
| Master   | HP Elitedesk G3 800 Mini | 64 GB | 256 GB  | k3s server + general workloads        |
| Worker   | Qotom Mini PC            | 8 GB  | TBD     | k3s agent + lightweight workloads     |

Both run **Debian 13 "Trixie"** (testing branch).

## Repository Structure

```
k8shomelab/
├── ansible/                 # Bootstrap provisioning
│   ├── inventory.ini        # Host definitions (master, worker)
│   ├── host_vars/           # Ansible Vault-encrypted secrets
│   └── playbooks/
│       └── k3s_master.yml   # Installs k3s on master + joins worker
├── docs/                    # Documentation
│   ├── CLAUDE.md            # ← You are here. AI entry point.
│   ├── architecture/        # Design decisions, diagrams
│   ├── guides/              # Setup, operations, troubleshooting
│   └── runbooks/            # DR, recovery procedures
├── kubernetes/              # ArgoCD-managed manifests
│   ├── argocd/              # ArgoCD config
│   │   └── bootstrap/       #   root-app.yaml (app-of-apps)
│   ├── apps/                # Application manifests (deployments, services)
│   └── infrastructure/      # cert-manager, Longhorn, ingress, etc.
├── scripts/                 # (Planned) Helper scripts
├── .github/                 # (Planned) GitHub Actions CI
├── LICENSE                  # MIT License
└── README.md                # Quick overview
```

## Architecture Overview

```
   User/Main PC
      │
      ├── SSH ──► Ansible (bootstrap, phase 1)
      │
      └── Chat/Web ──► AI Agent (in-cluster pod)
                           │
                           ├── Cloud LLM API (OpenAI/Anthropic)
                           ├── GitHub API ──► PR ──► Repo ──► ArgoCD ──► k3s
                           └── K8s API (read-only, for monitoring)
                                    │
Cloudflare ◄── Tunnel ───────────────┤
   │                                  │
   ▼                                  ▼
yourdomain.com                  k3s Cluster
                                    │
                    ┌───────────────┴───────────────┐
                    │                               │
           HP EliteDesk (master)             Qotom (worker)
           k3s server + workloads           k3s agent + workloads
           Longhorn, ArgoCD,                 lightweight apps
           Grafana, AI Agent
```

## Technology Stack

| Layer              | Tool                        | Status      |
| ------------------ | --------------------------- | ----------- |
| Container Runtime  | k3s (Rancher)               | ✅ Deployed |
| Bootstrap          | Ansible + Ansible Vault     | ✅ Done     |
| GitOps             | ArgoCD                      | ✅ Deployed |
| Ingress            | Traefik (k3s default)       | 🔜 Evaluate |
| TLS                | cert-manager + Let's Encrypt | 🔜 Planned  |
| External Access    | Cloudflare Tunnel           | 🔜 Planned  |
| Domain             | Cloudflare DNS              | 🔜 Planned  |
| Storage            | Longhorn                    | 🔜 Planned  |
| SSO/Auth           | OAuth2 Proxy / Authentik    | 🔜 Planned  |
| Monitoring         | Prometheus + Grafana        | 🔜 Planned  |
| Logging            | Loki + Promtail             | 🔜 Planned  |
| Secrets (GitOps)   | External Secrets / SOPS     | 🔜 Planned  |
| AI Agent           | In-cluster agent pod        | 🔜 Planned  |
| AI LLM             | Cloud API (OpenAI/Anthropic) | 🔜 Planned  |
| CI                 | GitHub Actions              | 🔜 Planned  |
| DR/Backup          | Velero + Longhorn snapshots | 🔜 Planned  |

## Roadmap

### Phase 1 — Foundation ✅ (Complete)
- [x] Two-node k3s cluster on Debian 13
- [x] Ansible playbook for bootstrap (install k3s, join worker)
- [x] Ansible Vault for SSH secrets
- [x] GitHub repo as source of truth

### Phase 2 — GitOps + Infrastructure Core (In Progress)
- [x] Install ArgoCD in cluster (app-of-apps pattern)
- [x] Define manifest directory structure (`kubernetes/`)
- [ ] Install cert-manager + ClusterIssuer (Let's Encrypt, DNS-01 with Cloudflare)
- [ ] Set up ingress — evaluate Traefik vs nginx-ingress
- [ ] Deploy Cloudflare Tunnel (cloudflared as pod)
- [ ] Configure domain DNS in Cloudflare, point to tunnel
- [ ] Deploy Longhorn for distributed block storage
- [ ] Set up OAuth2 Proxy / Authentik for SSO on exposed apps
- [ ] Set up External Secrets Operator or SOPS for GitOps-safe secrets

### Phase 3 — Observability
- [ ] Deploy kube-prometheus-stack (Prometheus + Grafana)
- [ ] Deploy Loki + Promtail for log aggregation
- [ ] Configure alerting rules
- [ ] Build dashboards for cluster + app metrics

### Phase 4 — AI Agent Platform
- [ ] Deploy agent pod(s) in-cluster with:
  - ServiceAccount (read-only K8s API + write GitHub API)
  - Cloud LLM API key (OpenAI/Anthropic)
  - GitHub token for PR creation
- [ ] Implement agent tools:
  - `read_cluster_status` — inspect pods, services, events
  - `create_deploy_pr` — generate manifests, open PR
  - `check_argocd_sync` — verify app health
  - `diagnose_failure` — investigate crashes/logs
- [ ] Build chat interface (web UI or Slack/Matrix bot)
- [ ] Wire ArgoCD API for sync triggers

### Phase 5 — Autonomous Behaviors
- [ ] Self-healing: Agent watches Prometheus alerts / K8s events → diagnoses → creates fix PR → ArgoCD syncs
- [ ] Deploy-on-command: "Deploy a Redis instance" → agent generates manifests → PR → sync
- [ ] DR/Restore: Agent can re-invoke Ansible or restore from etcd snapshots / Longhorn backups

### Phase 6 — Enhancements
- [ ] Container registry mirror (cache images locally)
- [ ] GitHub Actions CI (lint/test manifests)
- [ ] Velero for K8s resource backups
- [ ] Local LLM fallback for air-gapped operations
- [ ] Multi-agent setup (monitoring agent, deploy agent, etc.)

## Design Decisions (ADRs)

Each major decision gets a short record in `docs/architecture/decisions/`.

| ADR | Decision | Rationale |
| --- | -------- | --------- |
| 001 | k3s | Lightweight, single-binary, ARM-friendly, simple multi-node setup |
| 002 | ArgoCD install method | Raw manifest bootstrap + self-management via root app |
| 003 | Cloudflare Tunnel | No open ports, free tier, handles DNS+TLS, works behind CGNAT |
| 004 | ArgoCD over Flux | Mature, UI, sync status clarity, broader ecosystem |
| 005 | Longhorn over Rook/Ceph | Simpler to deploy, works well on 2 nodes, UI, built-in backups |
| —   | (Add decisions as they're made) | |

## Workflow

Every change follows this process:

1. **Create a feature branch** from `main` using the naming convention below
2. **Make changes**, commit with descriptive messages
3. **Push branch** to GitHub
4. **Open a pull request** — link any related issues with `Closes #<issue>`
5. **Human reviews** (you) — comment, approve, or request changes
6. **Merge** into `main` after approval
7. **Delete the feature branch** after merge (keeps branch list clean; history is preserved in the merge commit)

### Branch Naming Convention

```
<type>/<short-kebab-description>

Examples:
  feat/argocd-install
  feat/cloudflare-tunnel-setup
  fix/worker-join-error
  docs/add-adr-longhorn
  chore/update-ansible-inventory
  infra/longhorn-storage-setup
  refactor/k3s-playbook
```

**Types:** `feat` | `fix` | `docs` | `chore` | `infra` | `refactor`

### Commit Messages

```
<type>: brief description

Examples:
  feat: add ArgoCD installation manifest
  fix: correct k3s join command in playbook
  docs: add ADR for Longhorn storage decision
```

### GitHub Issues & Projects

- Each task is tracked as a **GitHub Issue** with labels (`phase-2`, `argocd`, `infra`, `storage`, `blocked`, etc.)
- A **GitHub Project** (kanban board) tracks progress: `Backlog → To Do → In Progress → In Review → Done`
- PRs reference issues: `Closes #12` in the PR description
- Branch names can include the issue number: `feat/12-argocd-install`

## AI Agent Instructions

When working on this project, follow these conventions:

1. **Read this file first** — Understand the full context before making changes.
2. **Follow the workflow** — Feature branch → commit → push → PR → human review → merge.
3. **GitOps flow** — All persistent changes to the cluster go through Git. Create/update manifests, not direct kubectl commands (except for debugging).
4. **Document as you go** — Every change should produce a note:
   - What was done and why
   - Any errors encountered and how they were resolved
   - Any gotchas or observations
5. **Where to put things:**
   - K8s manifests → `kubernetes/apps/<app-name>/`
   - Infra manifests → `kubernetes/infrastructure/`
   - ArgoCD bootstrap → `kubernetes/argocd/bootstrap/root-app.yaml`
   - Design decisions → `docs/architecture/decisions/`
   - Troubleshooting entries → `docs/guides/troubleshooting.md`
   - Runbooks → `docs/runbooks/`
6. **Secrets** — Never commit plaintext secrets. Use Ansible Vault (for bootstrap) or External Secrets Operator / SOPS (for GitOps).
7. **Create PRs for review** — All changes go through a PR. You do the work, the human reviews and merges.
8. **Notes are for future AIs and humans** — Write them so someone (or another AI) can pick up where you left off.

## Note-Taking Convention

All notes should follow this structure when applicable:

```
### YYYY-MM-DD: [Brief Title]

- **Context:** What was being done
- **Decision:** What was chosen and why
- **Result:** How it went
- **Errors:** Issues encountered + resolution
- **Docs:** Links to relevant docs/runbooks
```

Keep notes in the relevant file under `docs/architecture/`, `docs/guides/`, or `docs/runbooks/`.

## Common Commands

```bash
# SSH to nodes
ssh baki@control_node
ssh baki@worker_node1

# Run Ansible playbook
ansible-playbook -i ansible/inventory.ini ansible/playbooks/k3s_master.yml --ask-vault-pass

# k3s (on master)
k3s kubectl get nodes
k3s kubectl get pods -A

# k9s (from main PC)
k9s --context k3shomelab

# ArgoCD — install (one-time bootstrap, run from local machine)
kubectl create ns argo-cd
kubectl apply -n argo-cd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
kubectl wait --for=condition=Ready pods --all -n argo-cd

# ArgoCD — get initial admin password
kubectl -n argo-cd get secret argo-cd-initial-admin-secret \
  -o jsonpath="{.data.password}" | base64 -d

# ArgoCD — access UI
kubectl port-forward svc/argo-cd-argocd-server -n argo-cd 8080:443

# ArgoCD — apply root app (after bootstrap)
kubectl apply -f kubernetes/argocd/bootstrap/root-app.yaml
```
