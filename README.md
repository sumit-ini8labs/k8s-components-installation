# VictoriaMetrics Multitenant Multi-Cluster Monitoring with Prometheus & Grafana

This document describes how to set up **multi-cluster Kubernetes monitoring** using **Prometheus**, **VictoriaMetrics (cluster mode, multitenant)**, and **Grafana**.

The setup enables:

* Multiple Kubernetes clusters pushing metrics to a **central VictoriaMetrics cluster**
* Tenant-based isolation using `vm_account_id`
* Unified visualization in Grafana using multi-cluster dashboards

---

## Architecture Overview

```
Kubernetes Cluster A ─┐
                      ├─ Prometheus (remote_write)
Kubernetes Cluster B ─┘          │
                                 ▼
                        VictoriaMetrics Cluster
                      (vminsert / vmselect / vmstorage)
                                 │
                                 ▼
                              Grafana
```

* **Prometheus** scrapes metrics inside each Kubernetes cluster
* Metrics are sent via `remote_write` to **VictoriaMetrics vminsert**
* **vmselect** is used as the read endpoint for Grafana
* **Grafana** visualizes data across multiple clusters

---

## Prometheus Remote Write Configuration

The following configuration sends metrics from **Prometheus** to **VictoriaMetrics vminsert**.

👉 **This is the vminsert (write) endpoint**

```yaml
remoteWrite:
  - url: http://vminsert:8480/insert/multitenant/prometheus/api/v1/write
    queueConfig:
      capacity: 20000
      maxSamplesPerSend: 10000
      maxShards: 30
    externalLabels:
      vm_account_id: "2"
      cluster: "cluster-b"
```

### Explanation

* **vminsert endpoint**: Receives metrics via `remote_write`
* **vm_account_id**: Tenant identifier for VictoriaMetrics multitenancy
* **cluster**: Logical Kubernetes cluster name used in Grafana

---

---|-------------|
| `vm_account_id` | Tenant ID used by VictoriaMetrics (mandatory for multitenancy) |
| `cluster` | Logical cluster name used in Grafana dashboards |

> ⚠️ Ensure **each cluster uses a unique `vm_account_id` or `cluster` label**.

---

## VictoriaMetrics Cluster Installation (Helm)

### Add Helm Repository

```bash
helm repo add vm https://victoriametrics.github.io/helm-charts/
helm repo update
```

### Inspect Default Values

```bash
helm show values vm/victoria-metrics-cluster > values.yaml
```

### Install VictoriaMetrics Cluster

```bash
helm install vmc vm/victoria-metrics-cluster \
  -f values.yaml \
  -n <NAMESPACE> \
  --debug
```

---

## VictoriaMetrics Cluster with Prometheus Scrape Annotations

Install VictoriaMetrics Cluster with Prometheus scrape annotations enabled:

```bash
cat <<EOF | helm install vmcluster vm/victoria-metrics-cluster -f -
vmselect:
  podAnnotations:
    prometheus.io/scrape: "true"
    prometheus.io/port: "8481"

vminsert:
  podAnnotations:
    prometheus.io/scrape: "true"
    prometheus.io/port: "8480"

vmstorage:
  podAnnotations:
    prometheus.io/scrape: "true"
    prometheus.io/port: "8482"
EOF
```

This allows Prometheus to scrape VictoriaMetrics internal metrics.

---

## Grafana Configuration

### Add Prometheus Data Source

👉 **This is the vmselect (read/query) endpoint**

In **Grafana → Data Sources → Add data source → Prometheus**

**URL:**

```
http://vmselect:8481/select/multitenant/prometheus
```

* Uses **vmselect** for querying metrics
* Access: **Server**
* Click **Save & Test**

---

## Grafana Dashboards (Import IDs)

Import the following dashboards from Grafana Dashboard Marketplace:

### Kubernetes & Multi-Cluster Dashboards

| Dashboard Name                                                                     | ID        |
| ---------------------------------------------------------------------------------- | --------- |
| K8S Dashboard                                                                      | **15661** |
| K8s Node Metrics / Multi Clusters (Node Exporter, Prometheus, Grafana11, 2025, EN) | **22413** |
| Kubernetes Cluster (Prometheus)                                                    | **8171**  |

---

### Additional Recommended Dashboards

* **K8S Dashboard**

  * Tags: `Kubernetes`, `Prometheus`

* **K8s Node Metrics / Multi Clusters**

  * Tags: `Prometheus`, `node_exporter`

* **Kubernetes Cluster (Prometheus)**

  * Tags: `kubernetes`, `kubernetes-app`

* **Kubernetes Nodes**

  * Tags: `nodes`, `prometheus`

* **Multi Cluster Monitoring for Kubernetes**

  * Tags: `kubernetes`

* **Thanos / Overview**

* **VictoriaMetrics - cluster**

---

## Validation & Troubleshooting

### Verify Data in Grafana (Explore)

Use **Grafana Explore** to quickly validate that data is being read from VictoriaMetrics.

Steps:

1. Open **Grafana → Explore**
2. Select **Prometheus** datasource (vmselect endpoint)
3. Run the following queries:

```promql
up
```

```promql
up{cluster="cluster-b"}
```

```promql
up{vm_account_id="2"}
```

```promql
count(up) by (cluster)
```

Expected result:

* Metrics should return instantly
* `cluster` label should differentiate clusters
* Each tenant (`vm_account_id`) should show its own data

---

### Check Data in VictoriaMetrics UI

```
http://98.70.41.123:8481/select/multitenant/vmui/
```

Run a test query:

```
up
```

### Verify Tenant-Specific Data

```
up{vm_account_id="2"}
```

-----|---------|
| No data in Grafana | Verify Grafana datasource URL points to `vmselect` |
| Metrics visible in VMUI but not Grafana | Check Grafana time range & datasource |
| Missing cluster separation | Ensure `cluster` label is present |
| Partial data | Check `remote_write` queue configuration |

---

## Summary

# Centralized metrics storage using VictoriaMetrics
# Multitenancy via `vm_account_id`
# Multi-cluster visibility in Grafana
# Scalable and production-ready monitoring architecture

---

