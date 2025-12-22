# rook-ceph installation steps

Architecture of Rook-Ceph

What is Rook-Ceph?
Rook is a Kubernetes-native storage orchestrator, and Ceph is the distributed storage system that provides:
Block Storage → RBD
File Storage → CephFS
Object Storage → RGW
Rook automates deployment, scaling, upgrades, healing of Ceph inside Kubernetes.

High-Level Architecture Overview

Kubernetes
   │
   ├── Rook Operator (Brains)
   │        │
   │        └── Deploys & Manages Ceph Components
   │
   ├── Ceph Cluster (MON, MGR, OSD)
   │
   ├── CSI Drivers (RBD & CephFS)
   │
   └── Your Pods + PVCs


Rook-Ceph CRDs (Custom Resource Definitions)

Rook installs a set of CRDs that represent Ceph objects inside Kubernetes.
CRD
Purpose
CephCluster
Defines the entire Ceph cluster (MON, MGR, OSDs, networks, configs).
CephBlockPool
Block storage pool (RBD). Used for ReadWriteOnce persistent volumes.
CephFilesystem
Defines a CephFS filesystem for RWX volumes.
CephObjectStore
Deploys an S3-compatible storage (RGW).
CephNFS
NFS sharing over CephFS.
CephClient
Generates Ceph auth clients & keys.
CephBucket
Creates S3 buckets dynamically.


CRDs = "API objects" that Rook watches and converts into Ceph resources.




Rook Operator (The Brain)
The Rook Operator is a Kubernetes controller.
It watches CRDs:
CephCluster
CephBlockPool
CephFilesystem
CephObjectStore
CephNFS
When you create or modify a CRD, the Operator:

Deploys Ceph MONs, MGRs, OSDs
Monitors the health
Handles failover
Manages upgrades
Re-creates failed daemons
Creates pools, filesystems, keys, configs



Ceph Components Managed by Rook

(1) Ceph Monitor (MON)
Minimum 3 MONs for quorum.
Responsibilities:
Maintain cluster map (OSDs, pools, PGs)
Handle membership
Provide cluster consistency
Act as control-plane of Ceph
MON = Brain of Ceph cluster

(2) Ceph Manager (MGR)
Provides:
Metrics
Dashboard
Balancing
Orchestration
REST API
Rook typically deploys 2 MGRs (1 active, 1 standby).





(3) Ceph OSDs (Object Storage Daemons)
OSD = actual storage engine.
Each OSD:
Stores data chunks
Replicates / erasure codes data
Participates in health checks
Handles recovery, rebalancing
You need 1 OSD per storage disk.
More OSDs = more capacity + more performance.

(4) Ceph MDS (Metadata Server)
Only for CephFS (File Storage).
MDS stores directory metadata so thousands of Pods can share files.

(5) Ceph RGW (RADOS Gateway)
For Object Storage (S3/Swift).
Provides:
S3 API
Multi-site replication
Buckets, users, keys
Ceph Storage Types
Storage Type
Backed By
Use Case
Access Mode
RBD (Block Storage)
CephBlockPool
Databases, logs, block devices
RWO
CephFS (File System)
CephFilesystem + MDS
Shared app data, media
RWX
Object Store (S3)
CephObjectStore
Backups, binaries, ML data
API-based



Full Storage Flow (End-to-End)
When you create a PVC (RBD Block)
You apply PVC YAML
Kubernetes sees you use StorageClass rook-ceph-block
StorageClass points to CSI driver → rook-ceph.rbd.csi.ceph.com
RBD CSI provisioner receives request
CSI connects to Ceph MONs
Ceph creates RBD image in pool
PV gets created
PVC becomes Bound
When Pod starts → RBD mapped to node
Device formatted & mounted
Pod receives /data mount


Rook-Ceph Installation Guide


Install Rook Operator
Step 1: Clone the Rook repository
git clone https://github.com/rook/rook.git
cd rook/deploy/examples
Step 2: Apply the CRDs
kubectl apply -f crds.yaml
kubectl apply -f common.yaml
kubectl apply -f operator.yaml
Create the Ceph Cluster
Step 1: Apply cluster manifest
kubectl apply -f cluster.yaml

Step 2: Monitor cluster creation
kubectl get pods -n rook-ceph -w
k get cephcluster -n rook-ceph

k apply -f toolbox.yaml 
kubectl -n rook-ceph exec deploy/rook-ceph-tools -- ceph -s
​​kubectl -n rook-ceph exec deploy/rook-ceph-tools -- ceph health detail
This shows the global health, number of OSDs, PG states, and the top-level reason for HEALTH_WARN.

kubectl -n rook-ceph exec deploy/rook-ceph-tools -- ceph osd tree
kubectl -n rook-ceph exec deploy/rook-ceph-tools -- ceph osd status
kubectl -n rook-ceph exec deploy/rook-ceph-tools -- ceph pg stat
Check OSD inventory and state:

Deploy Ceph Dashboard (Optional)
kubectl apply -f dashboard-external-http.yaml
kubectl -n rook-ceph get secret rook-ceph-dashboard-password -o jsonpath='{.data.password}' | base64 --decode
Create StorageClass
kubectl apply -f csi/rbd/storageclass.yaml
kubectl apply -f pvc.yaml

kubectl get pvc
Verify Ceph Status
kubectl apply -f toolbox.yaml
kubectl -n rook-ceph exec -it rook-ceph-tools -- bash
ceph status
kubectl get cephcluster -n rook-ceph

Try scheduling the pod on a different node and check whether the PVC gets attached or not.
