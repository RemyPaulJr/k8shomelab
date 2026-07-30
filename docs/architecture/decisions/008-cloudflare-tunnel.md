# ADR 008: External Access via Cloudflare Tunnel

- **Date:** 2026-07-29
- **Status:** Accepted

## Context

Services in the homelab cluster need to be accessible from the internet. The cluster is behind CGNAT (no public IP), and we want to avoid opening any inbound ports for security.

## Decision

Use Cloudflare Tunnel (cloudflared) deployed as a Kubernetes Deployment to expose services externally without opening inbound firewall ports.

## Rationale

- No open ports required — cloudflard establishes an outbound-only connection to Cloudflare's edge
- Free tier handles DNS, TLS, and DDoS protection
- Works behind CGNAT (no public IP needed)
- Handles TLS termination at Cloudflare's edge
- Can route to multiple services via DNS records and ingress rules
- Provides additional security by keeping the origin server IP hidden

## Alternatives Considered

- Port forwarding on home router: Requires public IP or dynamic DNS, opens attack surface
- Tailscale/Funnel: Different auth model, less integrated with Cloudflare DNS
- VPN (WireGuard) + reverse proxy: More complex setup, requires client configuration
- Custom VPS proxy: Additional cost and maintenance

## Consequences

- A Cloudflare Tunnel token must be created via `cloudflared tunnel login` and stored as a Kubernetes Secret
- The cloudflared Deployment runs 2 replicas for high availability
- DNS is managed in Cloudflare — services are exposed as `*.remyinthecloud.com`
- Future ingresses will have TLS managed by Cloudflare (Flexible/Full) with cert-manager for origin certificates
- The tunnel token secret must be provisioned manually or via External Secrets Operator (planned for later)
