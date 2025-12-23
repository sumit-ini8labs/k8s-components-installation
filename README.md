# Rook-Ceph Monitoring Setup (Prometheus + Grafana)

This guide explains how to set up Grafana dashboards to monitor your Rook-Ceph cluster.

---

## 1️⃣ Check Ceph Cluster Health

Enter the Rook toolbox:

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- bash
```

Check Ceph cluster status:

```bash
ceph status
```

Check Ceph modules to ensure Prometheus is enabled:

```bash
ceph mgr module ls
```

> Expected: `prometheus` module should be `on`.

---

## 2️⃣ Verify Ceph MGR Services

Check the Ceph MGR services:

```bash
kubectl -n rook-ceph get svc -l app=rook-ceph-mgr
```

Check endpoints:

```bash
kubectl -n rook-ceph get endpoints rook-ceph-mgr
```

Check port name for ServiceMonitor:

```bash
kubectl -n rook-ceph get svc rook-ceph-mgr -o=jsonpath='{.spec.ports[*].name}'
```

> Expected output: `http-metrics`

---

## 3️⃣ Apply RBAC for Monitoring

Create the necessary roles, rolebindings, and service accounts for Prometheus to access Ceph metrics:

```bash
kubectl apply -f deploy/examples/monitoring/rbac.yaml
```

---

## 4️⃣ Create ServiceMonitor for Ceph MGR

Create `ceph-mgr-servicemonitor.yaml`:

```yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: ceph-mgr
  namespace: monitoring
  labels:
    release: monitoring   
spec:
  namespaceSelector:
    matchNames:
    - rook-ceph
  selector:
    matchLabels:
      app: rook-ceph-mgr
  endpoints:
  - port: http-metrics
    interval: 30s
    path: /metrics
```

Apply the ServiceMonitor:

```bash
kubectl apply -f ceph-mgr-servicemonitor.yaml
```

---

## 5️⃣ Verify Prometheus Targets

Port-forward Prometheus:

```bash
kubectl -n monitoring port-forward svc/monitoring-kube-prometheus-prometheus 9090:9090
```

Open in browser:

```
http://localhost:9090/targets
```

Check for the job `ceph-mgr` → it should be `UP`.

---

This completes the setup of Prometheus monitoring for Rook-Ceph and prepares Grafana dashboards to visualize metrics.
