# TLS: Per-Service Certificates

> **TL;DR:** Each service gets its own HTTPS certificate (e.g. `argocd.remyinthecloud.com`) instead of sharing one big wildcard cert across everything. cert-manager creates the certs automatically from a one-line Ingress annotation, so adding TLS to a new app is nearly free. Smaller blast radius, easier to manage, costs nothing extra.

## What is cert-manager?

cert-manager is a tool that runs in our cluster and automatically gets TLS certificates from Let's Encrypt.

Before cert-manager, you'd:
1. Log into a server
2. Run Certbot to get a certificate
3. Save the cert file
4. Configure your web server to use it
5. Set up a cron job to renew it every 60 days
6. Hope nothing broke

With cert-manager: you create a YAML file describing what you want, cert-manager does the rest automatically forever.

(Need a refresher on certificates and Let's Encrypt? See [TLS, Certificates, Let's Encrypt, and ACME](tls-certificates.md).)

## What is a ClusterIssuer?

A ClusterIssuer is a configuration that tells cert-manager **how** to get certificates.

Our ClusterIssuer (`letsencrypt-prod`) tells cert-manager:

- Use Let's Encrypt's production server
- Use DNS-01 challenge with the Cloudflare API token
- Use this email for expiry notices

A ClusterIssuer is "cluster-wide" (available to any namespace), unlike a regular Issuer which is limited to one namespace.

## What is a TLS Secret?

A certificate alone isn't enough — the web server also needs the **private key** (the secret half of the encryption pair). A **TLS Secret** is a Kubernetes Secret that stores the cert and its private key together, named after the service (e.g. `argocd-remyinthecloud-com-tls`).

Whoever references the Secret gets both halves. That's why spreading certs around is dangerous — the more places the private key lives, the more ways it can leak.

## What is a Certificate resource?

A Certificate is a YAML that says "I want a cert for this domain, issued by this ClusterIssuer."

When you create a Certificate resource, cert-manager:

1. Reads the Certificate YAML
2. Asks the ClusterIssuer for instructions
3. Talks to Cloudflare's API (DNS-01 challenge)
4. Gets the cert from Let's Encrypt
5. Stores it as a TLS Secret
6. Renews it automatically before expiry

## The wildcard certificate (we have one, but don't use it)

A **wildcard certificate** covers an entire domain at once: `*.remyinthecloud.com` works for `argocd.`, `grafana.`, `anything.` in front of the domain. Let's Encrypt issues it as a single cert with a single private key. cert-manager stores it as a Kubernetes Secret in the cert-manager namespace.

We actually have a wildcard Certificate in the repo (`kubernetes/infrastructure/cert-manager/wildcard-certificate.yaml`) — but **no Ingress uses it**. It exists as a fallback, and we deliberately chose per-service certs instead. Why? See below.

## The decision: per-service certs vs. sharing the wildcard

When you have a wildcard cert secret, you have two choices for how to use it:

### Option A: Share the wildcard secret (what we don't do)

- Copy the wildcard secret (`*.remyinthecloud.com`) into every namespace that needs TLS
- Every Ingress references the same secret

**Problems:**

- The private key is now copied across every namespace. If one namespace is compromised, the attacker gets the key for all your domains.
- If you accidentally delete the secret in one namespace, that service breaks.
- Hard to track where the secret has been copied.

### Option B: Per-service certs (what we do)

- Each service gets its own Certificate for its specific domain
- For example: `argocd.remyinthecloud.com` gets its own cert
- cert-manager issues a separate cert (using the same wildcard-capable ClusterIssuer)
- Each cert lives only in its own namespace

**Benefits:**

- If one namespace is compromised, only that service's cert is exposed. The rest of your services are fine.
- Each namespace manages its own cert. No cross-namespace dependencies.
- Cleaner auditing — you know exactly which certs exist and where.

## But isn't it wasteful to get multiple certs?

Each cert is free (Let's Encrypt). There's no cost to having separate certs per service. The only "cost" is a few extra API calls to Let's Encrypt, which is negligible.

For a homelab, this is the right choice — more secure and easier to manage.

## How it works in practice (the annotation trick)

You might expect us to write a full Certificate YAML per service. We don't need to — cert-manager has a shortcut:

**Put one annotation on the Ingress** and cert-manager creates the Certificate (and the TLS Secret) automatically:

```yaml
metadata:
  annotations:
    cert-manager.io/cluster-issuer: letsencrypt-prod
spec:
  tls:
    - hosts:
        - argocd.remyinthecloud.com
      secretName: argocd-remyinthecloud-com-tls
```

This is exactly what our ArgoCD Ingress does (`kubernetes/infrastructure/ingress/argocd.yaml`). cert-manager sees the annotation, reads which domains are listed under `spec.tls.hosts`, requests a cert from Let's Encrypt, and stores it as the Secret named in `secretName`.

So when we add a new service (say, Grafana):

1. Write the Ingress with the `cert-manager.io/cluster-issuer` annotation and a `secretName`
2. Push to Git — ArgoCD applies it (see [GitOps and ArgoCD](gitops-argocd.md))
3. cert-manager sees the annotation, gets the cert from Let's Encrypt, stores it as a Secret
4. Done. The cert auto-renews forever.

(The full Certificate resource is still available if you ever need more control — e.g. extra `dnsNames`, custom durations — but the annotation covers 95% of cases.)

No manual cert management, no copying secrets across namespaces, no single point of failure.

## Key point

**We use the same DNS-01 ClusterIssuer to get many smaller certs instead of sharing one big wildcard cert everywhere.** One Ingress annotation triggers it automatically. More secure, more manageable, and costs nothing extra.

## Terms

| Term | Plain-language meaning |
|------|------------------------|
| **Certbot** | The classic manual tool for getting Let's Encrypt certs |
| **ClusterIssuer** | Config telling cert-manager how to get certs (Let's Encrypt prod, DNS-01, Cloudflare token) |
| **Issuer** | Same as ClusterIssuer, but limited to one namespace |
| **Certificate** | A YAML resource requesting a cert for specific domains |
| **TLS Secret** | A Kubernetes Secret holding a cert + its private key |
| **Wildcard cert** | One cert covering `*.domain.com` |
| **DNS-01** | Proving domain ownership via a DNS record (see [TLS, Certificates](tls-certificates.md)) |
| **Annotation** | A key-value label on a Kubernetes resource that extra tools (like cert-manager) read |
| **Private key** | The secret half of encryption — leaks are why we don't share certs across namespaces |
| **Namespace** | A virtual partition inside the cluster that isolates apps from each other |

## Related docs

- [TLS, Certificates, Let's Encrypt, and ACME](tls-certificates.md) — how DNS-01 and Let's Encrypt work
- [Cloudflare Tunnel](cloudflare-tunnel.md) — Cloudflare's edge TLS in front of our cluster
- [GitOps and ArgoCD](gitops-argocd.md) — how all of this gets deployed from Git
