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

### Cloudflare Tunnel Not Connecting

**Symptom:** cloudflared pods crash-loop or show connection errors.

**Check:**
```bash
k3s kubectl logs -n cloudflare-tunnel deploy/cloudflared
k3s kubectl describe pod -n cloudflare-tunnel
```

**Common causes:**
- Tunnel token secret (`cloudflare-tunnel-credentials`) not created in `cloudflare-tunnel` namespace
- Token is expired or revoked — re-run `cloudflared tunnel token <tunnel-name>` and update the Secret
- Network egress is blocked — verify the node can reach `api.cloudflare.com`
- Wrong secret key name — deployment expects `tunnel-token` key in the Secret

### ClusterIssuer Fails to Provision Certificate

**Symptom:** Certificate remains in `pending` state, Order shows `invalid`.

**Check:**
```bash
k3s kubectl describe certificaterequest -A
k3s kubectl describe order -A
k3s kubectl logs -n cert-manager deploy/cert-manager
```

**Common causes:**
- Cloudflare API token secret (`cloudflare-api-token`) not created in `cert-manager` namespace
- API token lacks `Zone:DNS:Edit` permission in Cloudflare
- DNS record for the domain does not exist or is not pointed to Cloudflare
- Let's Encrypt rate limit hit — use staging issuer for testing

### ArgoCD App-of-Apps — Child App Not Created

**Symptom:** After updating `root-app.yaml` to point to `kubernetes/argocd/apps/`, child Applications don't appear.

**Check:**
```bash
k3s kubectl get application -n argo-cd
k3s kubectl describe application root -n argo-cd
```

**Resolution:**
- Ensure child Application manifests are valid YAML
- Ensure the `metadata.namespace` is `argo-cd` in each child Application
- The root Application must have `directory.recurse: true` and point to the correct path
- Check ArgoCD logs: `k3s kubectl logs -n argo-cd deploy/argocd-application-controller`
