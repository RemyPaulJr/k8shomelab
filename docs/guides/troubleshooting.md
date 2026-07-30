# Troubleshooting Guide

*Add entries here as issues are encountered and resolved.*

## Common Issues

### Worker Node Not Joining

**Symptom:** Worker shows `NotReady` or never appears in `k3s kubectl get nodes`.

**Check:**
```bash
# On worker, check k3s agent logs
journalctl -u k3s-agent -n 50 --no-pager

# Verify master is reachable from worker
curl -k https://<master-ip>:6443

# Verify node token is correct
cat /var/lib/rancher/k3s/server/node-token
```

**Resolution:** Re-run the Ansible playbook with `--ask-vault-pass`.

### Pod Stuck in Pending

**Symptom:** Pod won't schedule.

**Check:**
```bash
k3s kubectl describe pod -n <ns> <pod>
```

**Common causes:**
- Insufficient resources (8GB worker is tight — check `k3s kubectl describe node worker_node1`)
- No storage class available (needs Longhorn or local-path-provisioner)
- Wrong node selector / tolerations

### ArgoCD Root App Shows "Unknown" or "ComparisonError"

**Symptom:** Root Application shows `Sync Status: Unknown` or `ComparisonError` with errors like:
```
failed to list resources: ipaddresses.networking.k8s.io is forbidden
failed to list resources: nodes is forbidden
failed to list resources: challenges.acme.cert-manager.io is forbidden
```

**Cause:** The upstream ArgoCD install manifest targets the `argocd` namespace, but we deployed into `argo-cd`. The ClusterRoleBindings reference a ServiceAccount in `argocd` that doesn't exist, so the ArgoCD application controller has no RBAC permissions.

**Resolution:** Recreate the ClusterRoleBindings with the correct namespace:
```bash
# Fix the main ClusterRoleBindings to point to argo-cd namespace
kubectl delete clusterrolebinding argocd-application-controller
kubectl create clusterrolebinding argocd-application-controller \
  --clusterrole=argocd-application-controller \
  --serviceaccount=argo-cd:argocd-application-controller

kubectl delete clusterrolebinding argocd-server
kubectl create clusterrolebinding argocd-server \
  --clusterrole=argocd-server \
  --serviceaccount=argo-cd:argocd-server

kubectl delete clusterrolebinding argocd-applicationset-controller
kubectl create clusterrolebinding argocd-applicationset-controller \
  --clusterrole=argocd-applicationset-controller \
  --serviceaccount=argo-cd:argocd-applicationset-controller
```

Then delete and recreate the root Application to clear the cache:
```bash
kubectl delete application root -n argo-cd
kubectl apply -f https://raw.githubusercontent.com/RemyPaulJr/k8shomelab/main/kubernetes/argocd/bootstrap/root-app.yaml
```

### ArgoCD Install Fails with "CRD annotation too long"

**Symptom:** `kubectl apply -f install.yaml` fails with `metadata.annotations: Too long: may not be more than 262144 bytes`.

**Cause:** The ArgoCD ApplicationSet CRD contains large annotations that exceed the Kubernetes annotation size limit when applied with client-side apply. This is a known issue with newer ArgoCD versions on certain Kubernetes distributions (k3s included).

**Resolution:** Use server-side apply with force-conflicts instead:
```bash
kubectl apply --server-side --force-conflicts -n argo-cd \
  -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

### Certificate Errors

**Symptom:** Apps show TLS errors after cert-manager setup.

**Check:**
```bash
k3s kubectl get certificaterequests -A
k3s kubectl describe order -A
k3s kubectl logs -n cert-manager deploy/cert-manager
```

**Resolution:** Verify DNS records in Cloudflare and the ClusterIssuer configuration.
