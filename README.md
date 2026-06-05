# k3s-homelab

A production-style homelab Kubernetes cluster built on k3s, managed via GitOps with ArgoCD.

## Cluster Overview

- **Ingress:** Traefik (built-in k3s) with Cloudflare DNS-01 wildcard TLS
- **GitOps:** ArgoCD watching this repo and a private repo
- **CI/CD:** GitHub Actions for yaml validation
- **Secret management:** Kubernetes Secrets created manually, never stored in Git

## Stack

| App | Namespace | Description |
|-----|-----------|-------------|
| Homepage | `homepage` | Self-hosted dashboard |
| ntfy | `ntfy` | Push notification server |
| n8n | `n8n` | Workflow automation |
| ArgoCD | `argocd` | GitOps CD |
| Traefik | `kube-system` | Ingress controller + TLS |

## Repo Structure

```
k3s-homelab/
├── apps/
│   ├── homepage/        # Dashboard (base - private overlay in separate repo)
│   ├── ntfy/            # Notifications
│   ├── n8n/             # Automation (base - private overlay in separate repo)
│   └── argocd/          # ArgoCD ingress
├── infrastructure/
│   └── traefik/         # Traefik + Cloudflare TLS config
├── argocd-apps/         # ArgoCD Application definitions
└── .github/
    └── workflows/       # GitHub Actions CI validation
```

## Architecture

This repo uses **Kustomize** with a base/overlay pattern:
- Public repo contains base manifests with no sensitive data
- Private repo contains overlays that patch in IPs, credentials, and private config
- ArgoCD deploys from the private overlay, which references the public base

## Prerequisites

Create required secrets on the cluster before deploying. These are never stored in Git.

### Traefik / Cloudflare
```bash
kubectl create secret generic cloudflare-api-token \
  --namespace kube-system \
  --from-literal=api-token=<your-cloudflare-token>
```

### n8n
```bash
kubectl create secret generic n8n-secret \
  --namespace n8n \
  --from-literal=db-host=<postgres-host> \
  --from-literal=db-port=<postgres-port> \
  --from-literal=db-user=<postgres-user> \
  --from-literal=db-password=<postgres-password> \
  --from-literal=encryption-key=$(openssl rand -hex 32)
```

### Homepage
```bash
kubectl create secret generic homepage-secret \
  --namespace homepage \
  --from-literal=truenas-key=<your-truenas-key> \
  --from-literal=proxmox-password=<your-proxmox-password> \
  --from-literal=pihole-key=<your-pihole-key> \
  --from-literal=tailscale-key=<your-tailscale-key> \
  --from-literal=tailscale-device-id=<your-tailscale-device-id> \
  --from-literal=openweathermap-key=<your-openweathermap-key>
```

## Deployment

### 1. Install ArgoCD
```bash
kubectl create namespace argocd
kubectl apply -n argocd --server-side --force-conflicts \
  -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

### 2. Apply infrastructure
```bash
kubectl apply -f infrastructure/traefik/traefik-config.yaml
kubectl apply -f apps/argocd/argocd-ingress.yaml
```

### 3. Connect ArgoCD to repos and deploy apps
```bash
kubectl apply -f argocd-apps/
```
