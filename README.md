# Thanos on Kubernetes (K8s)

This document provides a **step-by-step guide** to install **Thanos on Kubernetes** using **Prometheus Operator (kube‑prometheus‑stack)** and **MinIO (S3‑compatible object storage)**. It includes **all required commands**, configuration files, and verification steps.

---

## Architecture Overview

```
                        ┌────────────────────┐
                        │      Grafana       │
                        │  (Dashboards)      │
                        └─────────▲──────────┘
                                  │
                     ┌────────────┴───────────┐
                     │      Thanos Query      │
                     │  (Global Query Layer)  │
                     └───────▲────────▲───────┘
                             │        │
        ┌─────────────────────┘        └─────────────────────────────┐
        │                                                             │
┌────────────┴────────────┐                     ┌───────────────────┴───────────────────┐
│ Prometheus + Sidecar    │                     │           Store Gateway               │
│ (Live Metrics + Upload) │                     │ (Historical Metrics from Object Store)│
└────────────▲────────────┘                     └───────────────────▲───────────────────┘
        │                                                             │
        └──────────────────┬─────────────────────────────────────────┘
                           │
                 ┌─────────▼─────────┐
                 │  Object Storage   │
                 │ (MinIO / S3 / GCS)│
                 └─────────▲─────────┘
                           │
                 ┌─────────┴─────────┐
                 │     Compactor     │
                 │ (Retention &      │
                 │  Downsampling)    │
                 └───────────────────┘
```

---

## Components

* **Prometheus** – Scrapes metrics and evaluates rules
* **Thanos Sidecar** – Uploads Prometheus blocks to object storage and exposes gRPC
* **Object Storage (MinIO)** – Durable long‑term metrics storage
* **Store Gateway** – Serves historical data from object storage
* **Compactor** – Compacts blocks, downsamples, and enforces retention
* **Thanos Query** – Aggregates and deduplicates metrics globally
* **Query Frontend** – (Optional) Query caching and splitting
* **Thanos Ruler** – (Optional) Rule evaluation and alerting
* **Alertmanager** – Routes alerts

---

## Prerequisites

* Kubernetes cluster
* `kubectl`, `helm`
* kube‑prometheus‑stack installed in `monitoring` namespace

---

## Step 3: Install MinIO (Object Storage for Thanos)

### Add MinIO Helm Repository

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

> ⚠️ **Production note**: Use Kubernetes secrets and stronger credentials in real environments.

### Verify

```bash
kubectl get pods -n monitoring | grep minio
kubectl get svc -n monitoring | grep minio
```

### Access MinIO Console

```bash
kubectl port-forward svc/minio-console 9443:9443 -n monitoring
```

* URL: [https://localhost:9443](https://localhost:9443)
* Username: `minioadmin`
* Password: `minioadmin`

Create a bucket and Access_key:

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
  access_key: jsdafioweursndfklsd
  secret_key: fwae89w7r9798dfasdjfljsdf8wuo
  insecure: true
```

Create the secret:

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

Edit the Prometheus CR:

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

Verify:

```bash
kubectl get pods -n monitoring
```

Prometheus pods should show **3/3 containers** (Prometheus + Config Reloader + Thanos Sidecar).

---

## Step 6: Install Thanos Query

Thanos Query connects to:

* Prometheus sidecars (live data)
* Store Gateways (historical data)
* Optional remote clusters (multi‑cluster)

### Example Configuration (Multi‑Cluster)

```yaml
spec:
  containers:
  - name: thanos-query
    image: quay.io/thanos/thanos:v0.39.0
    args:
      - query
      - --http-address=0.0.0.0:9090
      - --grpc-address=0.0.0.0:10901
      # Remote cluster store gateway / sidecar
      - --endpoint=98.70.25.226:10901
      # Local Prometheus sidecars
      - --endpoint=dnssrv+_grpc._tcp.prometheus-operated.monitoring.svc.cluster.local
      # Local store gateway
      - --endpoint=dnssrv+_grpc._tcp.thanos-store.monitoring.svc.cluster.local
```

Apply:

```bash
kubectl apply -f https://raw.githubusercontent.com/sumit-ini8labs/k8s-components-installation/refs/heads/thanos-installation/thanos-query.yaml
```

Verify:

```bash
kubectl get pods -n monitoring | grep thanos
```

---

## Step 7: Install Thanos Store Gateway

make sure the s3 secret name is correct

```bash
kubectl apply -f https://raw.githubusercontent.com/sumit-ini8labs/k8s-components-installation/refs/heads/thanos-installation/thanos-store-statefulSet.yaml
```

---

## Step 8: Install Thanos Compactor

make sure the s3 secret name is correct

```bash
kubectl apply -f https://raw.githubusercontent.com/sumit-ini8labs/k8s-components-installation/refs/heads/thanos-installation/thanos-compactor.yaml
```

---

## Step 9: Access Thanos Query UI

```bash
kubectl port-forward svc/thanos-query 9090:9090 -n monitoring
```

Open:

```
http://localhost:9090
```

---

## Step 10: Configure Grafana with Thanos

Port‑forward Grafana:

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

* Type: **Prometheus**
* URL:

```
http://thanos-query.monitoring.svc.cluster.local:9090
```

Click **Save & Test**.

---

## Step 11: Verification

Check connected stores:

```bash
kubectl port-forward svc/thanos-query 9090:9090 -n monitoring
```

Open:

```
http://localhost:9090/stores
```

Expected:

* ✅ Prometheus Sidecar – UP
* ✅ Store Gateway – UP
* ✅ Remote clusters (if configured) – UP

---

## Common Troubleshooting

### No Data in Grafana

* Ensure Prometheus sidecar is running
* Verify `thanos` bucket exists in MinIO
* Check Thanos Query `/stores` page

### Thanos Sidecar CrashLoop

```bash
kubectl logs prometheus-<pod-name> -c thanos-sidecar -n monitoring
```

### Store Gateway Not Showing

* Verify object storage secret
* Ensure compactor has write access
* Check Store Gateway logs

---

✅ **Thanos is now correctly installed and configured on Kubernetes with MinIO and Prometheus Operator.**
