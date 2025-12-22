# Rook-Ceph Installation and Overview

## 1. What is Rook-Ceph?

Rook is a **Kubernetes-native storage orchestrator**, and Ceph is a **distributed storage system**. Together, they provide:

| Storage Type      | Backed By            | Use Case                       | Access Mode |
| ----------------- | -------------------- | ------------------------------ | ----------- |
| RBD (Block)       | CephBlockPool        | Databases, logs, block devices | RWO         |
| CephFS (File)     | CephFilesystem + MDS | Shared app data, media         | RWX         |
| Object Store (S3) | CephObjectStore      | Backups, binaries, ML data     | API-based   |

Rook automates **deployment, scaling, upgrades, and healing** of Ceph inside Kubernetes.

---

## 2. High-Level Architecture

```
Kubernetes
 ├── Rook Operator (Brains)
 │    └── Deploys & Manages Ceph Components
 ├── Ceph Cluster (MON, MGR, OSD)
 ├── CSI Drivers (RBD & CephFS)
 └── Your Pods + PVCs
```

---

## 3. Rook-Ceph CRDs (Custom Resource Definitions)

| CRD Name        | Purpose                                                        |
| --------------- | -------------------------------------------------------------- |
| CephCluster     | Defines entire Ceph cluster (MON, MGR, OSDs, configs, network) |
| CephBlockPool   | Block storage pool (RBD) for RWO PVs                           |
| CephFilesystem  | CephFS filesystem for RWX volumes                              |
| CephObjectStore | Deploys S3-compatible object storage (RGW)                     |
| CephNFS         | NFS sharing over CephFS                                        |
| CephClient      | Generates Ceph authentication clients & keys                   |
| CephBucket      | Creates S3 buckets dynamically                                 |

> CRDs are Kubernetes API objects watched by the **Rook Operator**, which converts them into Ceph resources.

---

## 4. Ceph Components Managed by Rook

| Component | Purpose                                                                          |
| --------- | -------------------------------------------------------------------------------- |
| MON       | Cluster map, quorum, consistency (3 minimum)                                     |
| MGR       | Metrics, dashboard, orchestration, REST API (2 recommended: 1 active, 1 standby) |
| OSD       | Actual storage engine; stores & replicates data; 1 OSD per disk                  |
| MDS       | Metadata server for CephFS (required for RWX)                                    |
| RGW       | Object Storage Gateway (S3/Swift API)                                            |

---

## 5. Storage Flow Example (RBD Block PVC)

1. Create a **PVC** referencing a StorageClass (e.g., `rook-ceph-block`).
2. Kubernetes passes request to **CSI driver** (`rook-ceph.rbd.csi.ceph.com`).
3. CSI connects to Ceph MONs → Ceph creates **RBD image** in the pool.
4. PV is created → PVC becomes **Bound**.
5. Pod mounts the device → `/data` is accessible.

---

## 6. Rook-Ceph Installation Steps

### Step 1: Clone Rook repository

```bash
ggit clone --single-branch --branch v1.12.11 https://github.com/rook/rook.git
cd rook/deploy/examples
```

### Step 2: Install Rook Operator

```bash
cd rook/deploy/examples
kubectl create -f crds.yaml -f common.yaml -f operator.yaml
```

### Step 3: Create the Ceph Cluster

```bash
kubectl apply -f cluster.yaml
```

Monitor cluster creation:

```bash
kubectl get pods -n rook-ceph -w
kubectl get cephcluster -n rook-ceph
```

### Step 4: Install Toolbox for Ceph Management

```bash
kubectl apply -f toolbox.yaml
kubectl -n rook-ceph exec deploy/rook-ceph-tools -- ceph -s
kubectl -n rook-ceph exec deploy/rook-ceph-tools -- ceph health detail
kubectl -n rook-ceph exec deploy/rook-ceph-tools -- ceph osd tree
kubectl -n rook-ceph exec deploy/rook-ceph-tools -- ceph osd status
kubectl -n rook-ceph exec deploy/rook-ceph-tools -- ceph pg stat
```

### Step 5: (Optional) Deploy Ceph Dashboard

```bash
kubectl apply -f dashboard-external-http.yaml
kubectl -n rook-ceph get secret rook-ceph-dashboard-password -o jsonpath='{.data.password}' | base64 --decode
```

### Step 6: Create StorageClass and PVC

```bash
kubectl apply -f csi/rbd/storageclass.yaml
kubectl apply -f pvc.yaml
kubectl get pvc
```

### Step 7: Verify Ceph Status

```bash
kubectl apply -f toolbox.yaml
kubectl -n rook-ceph exec -it rook-ceph-tools -- bash
ceph status
kubectl get cephcluster -n rook-ceph
```

> Test by scheduling pods on different nodes to ensure PVCs attach correctly.
