# k8s-components-installation
# prepare inventory.ini file and replace ip of your node (3 master and 3 worker)

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


# modify this 2 line for cilium in k8s-cluster.yaml (/kubespray/inventory/mycluster/group_vars/k8s_cluster/k8s-cluster.yaml)

kube_network_plugin: cilium
cilium_kube_proxy_replacement: true


# ansible commands to run the playbook

pip install --upgrade pip
pip install -r /kubespray/requirements.txt
python3 -m venv kubespray-venv
source kubespray-venv/bin/activate

cd /kubespray 

ansible-playbook \
  -i inventory/mycluster/inventory.ini \
  --become \
  --become-user=root \
  cluster.yml


# To add worker nodes (add new worker ip in the inventory.ini file)

ansible-playbook -i inventory/mycluster/inventory.ini scale.yml -b -v

  


