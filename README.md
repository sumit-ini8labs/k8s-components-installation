# NATS with JetStream Installation on Kubernetes

This guide explains how to install NATS with JetStream on Kubernetes using Helm and manage the cluster.

---

## 1️⃣ Add NATS Helm Repository

```bash
helm repo add nats https://nats-io.github.io/k8s/helm/charts/
helm repo update
```

---

## 2️⃣ Install NATS with Helm

```bash
helm install nats nats/nats --namespace nats --create-namespace
```

> This will deploy a NATS cluster with JetStream enabled using default configurations.

---

## 3️⃣ Configure JetStream

### Update ConfigMap

* Edit or create `nats-config.yaml` and add the JetStream section:

```yaml
jetstream:
  store_dir: /data/jetstream
  max_mem_store: 1G
  max_file_store: 5G
```

* Apply the ConfigMap:

```bash
kubectl apply -f nats-config.yaml -n nats
```

### Update StatefulSet for Persistent Storage

* Ensure the StatefulSet mounts a persistent volume at `/data/jetstream`:

```yaml
volumeMounts:
  - name: nats-jetstream-data
    mountPath: /data/jetstream
volumes:
  - name: nats-jetstream-data
    persistentVolumeClaim:
      claimName: nats-jetstream-pvc
```

* If PVC does not exist, create one:

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: nats-jetstream-pvc
  namespace: nats
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 5Gi
```

* Apply the PVC:

```bash
kubectl apply -f nats-jetstream-pvc.yaml
```

* Restart NATS pods to pick up the new configuration:

```bash
kubectl rollout restart sts nats -n nats
```

---

## 4️⃣ Check StatefulSet and ConfigMap

Verify the NATS StatefulSet:

```bash
kubectl get sts nats -n nats -o yaml
```

Check the configuration:

```bash
kubectl get cm nats-config -n nats -o yaml
```

Example key configurations in `nats.conf`:

```json
{
  "http_port": 8222,
  "lame_duck_duration": "30s",
  "lame_duck_grace_period": "10s",
  "pid_file": "/var/run/nats/nats.pid",
  "port": 4222,
  "server_name": $SERVER_NAME,

  "jetstream": { ------------------->>>>>>>>.
    "store_dir": "/data/jetstream",
    "max_mem_store": "1G",
    "max_file_store": "5G"
  } -------------------------------->>>>>>>>

}
```

> This configuration enables JetStream storage with a memory store of 1G and file store of 5G.

---

## 5️⃣ Scale NATS Cluster

Increase replicas to 3:

```bash
kubectl scale statefulset nats -n nats --replicas=3
```

Check logs for subscribers:

```bash
kubectl logs -f deploy/nats-subscriber -n nats
```

---

## 6️⃣ Access NATS Pod

```bash
kubectl exec -it nats-box-5c8796647b-f6gwh -n nats -- sh
```

> This gives you shell access inside a NATS client pod.

---

## 7️⃣ Key StatefulSet Details

* **Replicas**: 3
* **JetStream Storage**: Mounted at `/data/jetstream` via PVC `nats-jetstream-pvc`
* **Configuration**: Mounted from ConfigMap `nats-config`
* **Ports**:

  * 4222 (NATS client)
  * 8222 (Monitoring)
* **Reloader Container**: Watches config changes and reloads NATS server

---

## 8️⃣ Uninstall NATS

To remove the deployment:

```bash
helm uninstall nats -n nats
kubectl delete ns nats
```

---

## 9️⃣ Reference YAMLs

### ConfigMap (nats-config)

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: nats-config
  namespace: nats
data:
  nats.conf: |
    {
      "http_port": 8222,
      "lame_duck_duration": "30s",
      "lame_duck_grace_period": "10s",
      "pid_file": "/var/run/nats/nats.pid",
      "port": 4222,
      "server_name": $SERVER_NAME,
      "jetstream": {
        "store_dir": "/data/jetstream",
        "max_mem_store": "1G",
        "max_file_store": "5G"
      }
    }
```

### StatefulSet (nats)

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: nats
  namespace: nats
spec:
  replicas: 3
  serviceName: nats-headless
  template:
    spec:
      containers:
      - name: nats
        image: nats:2.12.3-alpine
        args:
        - --config
        - /etc/nats-config/nats.conf
        volumeMounts:
        - name: nats-jetstream-data
          mountPath: /data/jetstream
      volumes:
      - name: nats-jetstream-data
        persistentVolumeClaim:
          claimName: nats-jetstream-pvc
```
