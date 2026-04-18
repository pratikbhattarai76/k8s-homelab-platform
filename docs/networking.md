# Networking

How traffic gets from the internet to pods, and how each networking component fits together.

## Traffic Flow

```
User visits portfolio.pratik-labs.xyz
    |
    ▼
Cloudflare DNS → resolves to Cloudflare edge servers
    |
    ▼
Cloudflare Tunnel → encrypted connection to cloudflared pod in cluster
    |
    ▼
cloudflared → forwards to http://traefik.kube-system.svc.cluster.local:80
    |
    ▼
Traefik → reads Host header, matches Ingress rule
    |
    ▼
Service → load balances across pod replicas
    |
    ▼
Pod → handles the request
```

## Cloudflare Tunnel

The server is behind CGNAT (Carrier-Grade NAT), meaning it doesn't have a real public IP and port forwarding won't work. Cloudflare Tunnel solves this by creating an outbound encrypted connection from the server to Cloudflare, so no open ports needed.

### How It Works

The `cloudflared` pod runs inside the cluster and maintains a persistent outbound connection to Cloudflare's edge network. When someone visits `portfolio.pratik-labs.xyz`, Cloudflare routes the request through the tunnel to the `cloudflared` pod, which forwards it to Traefik.

### Setup

1. Create a tunnel in the Cloudflare dashboard (Zero Trust → Networks → Tunnels)
2. Store the tunnel token as a Kubernetes Secret:

```bash
kubectl create secret generic cloudflared-token \
  --from-literal=token=<YOUR_TUNNEL_TOKEN> -n tunnel
```

3. Deploy cloudflared:

```bash
kubectl apply -f manifests/tunnel/deployment.yml
```

4. In the Cloudflare dashboard, add public hostnames for each service, all pointing to the same URL:

```
http://traefik.kube-system.svc.cluster.local:80
```

Traefik handles the routing from there based on the hostname.

### Adding a New Subdomain

For every new app you want to expose:

1. Create an Ingress manifest with the hostname (e.g. `myapp.pratik-labs.xyz`)
2. Add a public hostname in the Cloudflare tunnel dashboard pointing to `http://traefik.kube-system.svc.cluster.local:80`

The Cloudflare tunnel URL never changes. Only the subdomain and Ingress rule change per app.

## Traefik Ingress Controller

Traefik comes bundled with k3s and acts as the cluster's ingress controller. It watches for Ingress resources and routes incoming HTTP traffic based on the hostname.

### How Routing Works

When Traefik receives a request, it reads the `Host` header and matches it against all Ingress rules in the cluster. Each Ingress specifies which hostname maps to which Service:

```yaml
spec:
  ingressClassName: traefik
  rules:
    - host: portfolio.pratik-labs.xyz    # Match this hostname
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: portfolio          # Forward to this Service
                port:
                  number: 80             # On this port
```

If no Ingress rule matches the hostname, Traefik returns a 404.

### ingressClassName

Every Ingress should specify `ingressClassName: traefik` to explicitly declare which ingress controller handles it. With only Traefik installed it works without it (Traefik is the default), but it's good practice for clarity and to avoid issues if you add another ingress controller later (like Nginx).

## Services

Services provide stable internal addresses for pods. The cluster uses ClusterIP services (internal only) since external access is handled by Traefik + Cloudflare Tunnel.

```yaml
spec:
  selector:
    app: portfolio       # Find pods with this label
  ports:
    - port: 80           # Service listens on port 80
      targetPort: 4321   # Forwards to pod's port 4321
```

The Service port and the Ingress backend port must match. The Service's targetPort must match the container's port.

