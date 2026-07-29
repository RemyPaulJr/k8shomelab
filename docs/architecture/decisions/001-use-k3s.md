# ADR 001: Use k3s as Kubernetes Distribution

- **Date:** 2026-06-01
- **Status:** ✅ Accepted

## Context

We needed a Kubernetes distribution for a two-node homelab (HP EliteDesk 64GB + Qotom 8GB). Options considered: full k8s, MicroK8s, k3s, K0s, and K3s.

## Decision

Use **k3s** (Rancher's lightweight Kubernetes).

## Rationale

- Single-binary installation, minimal dependencies
- Low resource footprint — important for the 8GB worker node
- Built-in Traefik ingress, local-path-provisioner, Helm controller
- Simple multi-node setup (master + agent with a token)
- Large community, well-documented
- Compatible with standard K8s APIs and tools (kubectl, Helm, ArgoCD)

## Consequences

- Control plane runs on one node only (no HA — acceptable for homelab)
- Some edge-case features may differ from upstream K8s
- Traefik as default ingress may be replaced later depending on needs
