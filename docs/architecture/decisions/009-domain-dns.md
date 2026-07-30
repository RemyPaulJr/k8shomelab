# ADR 009: Domain & External DNS

- **Date:** 2026-07-29
- **Status:** Accepted

## Context

The cluster needs a domain name for external access. All services will be exposed as subdomains under a single root domain, with TLS certificates managed automatically.

## Decision

- **Domain:** `remyinthecloud.com`
- **Pattern:** `*.remyinthecloud.com` wildcard with per-service subdomains
- **DNS:** Cloudflare DNS (authoritative)
- **TLS:** Per-Ingress certificates via cert-manager (cert-manager.io/cluster-issuer annotation), backed by Let's Encrypt DNS-01
- **Wildcard cert:** Available for origin TLS use if needed, but per-Ingress certs are the default

## Rationale

- Single domain is simpler to manage than separate domains per service
- Wildcard DNS record (`*`) covers all current and future subdomains without per-record management
- Per-Ingress TLS certs are simpler than sharing a wildcard secret across namespaces (K8s secrets are namespace-scoped)
- Cloudflare DNS-01 validates via API, no open ports needed
- Consistent naming: `subdomain.remyinthecloud.com`

## Alternatives Considered

- Separate domains per service: More DNS management, higher cost
- Wildcard cert shared via cross-namespace secret replication: More complex, anti-pattern
- Self-signed certs: Browser warnings, not suitable for external access

## Consequences

- All services follow `servicename.remyinthecloud.com` naming
- Each Ingress needs `cert-manager.io/cluster-issuer` annotation and a TLS section
- Cloudflare Tunnel public hostnames must match Ingress hostnames
- Cloudflare handles edge TLS; nginx-ingress handles origin TLS via cert-manager
- `argocd.remyinthecloud.com` is the first service to validate the setup
