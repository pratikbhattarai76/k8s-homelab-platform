# Immich

Self-hosted photo management with GPU-accelerated machine learning for face detection, object recognition, and smart search. Accessible at `photos.pratik-labs.xyz`.

## Architecture 

Immich runs as multiple components:

- **Immich Server** - web UI and API, handles uploads, browsing, sharing
- **Machine Learning** - face detection, OCR (Optical Character Recognition), smart search. Runs on GPU (GTX 1660 Super) using the CUDA image.
- **PostgreSQL** - metadata database with VectorChord extension for vector similarity search. Manager by CloudNativePG operator
- **Valkey** - in-memory cache (Redis replacement), bundled with the Helm chart 

## Storage Layout

Two storage tiers seperate performance-sensitive data from bulk storage:

- **SSD (128GB)** - PostgreSQL database. Stores photo metadata, face embeddings, search vectors. 
- **HDD(1TB)** - Photo library mounted at `/mnt/storage/Photos`. Stores original photos, thumbnails, encoded videos, uploads.

## Prerequisites

Before installing Immich, the following must be in place:

1. HDD mounted at `/mnt/storage` (handled by `ansible/setup.yml`)
2. NVIDIA drivers and container toolkit installed (handled by `ansible/gpu.yml`)
3. CloudNativePG operator installed
4. `immich` namespace created

## Installation

### 1. CloudNativePG Operator

Manages the PostgreSQL lifecycle. Installed once, used by any app that needs Postgres.

```bash
helm repo add cnpg https://cloudnative-pg.github.io/charts
helm repo update
helm install cnpg cnpg/cloudnative-pg -n cnpg-system --create-namespace
```

### 2. Persistent Volume for Photo Library

Creates a PV pointing to the HDD and a PVC that Immich claims:

```bash
kubectl apply -f manifests/apps/immich/pvc.yml
```

The PV uses `hostPath` with `Retain` reclaim policy — deleting the PVC won't delete photos.

### 3. PostgreSQL Cluster

Creates a single-instance PostgreSQL with the VectorChord extension for vector similarity search:

```bash
kubectl apply -f manifests/apps/immich/postgres.yml
```

This deploys a CloudNativePG `Cluster` resource using the `cloudnative-vectorchord` image with `vchord.so` preloaded. The database initializes with the required extensions (vchord, cube, earthdistance).

Wait for the postgres pod to be running before proceeding:

```bash
kubectl get pods -n immich -w
```

### 4. Install Immich via Helm

```bash
helm repo add immich https://immich-app.github.io/immich-charts
helm repo update
helm install immich immich/immich -n immich -f helm/immich/values.yml
```

### 5. Cloudflare Hostname

Add a public hostname in the Cloudflare tunnel dashboard:
- Subdomain: `photos`
- Domain: `pratik-labs.xyz`
- Service: `http://traefik.kube-system.svc.cluster.local:80`

## GPU Acceleration

The machine learning pod runs the CUDA variant of the Immich ML image (`v2.6.3-cuda`) and requests the GPU via the `nvidia` RuntimeClass and resource limits.

Relevant values in `helm/immich/values.yml`:

GPU status can be verified with:

```bash
ssh k8s-local "nvidia-smi"
```

The ML process should appear using GPU memory.

## External Libraries

Existing photos on the HDD are accessed via an external library, not uploaded through Immich. The `Pictures` folder is mounted separately at `/external` inside the container to avoid conflicts with Immich's managed `/data` directory.

To add folders as an external library:

1. Administration → External Libraries → Create Library
2. Set the owner (the user who should see the photos)
3. Add import path: `/external` (or specific subfolders like `/external/Dashain`)
4. Save and scan

Exclusion patterns can be added to hide specific folders from scanning (e.g. `**/Unsorted/**`).

Enable folder view under Account Settings → Features → Folders to browse by the original folder structure.

## Sharing Photos

Immich doesn't support direct folder sharing between users. Two approaches:

- **Albums** - Select photos from folders, create an album, share with specific users. One scan, multiple shares.
- **Per-user external libraries** - Create separate external libraries owned by different users, each pointing to specific folder paths. Each user only sees their own library.

## Updating Configuration

```bash
helm upgrade immich immich/immich -n immich -f helm/immich/values.yml
```

## Useful Commands

```bash
# Check all Immich pods
kubectl get pods -n immich

# Server logs
kubectl logs deployment/immich-server -n immich -f

# ML logs (watch GPU processing)
kubectl logs <ml-pod-name> -n immich -f

# Check GPU usage
ssh k8s-local "nvidia-smi"

# PostgreSQL status
kubectl get cluster -n immich
```

