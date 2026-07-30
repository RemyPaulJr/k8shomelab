# ADR 007: Ingress Controller — nginx-ingress

- **Date:** 2026-07-29
- **Status:** Accepted

## Context

The cluster needs an ingress controller to route external traffic to services. k3s ships with Traefik by default, but we need to decide whether to keep it or replace it.

## Decision

Replace the default k3s Traefik ingress controller with **ingress-nginx** (community nginx ingress controller).

## Rationale

- Already deployed and working in the cluster
- Broader ecosystem support and more documentation than Traefik
- More familiar configuration model (standard nginx concepts)
- Better integration with cert-manager (well-documented `kubernetes.io/tls-acme: "true"` annotation)
- More widely used in production Kubernetes environments
- Larger community and more third-party tooling support

## Alternatives Considered

- Traefik (k3s default): Simpler default setup, but less flexible for advanced routing, smaller community
- HAProxy ingress: More complex, overkill for homelab scale
- Istio gateway: Too heavy for a 2-node cluster

## Consequences

- k3s was installed with `--disable=traefik` (or Traefik was removed post-install)
- The default k3s Traefik ingress class is replaced by nginx
- Managed as an ArgoCD Application via Helm chart
- cert-manager annotations work out of the box with the default ingress class
