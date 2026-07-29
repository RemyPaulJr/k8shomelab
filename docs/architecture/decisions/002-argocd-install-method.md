# ADR 002: ArgoCD Installation Method

- **Date:** 2026-07-29
- **Status:** ✅ Accepted

## Context

We needed to install ArgoCD as the GitOps agent for the k3s cluster. The installation happens once (bootstrap), after which ArgoCD manages itself via GitOps. We evaluated how to perform this initial install.

## Decision

Use a **two-phase approach**:

1. **Bootstrap:** Apply the upstream ArgoCD install manifest directly from GitHub via `kubectl apply -f` (raw manifests).
2. **Self-management:** Register a root Application that watches this repo's `kubernetes/` directory. From that point on, ArgoCD manages its own configuration.

## Rationale

- Raw manifests require no extra tooling (Helm, operator) on the bootstrap machine
- The bootstrap is a one-time operation — optimization for repeatability is unnecessary
- The upstream install manifest is battle-tested and matches the release exactly
- After bootstrap, ArgoCD can manage its own upgrades through the Helm chart or manifests in the repo
- Avoids the chicken-and-egg problem of using GitOps to install the GitOps tool

## Alternatives Considered

- **Helm CLI install:** `helm install argo-cd argo/argo-cd`. Would require Helm installed. Adds a tool dependency for a one-time operation.
- **ArgoCD Operator:** More complex, adds CRD overhead. Overkill for a homelab.
- **Kustomize:** Could wrap the upstream manifest, but adds no value for the initial install since we're not customizing it.

## Consequences

- The bootstrap step must be run manually (or via a future Ansible playbook)
- Upgrades to ArgoCD itself will be managed through the repo (a manifest change in `kubernetes/argocd/`)
- The root Application in `kubernetes/argocd/bootstrap/root-app.yaml` is the single source of truth for what ArgoCD manages
