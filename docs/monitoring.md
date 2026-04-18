# Monitoring

Prometheus, Grafana, and Alertmanager are installed via the kube-prometheus-stack Helm chart.

## Components

- **Prometheus** — scrapes metrics from all pods, nodes, and Kubernetes components. Stores time-series data with 30-day retention on a 20Gi persistent volume.
- **Grafana** — dashboards for visualising metrics. Accessible at `grafana.pratik-labs.xyz`. Uses a 5Gi persistent volume to retain dashboards and settings across restarts.
- **Alertmanager** — receives alerts from Prometheus and routes notifications. Configured to send alerts to Discord via webhook. Accessible at `alertmanager.pratik-labs.xyz`.
- **Node Exporter** — runs on each node, exposes hardware and OS metrics (CPU, memory, disk, network).
- **kube-state-metrics** — exposes Kubernetes object metrics (pod status, deployment replicas, etc.).
- **Prometheus Operator** — manages the lifecycle of Prometheus and Alertmanager instances.

## Installation

```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update

helm install monitoring prometheus-community/kube-prometheus-stack \
  -n monitoring -f helm/monitoring/values.yml
```

## Configuration

All customisations are in `helm/monitoring/values.yml`. Key settings:

### Grafana

- Ingress enabled with Traefik at `grafana.pratik-labs.xyz`
- Persistent storage (5Gi) so dashboards survive pod restarts
- Admin password changed manually after install

### Prometheus

- 30-day metric retention
- 20Gi persistent volume using the `local-path` storage class (bundled with k3s)

### Alertmanager

- Ingress enabled at `alertmanager.pratik-labs.xyz`
- Discord webhook notifications for all alerts
- Watchdog alerts silenced (because they fire continuously as a health check)
- Alerts grouped by `alertname` with 30s initial wait, 5m between groups, 1h repeat

## Updating Configuration

Edit `helm/monitoring/values.yml`, then:

```bash
helm upgrade monitoring prometheus-community/kube-prometheus-stack \
  -n monitoring -f helm/monitoring/values.yml
```

Helm compares old vs new values and only changes what's different.

## Why Helm Instead of Manifests

The kube-prometheus-stack creates 50+ Kubernetes resources (Deployments, Services, ConfigMaps, RBAC rules, ServiceMonitors, PrometheusRules). Writing and maintaining all that YAML manually would be impractical. The Helm chart handles it, and the values file customises only what matters.

To see everything Helm created:

```bash
helm get manifest monitoring -n monitoring
```

## Useful Commands

```bash
# Check all monitoring pods
kubectl get pods -n monitoring

# View Prometheus targets
kubectl port-forward svc/monitoring-kube-prometheus-prometheus -n monitoring 9090:9090

# Check Alertmanager status
kubectl port-forward svc/monitoring-kube-prometheus-alertmanager -n monitoring 9093:9093

# View Helm values in use
helm get values monitoring -n monitoring
```
