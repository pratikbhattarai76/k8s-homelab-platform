# Architectural Decisions

Why things are the way they are.

## k3s over kubeadm

k3s is a fully compatible Kubernetes distribution packaged as a single binary. It's not a toy or a simulator - everything learned on k3s transfers directly to production clusters like EKS, GKE, AKS.

kubeadm is the full Kubernetes install. It gives more control over individual components but requires significantly more resources, setup time, and ongoing maintenance. For a single-node homelab, that complexity adds no value.

k3s bundles Traefik, CoreDNS, containerd, and a local storage provider out of the box. With kubeadm, you'd install and configure each of these separately.

Adding a worker node later is also a single command with k3s.

## Traefik over Nginx Ingress

Traefik comes bundled with k3s. Installing Nginx Ingress Controller would mean disabling Traefik and managing a separate component for no practical benefit at this scale.

Both are production-grade ingress controllers. Traefik is lighter and integrates well with k3s. Nginx Ingress is more common in enterprise environments but functionally equivalent for this use case.

## Cloudflare Tunnel over Port Forwarding

The server is behind CGNAT, so port forwarding isn't possible. Even if it were, Cloudflare Tunnel is the better approach:

- No open ports on the home network
- Built-in DDoS protection via Cloudflare
- Free TLS certificates managed by Cloudflare
- Works from any network regardless of NAT type

## Helm for Complex Apps, Manifests for Simple Apps

Simple apps (demo, portfolio, tunnel) use plain Kubernetes manifests written by hand. You can read every line and understand exactly what's deployed.

Complex apps (Prometheus stack, Argo CD) use Helm. The kube-prometheus-stack alone creates 50+ resources. Writing and maintaining that YAML manually would be impractical and error-prone.

The values file is the only thing to customise and Helm handles the rest.

## GitOps over Manual kubectl

Every deployment goes through Git. Benefits:

- Full audit trail - every change is a commit with author, timestamp, and message
- Easy rollbacks - revert a commit and Argo CD syncs
- No "someone ran kubectl and nobody knows what changed" situations
- Reproducible - clone the repo and rebuild the entire cluster

The only exception is the monitoring stack, which is still managed via manual `helm upgrade`. This is a practical choice for me - Argo CD can manage Helm apps but the configuration is more complex. It can though be migrated later.

## Secrets Out of Git

Sensitive values (SMTP passwords, tunnel tokens) are stored as Kubernetes Secrets, created manually via `kubectl create secret`. They never appear in committed YAML files.

This is the simplest approach for my homelab.

## Ansible for Server Setup

Ansible automates the boring stuff like installing packages, setting timezone, installing k3s. Without it, SSHing and running commands manually means creating a snowflake server that can't be reproduced.

Ansible is agentless (just SSH)

## SSD for Metadata, HDD for Photos

The server has two drives: a 120GB SSD and a 1TB HDD. The split is deliberate. PostgreSQL (metadata, face embeddings, vector indexes) sits on the SSD for fast queries. Photos, thumbnails, encoded videos, and uploads sit on the HDD because they're large but sequentially accessed. This keeps the SSD from filling up while giving the database the I/O performance it benefits from most.

## CloudNativePG over Bitnami Postgres

Immich requires PostgreSQL with the VectorChord extension for vector similarity search. The Immich Helm chart removed its bundled Bitnami Postgres subchart because it caused upgrade issues and didn't align well with the custom pgvecto.rs/VectorChord images.

CloudNativePG is a Kubernetes operator purpose-built for managing PostgreSQL lifecycle — it handles initialization, failover, and uses the correct `cloudnative-vectorchord` image with `vchord.so` preloaded. For a single-instance homelab setup it might seem like overkill, but it's the approach Immich officially recommends and it's what production clusters use.

## GPU in Kubernetes via NVIDIA Device Plugin

Immich's machine learning (face detection, OCR, smart search) benefits significantly from GPU acceleration. Rather than running ML outside the cluster, the GPU is exposed to Kubernetes through three layers: the NVIDIA driver on the host, the NVIDIA container toolkit for containerd, and the NVIDIA k8s device plugin that registers `nvidia.com/gpu` as a schedulable resource.

The ML pod requests the GPU via a resource limit and runs the CUDA variant of the Immich image. This keeps everything inside the cluster with no special-case processes running on the host.

## Tailscale for Remote kubectl Access

k3s generates TLS certificates for the API server. By default, these only cover `127.0.0.1` and the local IP. Adding the Tailscale MagicDNS hostname (`k8s-server.taile34db4.ts.net`) as a TLS SAN means kubectl works from anywhere on the Tailscale network without certificate errors.

Using the Tailscale hostname instead of the IP means the kubeconfig stays valid even if the Tailscale IP changes (e.g. after reinstalling Tailscale). MagicDNS always resolves to the current IP.

## External Library Separate from Immich Library

Existing photos are mounted at `/external` inside the Immich container, separate from Immich's managed `/data` directory. This avoids conflicts — Immich creates its own folder structure (library, thumbs, upload, encoded-video) under `/data`, and pointing an external library to the same path causes errors. Keeping them separate means Immich can scan existing photos without interfering with its own storage

## Bitnami PostgreSQL for Nextcloud over CloudNativePG

Immich uses CloudNativePG because it requires the VectorChord extension for vector similarity search — a custom PostgreSQL image that the CloudNativePG operator manages. Nextcloud has no such requirement. It works with standard PostgreSQL.

The Nextcloud Helm chart bundles the Bitnami PostgreSQL subchart, which handles deployment, initialization, and persistence automatically. Using CloudNativePG here would add operator complexity for no benefit. Different tools for different requirements.

## Nextcloud External Storage over Direct Mount

Nextcloud's application data (config, app state) lives on the SSD via `local-path`. The HDD is mounted separately at `/external/storage` and surfaced through the External Storage app.

This keeps Nextcloud's own data on fast storage while providing web access to the HDD. It also means Nextcloud doesn't "own" the HDD files — they remain accessible independently, and multiple apps (Immich, Nextcloud) can reference the same physical storage without conflict.
