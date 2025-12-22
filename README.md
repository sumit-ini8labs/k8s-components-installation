# Setup Grafana Dashboard for CephCluster


Configuration of Grafana dashboard for ceph cluster
Rook-Ceph Monitoring Setup Guide (Prometheus + Grafana) 1️⃣ Check Ceph Cluster Health

Enter rook toolbox
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- bash

Check Ceph cluster status
ceph status

Check Ceph modules (ensure prometheus is enabled)
ceph mgr module ls

Expected: prometheus module should be on.

2️⃣ Verify Ceph MGR Services

Check mgr services
kubectl -n rook-ceph get svc -l app=rook-ceph-mgr

Check endpoints
kubectl -n rook-ceph get endpoints rook-ceph-mgr

Check port name for ServiceMonitor
kubectl -n rook-ceph get svc rook-ceph-mgr -o=jsonpath='{.spec.ports[*].name}'

Expected output: http-metrics
3️⃣ Apply RBAC for Monitoring kubectl apply -f deploy/examples/monitoring/rbac.yaml This creates the necessary roles, rolebindings, and service accounts for Prometheus to access Ceph metrics.

4️⃣ Create ServiceMonitor for Ceph MGR File: ceph-mgr-servicemonitor.yaml apiVersion: monitoring.coreos.com/v1 kind: ServiceMonitor metadata: name: ceph-mgr namespace: monitoring labels: release: monitoring # must match Prometheus Helm release spec: namespaceSelector: matchNames: - rook-ceph selector: matchLabels: app: rook-ceph-mgr endpoints: - port: http-metrics # must match service port name path: /metrics interval: 15s relabelings: - targetLabel: cluster replacement: rook-ceph

kubectl apply -f ceph-mgr-servicemonitor.yaml

5️⃣ Verify Prometheus Targets

Port-forward Prometheus
kubectl -n monitoring port-forward svc/monitoring-kube-prometheus-prometheus 9090:9090

Open browser
http://localhost:9090/targets

Look for job "ceph-mgr" → should be UP
