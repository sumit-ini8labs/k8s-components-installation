# k8s-components-installation

                        ┌────────────────────┐
                        │      Grafana       │
                        │ (PromQL via Query) │
                        └─────────▲──────────┘
                                  │
                         ┌────────┴────────┐
                         │   Thanos Query  │
                         │ (Global Query)  │
                         └───▲────────▲────┘
                             │        │
            ┌────────────────┘        └────────────────┐
            │                                          │
┌───────────┴───────────┐                 ┌────────────┴────────────┐
│  Prometheus + Sidecar │                 │     Store Gateway       │
│  (real-time metrics)  │                 │ (historical metrics)    │
└───────────▲───────────┘                 └────────────-▲───────────┘
            │                                           │
            │                                           │
            └──────────────┬────────────────────────────┘
                           │
                 ┌─────────▼─────────┐
                 │  Object Storage   │
                 │ (MinIO / S3 / GCS)│
                 └─────────▲─────────┘
                           │
                 ┌─────────┴─────────┐
                 │     Compactor     │
                 │ (downsampling &  │
                 │   retention)     │
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
