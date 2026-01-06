# k8s-components-installation

# Thanos Monitoring Architecture

This repository contains the configuration and architectural overview for our highly available, long-term metrics storage solution using **Thanos** and **Prometheus**.

## Architecture Overview

The following diagram illustrates how Thanos extends Prometheus to provide a global query view and unlimited retention by leveraging Object Storage.

```text
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



```bash
helm install my-release oci://registry-1.docker.io/bitnamicharts/thanos  -n monitoring
```

```bash
k edit prometheus prometheus-k8s -oyaml -n monitoring
```

```yaml
type: S3
config:
  bucket: thanos
  endpoint: minio.minio.svc.cluster.local:443
  access_key: WxkacHz2rVeqdF8wMMgm
  secret_key: mtqaClQTywo1qqUR1hEmv5RqITTTRweWMUgkJZLB
  insecure: false
  http_config:
    insecure_skip_verify: true
```
```bash
kubectl create secret generic thanos-objstore \
  -n monitoring \
  --from-file=objstore.yaml
```


```yaml
apiVersion: monitoring.coreos.com/v1
kind: Prometheus
  labels:
    prometheus: prometheus-k8s
  name: prometheus-k8s
spec:
spec:
  evaluationInterval: 30s
  portName: web
  replicas: 1
  ruleSelector:
    matchLabels:
      prometheus: k8s
      role: prometheus-rulefiles
  scrapeInterval: 30s
  serviceMonitorSelector:
    matchLabels:
      app.kubernetes.io/name: prometheus
----------->>>>>>>>>>
  thanos: 
    blockSize: 2m
    objectStorageConfig:
      key: objstore.yaml
      name: thanos-objstore
    version: v0.40.0
```
```
k rollout restart sts prometheus-k8s -n monitoring
```
