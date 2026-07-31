# Cloudflare Tunnel

> **TL;DR:** We can't open ports on our home internet (CGNAT), so instead the cluster reaches *out* to Cloudflare and holds an open tunnel. Visitors hit Cloudflare, and Cloudflare passes traffic through the tunnel to us. No inbound ports, free TLS + DDoS protection, and our home IP stays hidden.

## The problem: no public IP and CGNAT

Most home internet connections don't have a real public IP address. Instead, your ISP uses **CGNAT** (Carrier-Grade NAT). This means you share a public IP with dozens of other customers. You can't open ports, you can't forward traffic, and you can't host anything that needs inbound connections.

Even if you *do* have a public IP, opening ports means:

- Your home IP is exposed to the internet
- You're responsible for DDoS protection, TLS certs, and security patches on the edge
- Your ISP might block port 80/443

## The old solution: port forwarding + reverse proxy

Before Cloudflare Tunnel, you'd:

1. Buy a domain
2. Set your public IP in a DNS A record
3. Open port 443 on your router (port forward)
4. Run a reverse proxy (like nginx) that routes traffic to the right service
5. Set up Let's Encrypt manually for TLS

This works but: **you need a public IP, you open inbound ports, your home IP is public.**

## The Cloudflare Tunnel solution

Instead of opening a port and waiting for connections, your cluster **reaches out to Cloudflare and makes a tunnel**. Think of it like a secret passage:

```
Browser ──HTTPS──> Cloudflare ────tunnel────> cloudflared pod ──> nginx ──> your app
```

- The tunnel is initiated *from inside your network* (outbound connection)
- Cloudflare accepts requests on your domain and forwards them through the tunnel
- Your router never accepted an inbound connection

## What cloudflared does

`cloudflared` is a small program that runs as a pod in our cluster. It:

1. Connects to Cloudflare's edge (outbound)
2. Keeps the connection alive
3. Forwards incoming requests from Cloudflare to the right internal service (nginx)

We run 2 replicas for redundancy. If one dies, the other keeps the tunnel open.

## What happens if the tunnel drops?

The tunnel is a persistent connection, and connections can break (ISP hiccup, pod restart, Cloudflare maintenance). When that happens:

- **cloudflared reconnects automatically** — it's designed to retry endlessly until the tunnel is back
- **The second replica covers the gap** — two connections are up at once; if one drops, traffic flows through the other while the first reconnects
- **You usually won't notice** — visitors might see a brief loading pause at worst

This is exactly why we run 2 replicas: the hard part isn't making the tunnel, it's making sure it *comes back* — and the replicas handle that for us.

## What we get for free from Cloudflare

| Feature | What it means |
|---------|---------------|
| **TLS termination** | Cloudflare puts its own HTTPS certificate on the user-facing side, so your browser sees a valid padlock before traffic ever reaches us. (Cloudflare → us is also encrypted — we run our own certs inside, see [TLS: Per-Service Certificates](tls-per-ingress.md).) |
| **DDoS protection** | Cloudflare's network absorbs attacks before they reach your home. Your little home server never sees the flood. |
| **Origin IP hiding** | The only IP anyone sees is Cloudflare's. Your home IP stays hidden. Even if someone finds the tunnel egress IP, it's Cloudflare's, not yours. |
| **Free** | Basic plan is free for personal use. |

## CGNAT compatibility

Since the tunnel is an **outbound connection**, CGNAT doesn't matter. Your router sees the tunnel as just another outbound request (like loading a webpage). The tunnel stays open, and Cloudflare can send data back through it at any time.

This is the critical reason we chose this approach — no other method works behind CGNAT without a VPS proxy.

## How our traffic flows for a real request

1. You type `argocd.remyinthecloud.com` in a browser
2. DNS resolves to Cloudflare's IP
3. Cloudflare receives the request
4. Cloudflare finds an active tunnel to our cluster
5. Cloudflare sends the request through the tunnel
6. `cloudflared` pod receives it
7. `cloudflared` forwards it to the nginx ingress controller
8. nginx routes to the ArgoCD web UI Service
9. ArgoCD responds, data flows back the same path

All of this happens in milliseconds. To you, it feels like loading any normal website.

## Key point

**We have zero inbound firewall ports open.** The tunnel is like calling Cloudflare, handing them the phone, and saying "when someone asks for me, tell them to talk through this line."

## Terms

| Term | Plain-language meaning |
|------|------------------------|
| **CGNAT** | Carrier-Grade NAT — your ISP shares one public IP among many customers |
| **Public IP** | The internet-facing address of a network (ours is shared, not ours alone) |
| **Port forwarding** | Telling your router "send traffic on port X to machine Y" — needs a public IP |
| **Reverse proxy** | A server that receives traffic and forwards it to the right app |
| **Tunnel** | A persistent outbound connection that Cloudflare uses to reach us |
| **cloudflared** | The program that maintains the tunnel (runs as a pod in our cluster) |
| **Replica** | An extra copy of a pod for redundancy |
| **TLS termination** | The edge (Cloudflare) handling the HTTPS certificate so browsers trust the site |
| **Origin IP** | The real IP of our home network (hidden behind Cloudflare) |
| **VPS** | A rented server on the public internet — the expensive alternative to a tunnel |

## Related docs

- [TLS, Certificates, Let's Encrypt, and ACME](tls-certificates.md) — how HTTPS certs work at both edges
- [TLS: Per-Service Certificates](tls-per-ingress.md) — the certs we use inside the cluster
- [GitOps and ArgoCD](gitops-argocd.md) — how cloudflared is deployed and managed
