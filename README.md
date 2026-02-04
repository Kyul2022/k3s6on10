# Deploying kube-prometheus-stack on k3s

This guide walks through deploying the Prometheus monitoring stack on a k3s cluster.

## Prerequisites

- k3s cluster running
- kubectl configured to access the cluster
- Helm installed

## Steps

### 1. Add the Prometheus Community Helm Repository

```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update
```

### 2. Create the Namespace

```bash
kubectl create namespace k3s6on10
```

### 3. Create Grafana Admin Secret

Create a secret to store Grafana admin credentials:

```bash
kubectl create secret generic grafana-secret \
  --from-literal=admin-user=admin \
  --from-literal=admin-password=Hola1234 \
  -n k3s6on10
```

### 4. Create Custom Values File

Create a file called `myValues.yaml` with your customizations:

```yaml
grafana:
  admin:
    existingSecret: "grafana-secret"
  service:
    type: LoadBalancer
    port: 9010
prometheus:
  prometheusSpec:
    storageSpec:
      volumeClaimTemplate:
        spec:
          storageClassName: local-path
          accessModes: ["ReadWriteOnce"]
          resources:
            requests:
              storage: 2Gi
    
```

**Key configurations:**
- **Grafana service**: Exposed via LoadBalancer on port 9010
- **Grafana credentials**: Using the secret created in step 3
- **Prometheus storage**: Persistent volume with 50Gi storage

### 5. Install the Chart

```bash
helm install kube-prometheus-stack \
  prometheus-community/kube-prometheus-stack \
  -n k3s6on10 \
  -f myValues.yaml
```

### 6. Verify Deployment

Check that all pods are running:

```bash
kubectl get pods -n k3s6on10
```

Check services:

```bash
kubectl get svc -n k3s6on10
```

You should see the Grafana service with an external IP assigned.

### 7. Access Grafana

Get the Grafana external IP:

```bash
kubectl get svc kube-prometheus-stack-grafana -n k3s6on10
```

Access Grafana at: `http://<EXTERNAL-IP>:9010`

Login with:
- Username: `admin`
- Password: `Hola1234` (or whatever you set in the secret)

### 8. Access Prometheus UI (Optional)

To access Prometheus for debugging, use port-forward:

```bash
kubectl port-forward svc/kube-prometheus-stack-prometheus 9090:9090 -n k3s6on10
```

Then open: `http://localhost:9090`

Check targets at: `http://localhost:9090/targets`

## What Gets Deployed

The kube-prometheus-stack includes:
- **Prometheus**: Metrics collection and storage
- **Grafana**: Visualization dashboards
- **Alertmanager**: Alert handling
- **Node Exporter**: Node-level metrics
- **Kube State Metrics**: Kubernetes resource metrics
- **Prometheus Operator**: Manages Prometheus instances

## Default Dashboards

Grafana comes with pre-configured dashboards for:
- Kubernetes cluster overview
- Node metrics (CPU, memory, disk, network)
- Pod metrics
- Persistent volumes
- CoreDNS
- And more

## k3s Specific Notes

On k3s, some ServiceMonitors will show as "empty pools":
- kube-controller-manager
- kube-scheduler
- kube-proxy
- etcd

This is expected because k3s runs these components in a single binary differently than standard Kubernetes. The important metrics (nodes, pods, API server, kubelet) still work fine.

## Troubleshooting

If Grafana shows "no data":
1. Wait a few minutes for Prometheus to collect metrics
2. Check Prometheus targets: `http://localhost:9090/targets` (via port-forward)
3. Verify datasource in Grafana: Settings → Datasources → Prometheus
4. Check that all pods are running: `kubectl get pods -n k3s6on10`

## Understanding the Values File

The full values file is 5000+ lines. Key sections:
- **grafana**: Grafana configuration (dashboards, datasources, service, ingress)
- **prometheus**: Prometheus server configuration (storage, scraping, rules)
- **alertmanager**: Alert routing and handling
- **kubeApiServer, kubelet, etc.**: What Prometheus scrapes
- **defaultRules**: Pre-configured alerting rules

Most values are commented examples — only uncomment what you need to customize."# k3s6on10" 
