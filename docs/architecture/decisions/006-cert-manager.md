# ADR 006: cert-manager for TLS Certificate Management

- **Date:** 2026-07-29
- **Status:** Accepted

## Context

All external-facing services need TLS certificates. The cluster needs an automated certificate management solution that integrates with Let's Encrypt and Cloudflare DNS for DNS-01 challenges (since there are no open ingress ports).

## Decision

Use cert-manager with a ClusterIssuer configured for Let's Encrypt (production) using DNS-01 challenge via Cloudflare API token.

## Rationale

- cert-manager is the de-facto standard for K8s certificate management
- Cloudflare DNS-01 works without opening ingress ports (no open ports needed for challenge validation)
- Supports wildcard certificates
- Native integration with ingress controllers via annotations
- Managed as an ArgoCD Application via Helm chart (jetstack repo)

## Alternatives Considered

- Manual cert management via `kubectl create secret tls`: Not automated, no renewal
- Traefik's built-in ACME: Locked to Traefik ingress; we chose nginx-ingress
- Caddy: Would require a different ingress approach entirely

## Consequences

- Cloudflare API token must be created as a Secret (`cloudflare-api-token`) in the cluster before the ClusterIssuer can provision certificates
- cert-manager CRDs are managed via the Helm chart (`installCRDs: true`)
- Certificate renewal is automatic (cert-manager handles it)
- All future ingresses can reference the ClusterIssuer via annotation
