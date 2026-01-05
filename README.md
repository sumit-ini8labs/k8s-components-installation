# k8s-components-installation

git clone https://github.com/prometheus-operator/kube-prometheus.git

# Create the namespace and CRDs, and then wait for them to be available before creating the remaining resources
kubectl create -f manifests/setup

# Wait until the "servicemonitors" CRD is created. The message "No resources found" means success in this context.
until kubectl get servicemonitors --all-namespaces ; do date; sleep 1; echo ""; done

kubectl create -f manifests/

kubectl create -f manifests/setup -f manifests


kubectl delete --ignore-not-found=true -f manifests/ -f manifests/setup
