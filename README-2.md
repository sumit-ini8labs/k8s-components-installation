# Kubernetes Cluster Deployment with Kubespray

This guide provides the necessary steps to deploy a production-ready Kubernetes cluster using Kubespray.

## 0️⃣ System Preparation

Kubespray requires **Python 3.10+** and **Ansible 2.14+**. Follow the steps for your operating system:

### For Ubuntu / Debian
```bash
sudo apt update
sudo apt install -y python3 python3-pip python3-venv git
```

### For CentOS / RHEL / Rocky Linux
```bash
sudo dnf install -y python3 python3-pip python3-setuptools git
```

### Install Ansible
It is best to install Ansible via `pip` to ensure version compatibility:
```bash
# Optional: Use a virtual environment
python3 -m venv kubespray-venv
source kubespray-venv/bin/activate

# Install Ansible and dependencies
pip install --upgrade pip
git clone -b master https://github.com/kubernetes-sigs/kubespray.git
cd kubespray
pip install -r requirements.txt
```

---

## 1️⃣ Verify Versions (Optional)

To check the default versions configured in your current Kubespray branch:

```bash
# Check the default Kubernetes version
grep "kube_version:" roles/kubespray_defaults/defaults/main/main.yml

# List the latest supported K8s versions and their checksums
head -n 20 roles/kubespray_defaults/vars/main/checksums.yml | grep -A 5 "kubelet_checksums:"

# Check the Kubespray version
grep "version:" galaxy.yml
```

---

## 2️⃣ Prepare Inventory

Prepare your inventory by copying the sample:

```bash
cp -rfp inventory/sample inventory/mycluster
```

Edit `inventory/mycluster/inventory.ini` to define your cluster nodes. Replace the IP addresses with your actual node IPs:

```ini
[all]
master1 ansible_host=10.42.0.51 ip=10.42.0.51
master2 ansible_host=10.42.0.52 ip=10.42.0.52
master3 ansible_host=10.42.0.53 ip=10.42.0.53
worker1 ansible_host=10.42.0.54 ip=10.42.0.54
worker2 ansible_host=10.42.0.55 ip=10.42.0.55
worker3 ansible_host=10.42.0.56 ip=10.42.0.56

[kube_control_plane]
master1
master2
master3

[etcd]
master1
master2
master3

[kube_node]
worker1
worker2
worker3

[calico_rr]

[k8s_cluster:children]
kube_control_plane
kube_node
```

Test connectivity:
```bash
ansible all -i inventory/mycluster/inventory.ini -m ping
```

---

## 3️⃣ Configure Networking (Cilium)

To use **Cilium** instead of the default Calico, edit `inventory/mycluster/group_vars/k8s_cluster/k8s-cluster.yml`:

```yaml
kube_network_plugin: cilium
cilium_kube_proxy_replacement: true
```

---

## 4️⃣ Deploy Kubernetes Cluster

Run the main playbook to deploy the cluster:

```bash
ansible-playbook -i inventory/mycluster/inventory.ini --become cluster.yml
```

> **Note**: This process can take 15-30 minutes depending on your hardware and network.

---

## 5️⃣ Scaling the Cluster

To add a new worker node:

1. Update your `inventory/mycluster/inventory.ini` to include the new node:
   ```ini
   worker4 ansible_host=10.42.0.57 ip=10.42.0.57
   ```
   Add it to the `[kube_node]` group.

2. Run the **scale playbook**, limiting the execution to the new node to save time:
   ```bash
   ansible-playbook -i inventory/mycluster/inventory.ini scale.yml -b --limit=worker4
   ```

---

✅ Your production-ready **Highly Available** cluster is now ready!
