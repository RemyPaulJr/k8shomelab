# TLS, Certificates, Let's Encrypt, and ACME

> **TL;DR:** HTTPS encryption needs a certificate — a "digital ID card" proving a site is who it says it is. We get ours free from Let's Encrypt using DNS-01, which works without opening any ports (required behind CGNAT). cert-manager handles everything automatically.

## What is TLS / SSL?

When you visit a website with `https://` in the URL, the connection is encrypted. No one in between (your ISP, a coffee shop WiFi, etc.) can read what you're sending or receiving. That encryption is TLS (formerly called SSL).

For TLS to work, the server needs a **certificate**. A certificate is like a digital ID card that says "this server is really `argocd.remyinthecloud.com`".

## What is a Certificate Authority (CA)?

Anyone can create a certificate saying they're `google.com`. That's why we need a CA — a trusted third party that checks "does this person actually own this domain?" and then signs the certificate.

Your browser trusts certain CAs by default. If a certificate is signed by a trusted CA, your browser shows a padlock. If it's unsigned or self-made, you get a warning.

## What is Let's Encrypt?

Let's Encrypt is a **free** CA. Unlike paid CAs (which charge $50–$300/year per cert), Let's Encrypt gives you certificates for free. The catch is they expire every 90 days — you need automation to renew them.

**Why 90 days?** Short-lived certs are actually a security feature. If a cert (or its private key) is ever leaked, it becomes useless quickly instead of staying valid for a year. Let's Encrypt trades long expiry for safety — and makes up for the renewal hassle with automation.

That's what **cert-manager** does in our cluster. It talks to Let's Encrypt automatically, gets new certs, and renews them before they expire. You never think about it.

## What is ACME?

ACME is the **protocol** (the language) that cert-manager and Let's Encrypt use to talk to each other.

When cert-manager says "I need a cert for `argocd.remyinthecloud.com`", it speaks ACME. When Let's Encrypt says "prove you own that domain first", it also speaks ACME.

## Why do we need to "prove ownership"?

If anyone could get a cert for any domain without proving ownership, I could get a Let's Encrypt cert for `google.com`, set up a fake site, and your browser would show a padlock. The proof step prevents this.

The proof step is called an **ACME challenge**. There are two types: HTTP-01 and DNS-01.

## HTTP-01 (the simple way)

Let's Encrypt says: "Put this specific file at `http://yourdomain.com/.well-known/acme-challenge/TOKEN` and I'll check it's there."

- Your web server needs to be reachable on port 80 (inbound)
- Your domain must be pointing to your server
- Only works for individual domains (`argocd.remyinthecloud.com`), **not wildcards** (`*.remyinthecloud.com`)

**Why we don't use it:** We can't open port 80 on our home network (CGNAT, no public IP, no inbound ports at all).

## DNS-01 (what we use)

Let's Encrypt says: "Create a DNS record `_acme-challenge.yourdomain.com` with this value, and I'll check the public DNS to see if it's there."

A **DNS record** is an entry in the internet's global phone book (DNS = Domain Name System). Normally DNS records map a name to an IP address ("google.com → 142.250.x.x"). The challenge record is a special temporary one that just proves you control the domain.

The flow:

```
cert-manager ──API call──> Cloudflare creates the challenge DNS record
Let's Encrypt ──checks public DNS──> record found ✅ ──> cert issued
cert-manager ──asks Cloudflare to remove the record──> done
```

- cert-manager connects to Cloudflare's API (outbound only)
- Cloudflare creates the DNS record
- Let's Encrypt reads the DNS record
- Once verified, the record is removed

**Why we use it:**

1. **No inbound ports needed** — cert-manager talks out to Cloudflare, Let's Encrypt reads public DNS. Our router never accepts an incoming connection.
2. **Supports wildcards** — proving you control the DNS zone proves you control `*`.anything.
3. **Works behind CGNAT** — doesn't matter what your IP is, DNS is global.

**How does cert-manager talk to Cloudflare?** With an API token — a secret key stored in the cluster (`kubernetes/infrastructure/cert-manager/cluster-issuer.yaml` references it). It's the key that tells Cloudflare "this request is authorized by the domain owner." Cloudflare then allows cert-manager to create (and remove) DNS records.

## Summary

| | HTTP-01 | DNS-01 |
|---|---|---|
| Needs open port? | Yes (port 80) | No |
| Works behind CGNAT? | No | Yes |
| Supports wildcards? | No | Yes |
| How you prove ownership | Serve a file | Create a DNS record |
| What we use | ❌ | ✅ |

## Terms

| Term | Plain-language meaning |
|------|------------------------|
| **TLS/SSL** | The encryption that protects HTTPS traffic |
| **Certificate** | A digital ID card proving a server is who it claims to be |
| **CA** | A trusted third party that signs certificates |
| **Let's Encrypt** | A free CA; its certs expire every 90 days |
| **ACME** | The protocol cert-manager uses to talk to Let's Encrypt |
| **Challenge** | The "prove you own the domain" step |
| **HTTP-01** | Proves ownership by serving a file on port 80 |
| **DNS-01** | Proves ownership by creating a DNS record |
| **DNS record** | An entry in the internet's global phone book |
| **CGNAT** | Carrier-Grade NAT — your ISP shares one public IP among many customers |
| **API token** | A secret key that authorizes cert-manager to use Cloudflare's API |

## Related docs

- [TLS: Per-Service Certificates](tls-per-ingress.md) — how we apply this to each service
- [Cloudflare Tunnel](cloudflare-tunnel.md) — why our network can't accept inbound connections
- [GitOps and ArgoCD](gitops-argocd.md) — how cert-manager itself gets deployed
