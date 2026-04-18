# k8s-homelab-platform

A single-node Kubernetes homelab running on bare metal, built from scratch with infrastructure as code. Every component is version-controlled - no snowflake configurations.

## Tech Stack

- **OS:** Ubuntu 24.04 LTS Server
- **Kubernetes:** k3s
- **Ingress:** Traefik
- **Tunnel:** Cloudflare Tunnel
- **Monitoring:** Prometheus, Grafana, Alertmanager (via Helm)
- **GitOps:** Argo CD
- **CI/CD:** GitHub Actions → GHCR → Argo CD
- **Configuration Management:** Ansible

## Repository Structure

```
.
├── ansible
│   ├── inventory.ini
│   ├── k3s-server.yml
│   ├── kubeconfig.yml
│   └── setup.yml
├── docs
│   ├── decisions.md
│   ├── gitops.md
│   ├── monitoring.md
│   ├── networking.md
│   └── setup.md
├── helm
│   ├── argocd
│   │   └── values.yml
│   └── monitoring
│       └── values.yml
├── manifests
│   ├── apps
│   │   ├── demo
│   │   │   ├── deployment.yml
│   │   │   ├── ingress.yml
│   │   │   └── service.yml
│   │   └── portfolio
│   │       ├── deployment.yml
│   │       ├── ingress.yml
│   │       └── service.yml
│   ├── namespaces
│   │   └── namespaces.yml
│   └── tunnel
│       └── deployment.yml
└── README.md
```

## Deployed Services

| Service | URL | Managed By |
|---------|-----|------------|
| Portfolio | portfolio.pratik-labs.xyz | Argo CD |
| Demo | demo.pratik-labs.xyz | Argo CD |
| Grafana | grafana.pratik-labs.xyz | Helm |
| Alertmanager | alertmanager.pratik-labs.xyz | Helm |
| Argo CD | argocd.pratik-labs.xyz | Helm |

## CI/CD Pipeline

```
Push code → GitHub Actions (build + push image) → Update manifest → Argo CD deploys
```

No manual intervention. Push code, get a deployment.

## Documentation

- **[Setup Guide](docs/setup.md)** — Server setup, k3s installation, and initial configuration
- **[Networking](docs/networking.md)** — Cloudflare Tunnel, Traefik ingress, and traffic flow
- **[Monitoring](docs/monitoring.md)** — Prometheus, Grafana, and Alertmanager configuration
- **[GitOps & CI/CD](docs/gitops.md)** — Argo CD setup and automated deployment pipeline
- **[Decisions](docs/decisions.md)** — Why k3s, why Traefik, and other architectural choices
