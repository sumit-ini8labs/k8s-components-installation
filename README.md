# k8s-components-installation

# Grafana Installation on Kubernetes (K3s) using Helm

This guide explains how to install Grafana on your K3s cluster using Helm and access it locally.

Run the following commands to install and access Grafana:

```bash
helm repo add grafana https://grafana.github.io/helm-charts && \
helm repo update && \
kubectl create namespace monitoring && \
helm search repo grafana/grafana && \
helm install my-grafana grafana/grafana --namespace monitoring && \
kubectl get secret --namespace monitoring my-grafana -o jsonpath="{.data.admin-password}" | base64 --decode && \
export POD_NAME=$(kubectl get pods --namespace monitoring -l "app.kubernetes.io/name=grafana,app.kubernetes.io/instance=my-grafana" -o jsonpath="{.items[0].metadata.name}") && \
kubectl --namespace monitoring port-forward $POD_NAME 3000:3000
