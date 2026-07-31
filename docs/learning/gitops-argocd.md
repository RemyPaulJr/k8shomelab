# GitOps and ArgoCD

> **TL;DR:** Your GitHub repo is the single source of truth for the cluster. ArgoCD watches the repo and automatically makes the cluster match it — no SSH-ing into servers, no manual kubectl. This doc explains how, and why it matters to us.

## What is GitOps?

GitOps means **your Git repo is the source of truth** for everything in your cluster.

Instead of SSH-ing into a server and running commands, you:

1. Write YAML files describing what you want (apps, configs, etc.)
2. Push them to GitHub
3. A tool (ArgoCD) sees the changes and applies them to the cluster automatically

**Think of it like cooking from a recipe book:**

- The **Git repo** is the recipe book — it has the "recipe" (YAML) for every dish (app) in the restaurant.
- The **cluster** is the kitchen — where the cooking actually happens.
- **ArgoCD** is the chef's assistant whose only job is: "keep the kitchen exactly matching the recipe book, forever."

If someone breaks something in the cluster, you don't fix it on the server — you fix the YAML file in Git, push, and ArgoCD syncs it back. The cluster always matches the repo.

## What does ArgoCD do?

ArgoCD is a **Kubernetes app that lives inside your cluster**. It:

- Watches your GitHub repo
- Compares what's in the repo to what's running in the cluster
- If they don't match (someone made a change, you pushed an update), it syncs them

**"Sync" just means:** make the cluster match the repo. ArgoCD checks the repo every ~3 minutes (or when you tell it to) and re-applies anything that changed.

## Why not just use kubectl?

- `kubectl apply -f file.yaml` works, but only you ran it. No history, no audit trail, no automation.
- With GitOps, every change is in Git — you can see who changed what, when, and roll back easily.
- If a node dies and gets rebuilt, ArgoCD re-deploys everything automatically. No manual steps.

## The app-of-apps pattern

This is how our cluster is organized:

```
root-app.yaml
  ├── cert-manager.yaml   (creates an App that manages cert-manager)
  ├── ingress-nginx.yaml  (creates an App that manages nginx)
  ├── cloudflare-tunnel.yaml (creates an App that manages the tunnel)
  └── argocd-ingress.yaml (creates an App that manages the ArgoCD UI)
```

**Without app-of-apps:** You'd have to `kubectl apply` each of those 4 YAML files by hand. Add a new app? Run another kubectl command.

**With app-of-apps:** You apply `root-app.yaml` once. The root app watches the `kubernetes/argocd/apps/` folder. When you add a new YAML file there, the root app picks it up and creates the child app automatically.

Each child YAML (e.g. `cert-manager.yaml`) is itself an ArgoCD Application that points to its own folder of manifests or a Helm chart.

**Why "app-of-apps"?** Because one ArgoCD Application *contains* a list of other Applications. It's apps all the way down — one command to bootstrap everything, and one file to add anything new.

## How we bootstrapped it

You can't use ArgoCD to install ArgoCD (chicken-and-egg problem). So we split it into 2 steps:

1. **Step 1 — Manual install:** Run `kubectl apply -f install.yaml` to install ArgoCD into the cluster. This gets ArgoCD running.
2. **Step 2 — Root app:** Run `kubectl apply -f root-app.yaml` to tell ArgoCD "watch this repo and manage yourself from now on."

After step 2, ArgoCD manages its own config. If someone breaks it, you fix the YAML in Git, push, and ArgoCD fixes itself.

## How ArgoCD keeps things in line (our actual settings)

Our root app (`kubernetes/argocd/bootstrap/root-app.yaml`) has three settings that matter:

```yaml
syncPolicy:
  automated:
    prune: true
    selfHeal: true
```

- **`automated`** — ArgoCD syncs on its own. You don't have to click "Sync" every time something changes.
- **`selfHeal: true`** — If someone manually changes something in the cluster (e.g. `kubectl edit` a deployment), ArgoCD notices the cluster no longer matches Git and **reverts it**. Manual edits don't stick.
- **`prune: true`** — If a resource is removed from Git, ArgoCD **deletes it** from the cluster. No orphaned resources left behind from deleted YAMLs.

Together they mean: **Git is the only way to make a permanent change.** Anything else gets undone.

## How new apps get deployed

1. Write the app YAMLs (Deployment, Service, Ingress, etc.) in `kubernetes/apps/<app-name>/` — infrastructure pieces (cert-manager, ingress controller, tunnel) live in `kubernetes/infrastructure/`
2. Write an ArgoCD Application YAML in `kubernetes/argocd/apps/` pointing to that folder
3. Push to GitHub
4. Root app syncs (every 3 minutes or manually) and sees the new Application YAML
5. Root app creates the child Application
6. The child Application applies your manifests to the cluster

Everything is in Git. Nothing is lost.

## Key point

**The repo is reality.** The cluster is just a copy. You never SSH in and edit a file. You never run `kubectl` to make a permanent change. You edit the repo, push, and ArgoCD does the rest.

## Terms

| Term | Plain-language meaning |
|------|------------------------|
| **GitOps** | Using a Git repo as the single source of truth for a cluster's state |
| **Sync** | ArgoCD making the cluster match the repo |
| **App-of-apps** | One ArgoCD Application that manages a list of other Applications |
| **Child app** | An ArgoCD Application created by the root app, managing one component |
| **Drift** | When the cluster's actual state differs from the repo |
| **selfHeal** | ArgoCD reverts any manual cluster changes back to what Git says |
| **prune** | ArgoCD deletes cluster resources that were removed from Git |
| **Helm chart** | A packaged collection of Kubernetes manifests with configurable options |

## Related docs

- [Cloudflare Tunnel](cloudflare-tunnel.md) — how the ArgoCD UI is exposed externally
- [TLS: Per-Service Certificates](tls-per-ingress.md) — how the ArgoCD web UI gets its HTTPS cert
