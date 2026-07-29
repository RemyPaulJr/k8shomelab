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
- User `remy` with sudo access

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

## Step 5: Set up kubeconfig on Main PC

Copy the kubeconfig from master:

```bash
ssh remy@control_node "cat /etc/rancher/k3s/k3s.yaml" > ~/.kube/config
# Replace "127.0.0.1" with the master's IP
```

## Next Steps

Proceed to Phase 2 (GitOps + Infrastructure) once the cluster is verified.
