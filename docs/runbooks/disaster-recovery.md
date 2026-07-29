# Disaster Recovery Runbook

*This will be filled in as backup and recovery procedures are established.*

## Scope

Recovering the entire cluster from a catastrophic failure (e.g., both nodes offline, corrupted etcd, complete data loss).

## Prerequisites

- [ ] Ansible installed on recovery machine
- [ ] Access to this repo
- [ ] Vault password for Ansible
- [ ] (Future) Longhorn backup target (S3/NFS)
- [ ] (Future) Velero backup of K8s resources

## Recovery Procedure

### Full Cluster Restore

1. Reinstall Debian 13 on both nodes
2. Configure SSH access
3. Run Ansible playbook to bootstrap k3s:
   ```bash
   cd k8shomelab
   ansible-playbook -i ansible/inventory.ini \
     ansible/playbooks/k3s_master.yml \
     --ask-vault-pass
   ```
4. Verify cluster is up: `k3s kubectl get nodes`
5. Install ArgoCD and point to this repo:
   ```bash
   k3s kubectl create namespace argocd
   k3s kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
   ```
6. ArgoCD will sync all apps from the repo
7. Restore Longhorn volumes from backups (if applicable)
8. Restore etcd snapshot (if applicable)

### Partial Recovery

- **Single node down:** See `node-failure.md`
- **ArgoCD broken:** Reapply manifests from repo
- **Certificate expiry:** cert-manager should auto-renew; check `k3s kubectl get certificates -A`
