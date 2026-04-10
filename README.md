# k8shomelab
Source of truth for my homelab running Kubernetes (k3s).

# Cluster
Control Node is a HP Elitedesk G3 800 Mini PC with 64gb of RAM and a 256gb SSD.

Worker Node is a Qotom Mini PC with 8gb of RAM.

Both are running Debian 13 for the OS and Lightweight Kubernetes k3s for the container orchestration.

# Applications
- GitOps: ArgoCD
    - I want to entirely do all of my work from my main pc or laptop using k9s. And this repo as the source of truth. So anything I deploy or change I want my cluster to point here for gitops.
- Local LLM
    - This one is pending as I'm not sure if it's worth without having a GPU.
- Grafana
    - Definitely want some sort of way to collect cluster and application metric, and visualize them. Monitoring and logging will be implemented early on.
