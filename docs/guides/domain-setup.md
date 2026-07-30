# Domain Setup Guide

This guide walks through connecting `remyinthecloud.com` to the cluster via Cloudflare Tunnel.

## Prerequisites

- Domain `remyinthecloud.com` registered and using Cloudflare nameservers
- Cloudflare Tunnel manifests already applied (see `kubernetes/infrastructure/cloudflare-tunnel/`)
- Cloudflare API token with `Zone:DNS:Edit` permission (needed for cert-manager)

## Step 1: Create the Tunnel

Run this on your local machine (not on the cluster nodes):

```bash
# Login to Cloudflare (opens browser)
cloudflared tunnel login

# Create the tunnel
cloudflared tunnel create remyinthecloud
```

This generates a credentials file (usually at `~/.cloudflared/<uuid>.json`) and a tunnel ID. Keep this file — you'll need the tunnel token.

## Step 2: Get the Tunnel Token

```bash
cloudflared tunnel token remyinthecloud
```

This outputs the tunnel token string. Copy it.

## Step 3: Deploy the Tunnel Token to the Cluster

SSH to the master node and create the secret:

```bash
kubectl create secret generic cloudflare-tunnel-credentials \
  -n cloudflare-tunnel \
  --from-literal=tunnel-token=<the-token-from-step-2>
```

The cloudflared pods should now connect. Verify:

```bash
kubectl get pods -n cloudflare-tunnel
kubectl logs -n cloudflare-tunnel deploy/cloudflared
```

## Step 4: Configure DNS in Cloudflare

Create a CNAME record pointing `*.remyinthecloud.com` to the tunnel:

```bash
# Using cloudflared CLI:
cloudflared tunnel route dns remyinthecloud *.remyinthecloud.com

# Or via Cloudflare Dashboard:
# 1. Go to Cloudflare Dashboard → remyinthecloud.com → DNS
# 2. Add a CNAME record:
#    Type: CNAME
#    Name: *
#    Target: <tunnel-uuid>.cfargotunnel.com
#    Proxy status: Proxied (orange cloud)
```

The tunnel UUID can be found in:
```bash
cloudflared tunnel info remyinthecloud
```

## Step 5: Configure Public Hostnames (Cloudflare Dashboard)

1. Go to [Cloudflare Zero Trust](https://one.dash.cloudflare.com/) → Access → Tunnels
2. Select the `remyinthecloud` tunnel
3. Click "Configure" → "Public Hostnames"
4. Add entries for each service:

| Subdomain | Service                  | Port |
|-----------|--------------------------|------|
| `argocd`  | `ingress-nginx-controller.ingress-nginx` | 443  |

For each entry:
- **Subdomain**: `argocd`
- **Domain**: `remyinthecloud.com`
- **Type**: HTTPS
- **URL**: `ingress-nginx-controller.ingress-nginx:443`
- **TLS Origin**: Set to "No TLS" (nginx-ingress handles TLS internally via cert-manager)

## Step 6: Verify the Certificate

ArgoCD Ingress uses `cert-manager.io/cluster-issuer: letsencrypt-prod` annotation, so cert-manager will automatically provision a TLS certificate for `argocd.remyinthecloud.com`.

Check progress:

```bash
kubectl get certificate -A
kubectl describe certificate argocd-remyinthecloud-com-tls -n argo-cd
```

## Step 7: Access ArgoCD

Once the certificate is ready and DNS has propagated:

```bash
# Open in browser
open https://argocd.remyinthecloud.com

# Get the admin password
kubectl -n argo-cd get secret argocd-initial-admin-secret \
  -o jsonpath="{.data.password}" | base64 -d
```

## Troubleshooting

### Certificate Not Provisioning
- Verify `cloudflare-api-token` secret exists in `cert-manager` namespace
- Check the API token has `Zone:DNS:Edit` permission
- Check cert-manager logs: `kubectl logs -n cert-manager deploy/cert-manager`
- Check CertificateRequests: `kubectl get certificaterequests -A`

### Tunnel Not Connecting
- Verify `cloudflare-tunnel-credentials` secret exists and has the correct token
- Check cloudflared logs: `kubectl logs -n cloudflare-tunnel deploy/cloudflared`
- Ensure outbound internet access from the cluster nodes

### DNS Not Resolving
- Verify CNAME record exists in Cloudflare DNS
- Check the tunnel is online in Cloudflare Zero Trust dashboard
- DNS propagation can take a few minutes

## Adding More Services

To expose additional services:

1. Create an Ingress resource in `kubernetes/infrastructure/ingress/` with `cert-manager.io/cluster-issuer: letsencrypt-prod` annotation
2. Add a public hostname in Cloudflare Zero Trust dashboard pointing `subdomain.remyinthecloud.com` → `ingress-nginx-controller.ingress-nginx:443`
3. Commit and push — ArgoCD syncs the Ingress
