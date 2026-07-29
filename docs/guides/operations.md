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
kubectl -n argo-cd get secret argo-cd-initial-admin-secret \
  -o jsonpath="{.data.password}" | base64 -d

# Access UI
kubectl port-forward svc/argo-cd-argocd-server -n argo-cd 8080:443

# Apply root app (self-management)
kubectl apply -f kubernetes/argocd/bootstrap/root-app.yaml

# Sync an application
argocd app sync <app-name>
```

## Longhorn (Once Installed)

```bash
# Check volumes
k3s kubectl -n longhorn-system get volumes

# Via UI: https://longhorn.yourdomain.com
```

## Backups

See `docs/runbooks/backup-restore.md` for backup procedures.
