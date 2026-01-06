# Grafana Installation on Kubernetes using Helm

This guide shows how to install **Grafana on Kubernetes** using **Helm**, with commands broken down step by step for clarity and easy troubleshooting.

---

## Prerequisites

* Kubernetes cluster up and running
* `kubectl` configured to access the cluster
* Helm v3 installed

Verify:

```bash
kubectl version
helm version
```

---

## Step 1: Add Grafana Helm Repository

```bash
helm repo add grafana https://grafana.github.io/helm-charts
```

Update repo index:

```bash
helm repo update
```

---

## Step 2: Create Monitoring Namespace

```bash
kubectl create namespace monitoring
```

---

## Step 3: Search Grafana Chart (Optional)

```bash
helm search repo grafana/grafana
```

---

## Step 4: Install Grafana using Helm

```bash
helm install my-grafana grafana/grafana \
  --namespace monitoring
```

Verify installation:

```bash
kubectl get pods -n monitoring
kubectl get svc -n monitoring
```

---

## Step 5: Get Grafana Admin Password

```bash
kubectl get secret my-grafana \
  --namespace monitoring \
  -o jsonpath="{.data.admin-password}" | base64 --decode
```

**Default username:** `admin`

---

## Step 6: Get Grafana Pod Name

```bash
export POD_NAME=$(kubectl get pods \
  --namespace monitoring \
  -l "app.kubernetes.io/name=grafana,app.kubernetes.io/instance=my-grafana" \
  -o jsonpath="{.items[0].metadata.name}")
```

Verify:

```bash
echo $POD_NAME
```

---

## Step 7: Access Grafana Dashboard (Port Forward)

```bash
kubectl --namespace monitoring port-forward $POD_NAME 3000:3000
```

Open in browser:

```
http://localhost:3000
```

Login:

* **Username:** admin
* **Password:** Retrieved in Step 5

---

## Cleanup (Optional)

```bash
helm uninstall my-grafana -n monitoring
kubectl delete namespace monitoring
```

---

✅ Grafana is now successfully installed and accessible on Kubernetes.
