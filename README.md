# Prepare Inventory
Create or edit the inventory.ini file for your cluster. Replace the IP addresses with your nodes’ IPs:

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

# Configure Cilium

Edit /kubespray/inventory/mycluster/group_vars/k8s_cluster/k8s-cluster.yaml and set the network plugin to Cilium:

kube_network_plugin: cilium
cilium_kube_proxy_replacement: true

This enables Cilium and replaces kube-proxy with Cilium’s eBPF mode.

# Install Python Dependencies
# Upgrade pip
pip install --upgrade pip

# Install Kubespray requirements
pip install -r /kubespray/requirements.txt

# create and activate virtual environment
python3 -m venv kubespray-venv
source kubespray-venv/bin/activate
# Deploy Kubernetes Cluster
cd /kubespray

ansible-playbook \
  -i inventory/mycluster/inventory.ini \
  --become \
  --become-user=root \
  cluster.yml

# Add New Worker Nodes
Add the new worker node(s) in the inventory file under [kube_node]:
worker4 ansible_host=10.42.0.57 ip=10.42.0.57
Run the scale playbook:

ansible-playbook -i inventory/mycluster/inventory.ini scale.yml -b -v
