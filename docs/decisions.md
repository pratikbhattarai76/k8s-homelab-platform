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
