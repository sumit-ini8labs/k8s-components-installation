# Thanos on Kubernetes (K8s)

This document provides a **step-by-step guide** to install **Thanos on Kubernetes** using **Prometheus Operator** and **MinIO (S3-compatible object storage)**. It includes **all required commands**, configuration files, and verification steps.

---

## Architecture Overview

```
Prometheus (HA) + Sidecar
            |
            v
        Object Storage
            ^
            |
        Compactor
            |
            v
      Store Gateway
            |
            v
        Thanos Query
            |
            v
          Grafana
```

---

- **Prometheus** – Collects and stores metrics locally and evaluates rules.
- **Thanos Sidecar** – Uploads metrics to object storage and enables global queries.
- **Object Storage** – Durable long-term storage for metrics data.
- **Store Gateway** – Serves historical metrics from object storage.
- **Compactor** – Compacts, downsamples, and applies retention policies.
- **Thanos Query** – Aggregates and deduplicates metrics globally.
- **Query Frontend** – Caches and splits queries for performance.
- **Thanos Ruler** – Evaluates rules and sends alerts.
- **Alertmanager** – Routes and manages alerts.



## Prerequisites

* Kubernetes cluster (K3s / K8s v1.24+ recommended)
* kubectl configured
* Helm v3 installed
* Internet access from cluster

Check versions:

```bash
kubectl version
helm version
```

---

## Step 1: Create Namespace

```bash
kubectl create namespace monitoring
```

---

## Step 2: Install Prometheus Operator (kube-prometheus-stack)

Add Helm repo:

```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update
```

Install:

```bash
helm install kube-prometheus prometheus-community/kube-prometheus-stack \
  -n monitoring
```

Verify:

```bash
kubectl get pods -n monitoring
```

---

## Step 3: Install MinIO (Object Storage for Thanos)

### Add MinIO Helm Repo

```bash
helm repo add minio https://charts.min.io/
helm repo update
```

### Install MinIO

```bash
helm install minio minio/minio \
  -n monitoring \
  --set accessKey=minioadmin \
  --set secretKey=minioadmin \
  --set persistence.enabled=true \
  --set persistence.size=10Gi
```

Verify:

```bash
kubectl get pods -n monitoring | grep minio
kubectl get svc -n monitoring | grep minio
```

### Access MinIO Console

Port-forward:

```bash
kubectl port-forward svc/minio-console 9443:9443 -n monitoring
```

Login:

* URL: [https://localhost:9443](https://localhost:9443)
* Username: `minioadmin`
* Password: `minioadmin`

Create bucket:

```
thanos
```

---

## Step 4: Create Thanos Object Storage Secret

Create `objstore.yaml`:

```yaml
type: S3
config:
  bucket: thanos
  endpoint: minio.monitoring.svc.cluster.local:9000
  access_key: minioadmin
  secret_key: minioadmin
  insecure: true
```

Create secret:

```bash
kubectl create secret generic thanos-objstore \
  --from-file=objstore.yaml \
  -n monitoring
```

Verify:

```bash
kubectl get secret thanos-objstore -n monitoring
```

---

## Step 5: Enable Thanos Sidecar in Prometheus

Edit Prometheus CR:

```bash
kubectl edit prometheus kube-prometheus-kube-prome-prometheus -n monitoring
```

Add under `spec`:

```yaml
spec:
  thanos:
    objectStorageConfig:
      existingSecret:
        name: thanos-objstore
        key: objstore.yaml
```

Save and exit.

Verify sidecar:

```bash
kubectl get pods -n monitoring
```

Prometheus pod should show **3/3 containers**.

---

## Step 6: Install Thanos Query

Add Bitnami repo:

```bash
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo update
```

Install Thanos Query:

```bash
helm install thanos-query bitnami/thanos \
  -n monitoring \
  --set query.enabled=true \
  --set query.replicaCount=1 \
  --set query.dnsDiscovery.sidecarsService=kube-prometheus-kube-prome-prometheus \
  --set query.dnsDiscovery.sidecarsNamespace=monitoring
```

Verify:

```bash
kubectl get pods -n monitoring | grep thanos
```

---

## Step 7: (Optional) Install Thanos Store Gateway

```bash
helm upgrade --install thanos-store bitnami/thanos \
  -n monitoring \
  --set storegateway.enabled=true \
  --set objstoreConfig.existingSecret=thanos-objstore
```

---

## Step 8: Access Thanos Query UI

Port-forward:

```bash
kubectl port-forward svc/thanos-query 9090:9090 -n monitoring
```

Open:

```
http://localhost:9090
```

---

## Step 9: Configure Grafana with Thanos

Port-forward Grafana:

```bash
kubectl port-forward svc/kube-prometheus-grafana 3000:80 -n monitoring
```

Login:

* Username: `admin`
* Password:

```bash
kubectl get secret kube-prometheus-grafana -n monitoring \
  -o jsonpath="{.data.admin-password}" | base64 -d
```

### Add Data Source

* Type: Prometheus
* URL:

```
http://thanos-query.monitoring.svc.cluster.local:9090
```

* Save & Test

---

## Step 10: Verification

Check targets:

```bash
kubectl port-forward svc/thanos-query 9090:9090 -n monitoring
```

Open:

```
http://localhost:9090/stores
```

Expected:

* Prometheus sidecar: UP
* Store Gateway (if installed): UP

---

## Common Troubleshooting

### No Data in Grafana

* Ensure Prometheus sidecar is running
* Verify bucket exists in MinIO
* Check Thanos Query `/stores`

### Thanos Sidecar CrashLoop

```bash
kubectl logs prometheus-<pod> -c thanos-sidecar -n monitoring
```

### Metrics Missing

* Ensure `insecure: true` for MinIO
* Verify endpoint and bucket name

---

## Cleanup

```bash
helm uninstall kube-prometheus -n monitoring
helm uninstall minio -n monitoring
helm uninstall thanos-query -n monitoring
kubectl delete namespace monitoring
```

---

## References

* [https://thanos.io](https://thanos.io)
* [https://github.com/prometheus-operator/prometheus-operator](https://github.com/prometheus-operator/prometheus-operator)
* [https://artifacthub.io/packages/helm/bitnami/thanos](https://artifacthub.io/packages/helm/bitnami/thanos)

---

✅ Thanos is now successfully installed on Kubernetes!
