# Daily Operations

## Cluster Access

```bash
# SSH to master
ssh baki@control_node

# Or use k9s from main PC (once kubeconfig is set up)
k9s --context k3shomelab
```

## Basic Commands

```bash
# Node status
k3s kubectl get nodes -o wide

# Pod status
k3s kubectl get pods -A

# Describe a problematic pod
k3s kubectl describe pod -n <ns> <pod>
k3s kubectl logs -n <ns> <pod>

# Cluster events
k3s kubectl get events -A --sort-by='.lastTimestamp'
```

## ArgoCD

```bash
# Install (one-time bootstrap)
kubectl create ns argo-cd
kubectl apply -n argo-cd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
kubectl wait --for=condition=Ready pods --all -n argo-cd

# Get admin password
kubectl -n argo-cd get secret argocd-initial-admin-secret \
  -o jsonpath="{.data.password}" | base64 -d

# Access UI
kubectl port-forward svc/argocd-server -n argo-cd 8080:443

# Apply root app (self-management)
kubectl apply -f kubernetes/argocd/bootstrap/root-app.yaml

# Sync an application
argocd app sync <app-name>
```

## cert-manager

Secrets required before ClusterIssuer can provision certificates:

```bash
# Create Cloudflare API token secret in cert-manager namespace
# Token needs Zone:DNS:Edit permission in Cloudflare
kubectl create secret generic cloudflare-api-token \
  -n cert-manager \
  --from-literal=api-token=<your-cloudflare-api-token>
```

Verify:
```bash
k3s kubectl get clusterissuer -A
k3s kubectl get certificate -A
```

## ingress-nginx

If replacing Traefik on an existing k3s install, disable Traefik first:

```bash
# On master node
k3s kubectl -n kube-system delete helmchart traefik
k3s kubectl -n kube-system delete helmchart traefik-crd
```

Verify:
```bash
k3s kubectl get pods -n ingress-nginx
```

## Cloudflare Tunnel

Secrets required before cloudflared can connect:

```bash
# 1. Create the tunnel on your local machine
cloudflared tunnel create <tunnel-name>

# 2. Get the tunnel token
cloudflared tunnel token <tunnel-name>

# 3. Create secret in the cluster
kubectl create secret generic cloudflare-tunnel-credentials \
  -n cloudflare-tunnel \
  --from-literal=tunnel-token=<tunnel-token>
```

Verify:
```bash
k3s kubectl get pods -n cloudflare-tunnel
k3s kubectl logs -n cloudflare-tunnel deploy/cloudflared
```

## Longhorn (Once Installed)

```bash
# Check volumes
k3s kubectl -n longhorn-system get volumes

# Via UI: https://longhorn.yourdomain.com
```

## Backups

See `docs/runbooks/backup-restore.md` for backup procedures.
