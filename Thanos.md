# Thanos on Kubernetes (K8s)

This document provides a **step-by-step guide** to install **Thanos on Kubernetes** using **Prometheus Operator** and **MinIO (S3-compatible object storage)**. It includes **all required commands**, configuration files, and verification steps.

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
        ┌─────────────────────┘        └─────-───────────────┐
        │                                                    │
┌────────────┴────────────┐                     ┌─────────────────┴─────────────────┐
│ Prometheus + Sidecar    │                     │         Store Gateway             │
│ (Live Metrics + Upload) │                     │ (Historical Metrics from S3)      │
└────────────▲────────────┘                     └─────────────────▲─────────────────┘
        │                                                    │
        └──────────────────┬─────────────────────────────────┘
                           │
                 ┌─────────▼─────────┐
                 │  Object Storage   │
                 │ (MinIO / S3 / GCS)│
                 └─────────▲─────────┘
                           │
                 ┌─────────┴────────┐
                 │     Compactor    │
                 │ (Retention &     │
                 │  Downsampling)   │
                 └──────────────────┘
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

Install Thanos Query: for multi cluster setup add the ip:port in arg section in query deployment

```ymal
    spec:
      containers:
      - name: thanos-query
        image: quay.io/thanos/thanos:v0.39.0
        args:
          - query
          - --http-address=0.0.0.0:9090
          - --grpc-address=0.0.0.0:10901
          - --endpoint=98.70.25.226:10901  ------>>>>>>>>> add thanos-store-gateway of cluster B address
          - --endpoint=dnssrv+_grpc._tcp.prometheus-operated.monitoring.svc.cluster.local
          - --endpoint=dnssrv+_grpc._tcp.thanos-store.monitoring.svc.cluster.local
```

```bash
kubectl apply -f https://raw.githubusercontent.com/sumit-ini8labs/k8s-components-installation/refs/heads/thanos-installation/thanos-query.yaml
```

Verify:

```bash
kubectl get pods -n monitoring | grep thanos
```

---

## Step 7: Install Thanos Store Gateway

```bash
kubectl -f apply https://raw.githubusercontent.com/sumit-ini8labs/k8s-components-installation/refs/heads/thanos-installation/thanos-store-statefulSet.yaml
```

---

## Step 9: Install thanos compactor
```bash
https://raw.githubusercontent.com/sumit-ini8labs/k8s-components-installation/refs/heads/thanos-installation/thanos-compactor.yaml
```

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

✅ Thanos is now successfully installed on Kubernetes!
