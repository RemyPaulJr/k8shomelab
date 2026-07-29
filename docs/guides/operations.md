# Daily Operations

## Cluster Access

```bash
# SSH to master
ssh remy@control_node

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

## ArgoCD (Once Installed)

```bash
# Get ArgoCD admin password
k3s kubectl -n argocd get secret argocd-initial-admin-secret \
  -o jsonpath="{.data.password}" | base64 -d

# Sync an application
argocd app sync <app-name>

# Via UI: https://argocd.yourdomain.com
```

## Longhorn (Once Installed)

```bash
# Check volumes
k3s kubectl -n longhorn-system get volumes

# Via UI: https://longhorn.yourdomain.com
```

## Backups

See `docs/runbooks/backup-restore.md` for backup procedures.
