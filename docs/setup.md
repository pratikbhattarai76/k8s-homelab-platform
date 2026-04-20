# Setup Guide

Step-by-step guide to setting up the homelab from a fresh Ubuntu install to a running k3s cluster.

## Prerequisites

- A machine dedicated as the server (desktop, mini PC, etc.)
- A dev machine (laptop/desktop) to manage the cluster from
- SSH key pair generated on the dev machine
- Ubuntu 24.04 LTS Server ISO on a bootable USB

## 1. Install Ubuntu Server

Flash Ubuntu 24.04 LTS Server ISO to a USB drive. During installation:

- Set a static IP address (e.g. `192.168.1.100`) so the server always has the same address on your network
- Enable OpenSSH server
- Create a user (e.g. `pratikserver`)

After installation, reboot and remove the USB.

## 2. SSH Access

From your dev machine, copy your SSH key to the server:

```bash
ssh-copy-id pratikserver@192.168.1.100
```

Add an SSH config entry for convenience. In `~/.ssh/config`:

```
Host homelab
    HostName 192.168.1.100
    User pratikserver
    IdentityFile ~/.ssh/id_ed25519
```

Now you can just run `ssh homelab` instead of typing the full address.

## 3. Ansible Inventory

The inventory file (`ansible/inventory.ini`) tells Ansible how to reach the server:

```ini
[control_plane]
local ansible_host=192.168.1.100 ansible_user=pratikserver ansible_ssh_private_key_file=~/.ssh/id_ed25519
```

Test connectivity:

```bash
ansible -i ansible/inventory.ini control_plane -m ping
```

Should return `pong`.

## 4. Base Server Setup

The `setup.yml` playbook handles post-install configuration:

- Updates and upgrades all packages
- Sets timezone to Asia/Kathmandu
- Installs basic dependencies (curl, git, wget, apt-transport-https)

```bash
ansible-playbook -i ansible/inventory.ini ansible/setup.yml
```

## 5. Install k3s

The `k3s-server.yml` playbook installs k3s on the server:

- Downloads and installs k3s via the official install script
- Waits for k3s to be ready
- Enables the k3s systemd service

```bash
ansible-playbook -i ansible/inventory.ini ansible/k3s-server.yml
```

k3s bundles everything needed: kubectl, container runtime (containerd), CoreDNS, Traefik ingress controller, and a local storage provider.

## 6. Get Kubeconfig

The `kubeconfig.yml` playbook fetches the kubeconfig from the server to your dev machine so you can run `kubectl` commands locally:

```bash
ansible-playbook -i ansible/inventory.ini ansible/kubeconfig.yml
```

This copies `/etc/rancher/k3s/k3s.yaml` from the server to `~/.kube/config` on your dev machine. After this, you need to update the server address in the kubeconfig from `127.0.0.1` to `192.168.1.100`:

```bash
sed -i 's/127.0.0.1/192.168.1.100/' ~/.kube/config
```

Verify it works:

```bash
kubectl get nodes
```

## 7. Create Namespaces

Namespaces provide logical separation inside the cluster:

```bash
kubectl apply -f manifests/namespaces/namespaces.yml
```

This creates four namespaces:

- `apps` — application workloads (demo, portfolio)
- `monitoring` — Prometheus, Grafana, Alertmanager
- `tunnel` — cloudflared tunnel
- `argocd` — Argo CD GitOps controller
- `immich` - Immich photo management

## Next Steps

With the cluster running and namespaces created, proceed to:

- [Networking](networking.md) - Set up Cloudflare Tunnel and deploy apps
- [Monitoring](monitoring.md) - Install the monitoring stack
- [GitOps & CI/CD](gitops.md) - Set up Argo CD and automated deployments
- [Immich](immich.md) - Deploy photo management

## GPU Setup (Optional)

If the server has an NVIDIA GPU, the `gpu.yml` playbook installs the driver and container toolkit:

```bash
ansible-playbook -i ansible/inventory.ini ansible/gpu.yml --ask-become-pass
```

After the playbook completes, reboot the server:

```bash
ssh k8s-local "sudo reboot"
```

Verify the GPU is detected:

```bash
ssh k8s-local "nvidia-smi"
```

The NVIDIA device plugin also needs to be installed in the cluster and the RuntimeClass created. See [Immich docs](immich.md) for the full GPU setup used by Immich's machine learning.

## Remote Access via Tailscale

k3s is installed with TLS SANs for both the local IP and the Tailscale MagicDNS hostname:

```
--tls-san k8s-server.taile34db4.ts.net --tls-san 192.168.1.100
```

Update the kubeconfig to use the Tailscale hostname so kubectl works from anywhere:

```bash
sed -i 's|https://192.168.1.100:6443|https://k8s-server.taile34db4.ts.net:6443|' ~/.kube/config
```

This works from both the local network and remote locations. Tailscale automatically picks the fastest path.
