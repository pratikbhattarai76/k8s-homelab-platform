# Nextcloud

Self-hosted file storage and collaboration platform. Accessible at `nextcloud.pratik-labs.xyz`.

## Architecture

Nextcloud runs as two components deployed via the official Helm chart:

- **Nextcloud** — web UI, file management, sharing, and WebDAV. Runs with a cron sidecar for background jobs.
- **PostgreSQL** — metadata database, deployed via the Bitnami subchart bundled with the Nextcloud Helm chart.

## Storage Layout

Two storage tiers separate application data from bulk file storage:

- **SSD (local-path)** — Nextcloud application data (`/var/www/html`) and PostgreSQL database. Stores config, app state, and metadata.
- **HDD (1TB)** — Mounted at `/mnt/storage` on the host, exposed inside the container at `/external/storage`. Provides access to existing files on the HDD via the External Storage app.

## Prerequisites

Before installing Nextcloud, the following must be in place:

1. HDD mounted at `/mnt/storage` (handled by `ansible/setup.yml`)
2. `nextcloud` namespace created via `manifests/namespaces/namespaces.yml`
3. PV and PVC created via `manifests/apps/nextcloud/pvc.yml`
4. Secrets created for admin and database credentials

## Installation

### Secrets

Two secrets are required. Do not commit these to Git.

```bash
kubectl create secret generic nextcloud-admin-secret \
  --from-literal=username=admin \
  --from-literal=password=<admin-password> \
  -n nextcloud

kubectl create secret generic nextcloud-db-secret \
  --from-literal=db-username=nextcloud \
  --from-literal=db-password=<db-password> \
  -n nextcloud
```

### PVC for HDD Access

The PV/PVC in `manifests/apps/nextcloud/pvc.yml` creates a 500Gi PersistentVolume pointing to `/mnt/storage` on the host with `Retain` reclaim policy — deleting the PVC won't delete files on the HDD.

```bash
kubectl apply -f manifests/apps/nextcloud/pvc.yml
```

### Helm Install

```bash
helm repo add nextcloud https://nextcloud.github.io/helm/
helm repo update
helm install nextcloud nextcloud/nextcloud -n nextcloud -f helm/nextcloud/values.yml
```

### Cloudflare Hostname

Add a public hostname in the Cloudflare tunnel dashboard:

- Subdomain: `nextcloud`
- Domain: `pratik-labs.xyz`
- Service: `http://traefik.kube-system.svc.cluster.local:80`

## Reverse Proxy Configuration

Nextcloud sits behind Cloudflare Tunnel → Traefik → Apache. Without proper proxy configuration, this causes redirect loops and "untrusted domain" errors. The `proxy.config.php` in `helm/nextcloud/values.yml` handles this:

- `trusted_proxies` — tells Nextcloud to trust headers from the Traefik pod network (`10.0.0.0/8`)
- `overwriteprotocol: https` — Cloudflare terminates TLS, so traffic arrives as HTTP. This forces Nextcloud to generate `https://` URLs.
- `forwarded_for_headers` — reads the client's real IP from `X-Forwarded-For` set by Traefik

## External Storage (HDD)

The HDD at `/mnt/storage` is mounted into the Nextcloud container at `/external/storage` via `extraVolumes` and `extraVolumeMounts` in the Helm values. To browse these files through the Nextcloud UI:

1. Go to Administration Settings → Apps
2. Search for **External storage support** and enable it
3. Go to Administration Settings → External storage
4. Add a new mount: Folder name `Storage`, type `Local`, path `/external/storage`, available for all users

## Differences from Immich

Both Nextcloud and Immich access the HDD, but differently:

- **Immich** uses CloudNativePG for PostgreSQL (required for VectorChord extension). Nextcloud uses the Bitnami PostgreSQL subchart bundled with its Helm chart — simpler, no operator needed.
- **Immich** mounts `/mnt/storage/Photos` directly as its library PVC. Nextcloud mounts `/mnt/storage` via the External Storage app — it doesn't own the files, it just provides a web interface to browse them.

## Useful Commands

```bash
# Check pods
kubectl get pods -n nextcloud

# Nextcloud logs
kubectl logs deployment/nextcloud -n nextcloud -f

# PostgreSQL logs
kubectl logs nextcloud-postgresql-0 -n nextcloud -f

# Verify HDD mount
kubectl exec -it deployment/nextcloud -n nextcloud -- ls /external/storage

# Get admin password
kubectl get secret nextcloud-admin-secret -n nextcloud -o jsonpath="{.data.password}" | base64 -d
```
