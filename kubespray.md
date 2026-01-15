# Why Use Kubespray for Kubernetes?

Kubespray is a powerful, production-ready Kubernetes distribution that uses Ansible to automate the deployment and management of Kubernetes clusters.

**Specific Versions:**
- **Kubernetes**: 1.34.3
- **Kubespray**: v2.29.1 (latest stable)

## Core Features

1. **High Availability**: Deploys multi-master clusters for high reliability.
2. **Composable Networking (CNI)**: Supports various network plugins like **Calico**, **Cilium**, **Flannel**, **Kube-Router**, and **Multus**.
3. **Flexible Container Runtimes (CRI)**: Supports **Containerd** (default), **CRI-O**, and **Docker**.
4. **Multi-Cloud & Bare Metal**: Works on AWS, GCE, Azure, OpenStack, vSphere, and physical "bare metal" servers.
5. **Lifecycle Management**: Includes built-in playbooks in the root `kubespray/` directory:

    - `[cluster.yml] ----> (/kubespray/cluster.yml)`: Initial deployment.

    - `[upgrade-cluster.yml] ----> (/kubespray/upgrade-cluster.yml)`: Zero-downtime upgrades.

    - `[scale.yml] ----> (/kubespray/scale.yml)`: Adding new nodes (masters or workers).

    - `[remove-node.yml] ----> (/kubespray/remove-node.yml)`: Safely removing nodes.

    - `[reset.yml] ----> (/kubespray/reset.yml)`: Completely cleaning up a cluster.
6. **Add-ons**: Easily enables features like **Helm**, **Cert-manager**, **Ingress Controllers** (Nginx, ALB), and **Storage Provisioners**.

## What you can do with it

- **Build Production Clusters**: It treats Kubernetes as a set of services (etcd, kube-apiserver, etc.) managed by Ansible, making it very transparent and customizable.
- **Hybrid Infrastructure**: You can use the same tooling to manage clusters across different clouds or mixed environments.
- **Day-2 Operations**: Scale up or down as your needs change, and keep your versions up to date with the upgrade playbooks.

## Top Benefits

### 1. Production-Ready High Availability (HA)
Unlike simpler tools (like `kubeadm` on its own), Kubespray is designed from the ground up for high availability. It automates the setup of multi-master configurations, external etcd clusters, and load balancers for the API server.

### 2. Extreme Flexibility (CNI & CRI)
You aren't locked into specific technologies. You can choose:
- **Network Plugins (CNI)**: Calico, Cilium, Flannel, Kube-Router, Multus, etc.
- **Container Runtimes (CRI)**: Containerd (default), CRI-O, Docker.

### 3. Comprehensive Lifecycle Management
Kubespray doesn't just *install* Kubernetes; it manages it throughout its life:
- **Scaling**: Easily add new worker or master nodes using `scale.yml`.
- **Upgrading**: Perform zero-downtime upgrades to newer Kubernetes versions using `upgrade-cluster.yml`.
- **Resetting**: Clean up a node or a whole cluster completely using `reset.yml`.

### 4. Zero Lock-In (Bare Metal & Cloud)
You can use the exact same tool and configuration to deploy to:
- Bare Metal servers
- Public Clouds (AWS, Azure, GCP, OpenStack)
- Virtualized environments (vSphere, Proxmox, KVM)

### 5. Rich Add-on Ecosystem
Kubespray makes it trivial to enable common cluster services through simple configuration toggles (`addons.yml`):
- **Helm**: Package management.
- **Cert-Manager**: Automated SSL/TLS certificates.
- **Ingress-Nginx/ALB**: External access to your apps.
- **Metrics-Server**: Enable horizontal pod autoscaling.
- **Dashboard**: Web UI for cluster management.

## Advanced Capabilities with Cilium
If you choose Cilium as your network plugin, Kubespray unlocks:
- **eBPF-based Performance**: Faster networking than traditional iptables.
- **Hubble**: Deep observability into network flows and security policies.
- **Network Encryption**: Secure node-to-node communication with WireGuard.

## Configuration Roadmap

All your cluster configurations are located in your inventory directory:
`/root/kubespray/inventory/mycluster/group_vars/`

| If you want to change... | Go to this file... |
| :--- | :--- |
| **Node IPs/Hostnames** | `[inventory.ini]--->(/kubespray/inventory/mycluster/inventory.ini)` |
| **Kubernetes Version** | `[k8s-cluster.yml]--->(/kubespray/inventory/mycluster/group_vars/k8s_cluster/k8s-cluster.yml)` |
| **Add-ons (Helm, Dashboard, etc.)** | `[addons.yml]--->(/kubespray/inventory/mycluster/group_vars/k8s_cluster/addons.yml)` |
| **Cilium Settings (Hubble, eBPF)** | `[k8s-net-cilium.yml]--->(/kubespray/inventory/mycluster/group_vars/k8s_cluster/k8s-net-cilium.yml)` |
| **Internal Networking (Pod/Service CIDRs)** | `[k8s-cluster.yml]--->(/kubespray/inventory/mycluster/group_vars/k8s_cluster/k8s-cluster.yml)` |
| **External/Cloud Provider (Azure/AWS)** | `[all.yml]--->(/kubespray/inventory/mycluster/group_vars/all/all.yml)` |

## How to Enable Add-ons

To enable additional features, modify the variables in: `[addons.yml]--->(/kubespray/inventory/mycluster/group_vars/k8s_cluster/addons.yml)`

| Feature | Variable to Change |
| :--- | :--- |
| **Helm** | `helm_enabled: true` |
| **Nginx Ingress** | `ingress_nginx_enabled: true` |
| **Cert-Manager** | `cert_manager_enabled: true` |
| **Metrics Server** | `metrics_server_enabled: true` |
| **Dashboard** | `dashboard_enabled: true` |
| **ArgoCD** | `argocd_enabled: true` |
| **Local Storage** | `local_path_provisioner_enabled: true` |

## Advanced Cilium Features

If using Cilium, you can unlock more power in: `[k8s-net-cilium.yml]--->(/kubespray/inventory/mycluster/group_vars/k8s_cluster/k8s-net-cilium.yml)`

1. **Hubble (Observability)**: Set `cilium_enable_hubble: true` and `cilium_enable_hubble_ui: true` to get a graphical service map.
2. **Kube-Proxy Replacement**: Set `cilium_kube_proxy_replacement: strict` for pure eBPF performance.
3. **Encryption**: Set `cilium_encryption_enabled: true` and `cilium_encryption_type: "wireguard"` for automatic node-to-node security.
4. **L2 Announcements**: Set `cilium_l2announcements: true` to handle External IPs without MetalLB.
5. **Ingress & Gateway API**: Set `cilium_extra_values: {}` (Line 390) to pass Helm values for enabling the Cilium Ingress Controller or Gateway API.

## Prerequisites and Operational Flow

### 1. Host Preparation
Before running Kubespray, ensure your deployment host is ready:
- **Python 3.10+**: Required for Ansible.
- **Ansible 2.14+**: Installed via `setup_kubespray_prereqs.sh`.
- **SSH Access**: You must be able to SSH into all nodes without a password.
  - Use `ssh-copy-id <user>@<node-ip>` for each node.
  - Ensure your user has `sudo` privileges without a password if possible.

### 2. Operational Logic
1. **Initial Setup**: Run `setup_kubespray_prereqs.sh` to install all binaries (`kubectl`, `helm`, `ansible`).
2. **Configuration**: Use the [Configuration Roadmap](#configuration-roadmap) to set your IPs and cluster settings.
3. **Deployment**: Run `cluster.yml`.
4. **Day-2 Maintenance**:
   - Use `scale.yml` to grow your cluster.
   - Use `upgrade-cluster.yml` to keep Kubernetes updated.
   - Use `remove-node.yml` to scale down.

## How to Apply Changes

After you modify any of the files above, run the main playbook to apply the configuration:

```bash
ansible-playbook -i inventory/mycluster/inventory.ini cluster.yml --become
```
