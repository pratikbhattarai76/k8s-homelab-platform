# GitOps & CI/CD

Argo CD watches this Git repository and automatically deploys changes to the cluster. Combined with GitHub Actions, pushing code to the portfolio repo triggers a full build-and-deploy pipeline with no manual steps.

## Argo CD

### What It Does

Argo CD continuously compares the desired state (YAML manifests in Git) with the actual state (what's running in the cluster). When they differ, it syncs — applying the changes to make the cluster match Git.

This means Git is the single source of truth. No one runs `kubectl apply` manually for deployments. Every change goes through Git, giving a full audit trail of who changed what, when, and why.

### Installation

```bash
helm repo add argo https://argoproj.github.io/argo-helm
helm repo update

helm install argocd argo/argo-cd -n argocd -f helm/argocd/values.yml
```

The values file configures:

- Traefik ingress at `argocd.pratik-labs.xyz`
- Insecure mode (HTTP, not HTTPS) since TLS terminates at Cloudflare

### Getting the Admin Password

```bash
kubectl get secret argocd-initial-admin-secret -n argocd -o jsonpath='{.data.password}' | base64 -d
```

Username is `admin`.

### Managed Applications

Argo CD manages these applications, all with auto-sync enabled:

| Application | Path | Namespace |
|-------------|------|-----------|
| demo | `manifests/apps/demo` | apps |
| portfolio | `manifests/apps/portfolio` | apps |
| namespaces | `manifests/namespaces` | default |
| tunnel | `manifests/tunnel` | tunnel |

The following are managed manually via Helm, not through Argo CD:

- Monitoring stack (kube-prometheus-stack)
- Argo CD itself
- Immich

### How Argo CD Stores Data

Argo CD doesn't need persistent volumes. Application definitions are stored as Kubernetes CRDs (in etcd, part of k3s). Redis is used as a UI cache only — if it restarts, it rebuilds from cluster state. Nothing is lost.

## CI/CD Pipeline

The portfolio app has a fully automated deployment pipeline across two repositories:

### Flow

```
1. Push code to portfolio-application-deployment repo
                    |
                    ▼
2. GitHub Actions: verify
   - Lint, type check, build verification
                    |
                    ▼
3. GitHub Actions: build-and-push
   - Build Docker image
   - Push to ghcr.io/pratikbhattarai76/portfolio-app
   - Tags: latest + commit-<sha>
                    |
                    ▼
4. GitHub Actions: update-deployment
   - Clone this repo (k8s-homelab-platform)
   - Update image tag in manifests/apps/portfolio/deployment.yml
   - Commit and push
                    |
                    ▼
5. Argo CD detects the YAML change
   - Polls repo every 3 minutes
   - Syncs: pulls new image, restarts pods
                    |
                    ▼
6. New version is live
```

### How the Image Tag Update Works

The CI workflow uses `sed` to find and replace the image tag in the deployment manifest:

```yaml
- name: Update image tag
  run: |
    sed -i "s|image: ghcr.io/pratikbhattarai76/portfolio-app:.*|image: ghcr.io/pratikbhattarai76/portfolio-app:commit-${GITHUB_SHA::7}|" manifests/apps/portfolio/deployment.yml
```

This changes the image line from whatever the current tag is to the new commit SHA. Argo CD sees the YAML changed and deploys.

### Authentication

The portfolio repo's GitHub Actions workflow needs permission to push to this repo. This is handled by a GitHub Personal Access Token (classic) with `repo` scope, stored as a repository secret called `PORTFOLIO_K8S_SERVER_DEPLOYMENT` in the portfolio repo.

### Commit Attribution

Automated commits appear as `github-actions` in the Git history, making it easy to distinguish automated deploys from manual changes.

## Adding a New App with CI/CD

1. Create the app's manifests in `manifests/apps/<app-name>/` (deployment, service, ingress)
2. Create an Argo CD Application pointing to that path (via UI or manifest)
3. In the app's CI workflow, add a job that updates the image tag in this repo after building
4. Add a Cloudflare public hostname for the subdomain

## kubectl Usage

With GitOps, `kubectl` is used for inspection and debugging only:

```bash
# View - check what's running
kubectl get pods -n <namespace>
kubectl get all -n <namespace>
kubectl logs deployment/<name> -n <namespace>

# Debug - investigate issues
kubectl describe pod <name> -n <namespace>
kubectl exec -it deployment/<name> -n <namespace> -- sh

# Never for deploying - that goes through Git
```
