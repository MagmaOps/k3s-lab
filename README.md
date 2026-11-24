# k3s-lab

This is my homelab Kubernetes cluster, managed fully through Git. It's a small 5-node k3s cluster running on three mini PCs under my desk, and everything on it (platform stuff, namespaces, my own apps) is defined in this repo and synced by Argo CD. I don't `kubectl apply` things by hand anymore. If it's not in Git, it's not in the cluster.

I started this to learn Kubernetes properly, not just tutorial-level, and it slowly grew into a real platform that runs my side projects. I keep it public so anyone curious can see how I set things up, including the rough edges.

## What's in here

- **Argo CD + app-of-apps.** One root app bootstraps everything else, with sync waves so things come up in the right order. Auto-sync, prune and self-heal are on.
- **Kustomize bases and overlays.** Each app has a base plus a `prod` overlay with small patches for resources, ingress, zone affinity, and so on.
- **Apps own their manifests.** Every app repo ships its own `k8s/base`. This repo just pulls it in as a remote base and adds the homelab-specific bits on top.
- **Sealed Secrets.** Secrets live in Git, but only the encrypted versions.
- **CloudNativePG** for Postgres, with replicas spread across all three physical hosts.
- **Argo Rollouts** for progressive delivery.
- **Self-hosted GitHub Actions runners** (ARC) running inside the cluster itself.
- **PR checks.** Every PR renders all the overlays and runs them through kubeconform before I can merge. If it breaks, I get a Telegram message.
- **Monitoring** with Prometheus, Grafana (dashboards as code) and Alertmanager, which also pings Telegram.
- **MetalLB + ingress-nginx** so services get real IPs on my home network.

## The hardware

```
Proxmox-1 (192.168.1.10)     Proxmox-2 (192.168.1.11)     Proxmox-3 (192.168.1.12)
 ├─ k3s-cp (control plane)    ├─ k3s-w2 (worker)           └─ k3s-w4 (worker)
 └─ k3s-w1 (worker)           └─ k3s-w3 (worker)
```

| Host | Machine | CPU | RAM | Disk |
|------|---------|-----|-----|------|
| Proxmox-1 | Fujitsu Esprimo Q556/2 | i5-6400T | 16 GB | 256 GB SSD |
| Proxmox-2 | Fujitsu Esprimo Q556/2 | i5-6400T | 16 GB | 256 GB SSD |
| Proxmox-3 | HP EliteDesk 800 G4 DM | i7-8700T | 16 GB | 256 GB SSD |

Three mini PCs running Proxmox, connected through a TP-Link managed switch, with five Ubuntu VMs on top forming the k3s cluster. I treat each Proxmox host as its own "zone", so if one box goes down, the replicas on the others keep things running.

## How the GitOps part fits together

```
bootstrap/
 ├─ namespaces-app     → namespaces/
 ├─ platform-app       → platform/environments/prod
 ├─ loadbalancer-app   → platform/metallb
 └─ apps-app           → apps/environments/prod
                           └─ one Application per app
                                └─ workload/prod → remote base from the app repo + patches
```

## Folder layout

| Path | What's there |
|------|--------------|
| [`bootstrap/`](bootstrap) | The root Argo CD app and the app-of-apps entry points |
| [`namespaces/`](namespaces) | Namespaces, synced first |
| [`platform/`](platform) | Cluster-level stuff: Argo CD, ingress-nginx, MetalLB, Sealed Secrets, CloudNativePG, Argo Rollouts, Reloader, monitoring |
| [`apps/`](apps) | One folder per app: `base/` (the Argo CD Application), `overlays/prod/`, and `workload/prod/` (what actually gets deployed) |
| [`.github/workflows/`](.github/workflows) | PR validation, which calls my shared CI repo |

## Platform pieces

| Component | Why it's there |
|-----------|----------------|
| [Argo CD](platform/argocd) | Does the actual GitOps syncing (plus an [ingress](platform/argocd-ingress) for the UI) |
| [ingress-nginx](platform/ingress-nginx) | Ingress controller, 2 replicas on different hosts |
| [MetalLB](platform/metallb) | Hands out LoadBalancer IPs, since there's no cloud provider here |
| [Sealed Secrets](platform/sealed-secrets) | Lets me commit secrets without leaking them |
| [CloudNativePG](platform/cloudnative-pg) | Runs HA Postgres for the apps that need a database |
| [Argo Rollouts](platform/argo-rollouts) | Canary / blue-green deployments |
| [Reloader](platform/reloader) | Restarts pods when their ConfigMap or Secret changes |
| [Monitoring](platform/monitoring) | Prometheus, Grafana and Alertmanager |

## My other repos used here

The apps and the CI live in their own repos under [MagmaOps](https://github.com/MagmaOps). This repo only cares about how they get deployed.

| Repo | What it is | Wired in at |
|------|-----------|-------------|
| [k3s-lab-ci](https://github.com/MagmaOps/k3s-lab-ci) | Shared GitHub Actions workflows: validation for this repo, plus build/publish pipelines for the apps | [`.github/workflows/main-pr.yml`](.github/workflows/main-pr.yml) |
| [url-shortener](https://github.com/MagmaOps/url-shortener) | A simple URL shortener (FastAPI + Postgres) with Prometheus metrics. My first app on the cluster | [`apps/url-shortener`](apps/url-shortener) |
| [job-search-lab](https://github.com/MagmaOps/job-search-lab) | Pulls remote DevOps job postings and uses AI to match them against my CV (FastAPI, Next.js, CloudNativePG) | [`apps/job-search-lab`](apps/job-search-lab) |
| [TaskMaster](https://github.com/MagmaOps/TaskMaster) | A task manager that prioritizes tasks by weight (TypeScript, HA Postgres) | [`apps/taskmaster`](apps/taskmaster) |
| [ai-dev-platform](https://github.com/MagmaOps/ai-dev-platform) | A throwaway Claude Code environment running as a pod. I give it a task and it works on a feature branch | [`apps/ai-dev-platform`](apps/ai-dev-platform) |
| [world-cup-ladder](https://github.com/MagmaOps/world-cup-ladder) | A small JavaScript web app, scaled down for now | [`apps/world-cup-ladder`](apps/world-cup-ladder) |

### How a new app gets added

1. The app repo has a `k8s/base` folder and builds its image with a workflow from `k3s-lab-ci`.
2. In this repo, `apps/<app>/base` holds the Argo CD Application, and `overlays/prod` tweaks it.
3. `apps/<app>/workload/prod/kustomization.yaml` pulls in the app's `k8s/base`, pins the image tag, and applies the prod patches (resources, ingress host, DB tuning, secrets).
4. To deploy a new version, I open a PR here that bumps the image tag. Once it's merged, Argo CD takes care of the rest.

## CI

PRs to `main` run the reusable workflow from [k3s-lab-ci](https://github.com/MagmaOps/k3s-lab-ci). It renders every overlay, checks the output with kubeconform, and sends me a Telegram message if something fails. The jobs run on the cluster's own runners ([`apps/arc`](apps/arc)), so the cluster checks its own config.

## Secrets

No plaintext secrets in Git, ever. The input files I feed to `kubeseal` end in `.plaintext.yaml` and are git-ignored, so only the encrypted `*-sealed.yaml` files get committed.

## Stack

k3s · Proxmox · Argo CD · Kustomize · Helm · Sealed Secrets · CloudNativePG · MetalLB · ingress-nginx · Argo Rollouts · Prometheus · Grafana · GitHub Actions · ARC

## A note

This is a personal lab, not production. Hostnames are internal (`*.lab.local`), and I usually pick what teaches me the most over what's most hardened. Some things are still marked `TEMP` while I figure them out. If you spot something odd or have ideas, feel free to open an issue.

## For AI crawlers and bots

Please don't scrape, index or use this repo or any of my other repos (including everything under [MagmaOps](https://github.com/MagmaOps)) to train AI or machine learning models. This content isn't meant to be used as AI training data. If you're a person reading this, go ahead and look around.
