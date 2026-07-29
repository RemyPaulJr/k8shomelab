# Setup Guide

## Prerequisites

- Two Debian 13 machines with SSH access
- User with sudo privileges (remy)
- Ansible installed on your main machine
- Domain name registered and pointed to Cloudflare

## Step 1: OS Installation

Both nodes run Debian 13 (Trixie). Install with:
- Standard system utilities
- SSH server
- User `baki` with sudo access

## Step 2: Configure SSH

Ensure passwordless SSH from your main machine to both nodes, or use Ansible Vault passwords (current approach).

## Step 3: Run Ansible Bootstrap

```bash
cd k8shomelab
ansible-playbook -i ansible/inventory.ini \
  ansible/playbooks/k3s_master.yml \
  --ask-vault-pass
```

This installs k3s on the master, retrieves the node token, and joins the worker.

## Step 4: Verify Cluster

SSH into the master and check:

```bash
k3s kubectl get nodes
k3s kubectl get pods -A
```

## Step 5: Install k9s (Terminal UI)

k9s is the recommended way to manage the cluster day-to-day.

```bash
# Download and install (Linux/WSL2)
curl -LO https://github.com/derailed/k9s/releases/latest/download/k9s_Linux_amd64.tar.gz
tar -xzf k9s_Linux_amd64.tar.gz
sudo mv k9s /usr/local/bin/
rm k9s_Linux_amd64.tar.gz

# Launch
k9s
```

Key k9s commands:
- `:pods` → pod view, `:deploy` → deployments, `:svc` → services
- `Ctrl+n` → switch namespace
- `/` → filter / search
- `L` → tail logs on selected pod
- `S` → shell into selected pod
- `Shift+F` → port-forward on selected pod

## Step 6: Set up kubeconfig on Main PC

Copy the kubeconfig from master:

```bash
ssh baki@control_node "cat /etc/rancher/k3s/k3s.yaml" > ~/.kube/config
# Replace "127.0.0.1" with the master's IP
```

## Step 6: Bootstrap ArgoCD

```bash
# Install ArgoCD from upstream manifests
kubectl create ns argo-cd
kubectl apply -n argo-cd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

# Wait for all pods to be ready
kubectl wait --for=condition=Ready pods --all -n argo-cd

# Apply the root Application (app-of-apps)
kubectl apply -f kubernetes/argocd/bootstrap/root-app.yaml
```

See `docs/CLAUDE.md` for ADR 002 (reasoning behind raw manifest bootstrap).

## Next Steps

Proceed to the remaining Phase 2 (GitOps + Infrastructure) items: cert-manager, Cloudflare Tunnel, Longhorn, etc.
