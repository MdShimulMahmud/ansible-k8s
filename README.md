# 🧩 Ansible for Kubernetes Cluster Provisioning 🚀

This project uses **Ansible** to completely automate the creation of a **multi-node Kubernetes cluster** using **kubeadm**.  
The playbooks will prepare the nodes, initialize a control plane, and join worker nodes — resulting in a clean, functional cluster.

---

## 🧱 Prerequisites

Before you begin, ensure you have the following:

- ✅ **Ansible** installed on your control node (e.g., your local machine or a bastion host)
- ✅ **Passwordless SSH access** from your control node to all target server nodes
- ✅ **Three or more servers** running a Debian-based OS like **Ubuntu 22.04**

---

## ⚙️ Configuration

1. Clone this repository or create the project files as shown in the structure below.
2. Edit the `inventory.ini` file to match the IP addresses and SSH details for your master and worker nodes.

---

## 📁 Project Structure
```
ansible-k8s/
├── inventory.ini
├── README.md
├── site.yml
└── playbooks/
    ├── playbook-01-prepare-nodes.yml
    ├── playbook-02-control-plane.yml
    └── playbook-03-workers.yml
```

---

## 🧾 inventory.ini

```ini
[control_plane]
master1 ansible_host=YOUR_MASTER_IP

[workers]
worker1 ansible_host=YOUR_WORKER1_IP
worker2 ansible_host=YOUR_WORKER2_IP

[all:vars]
ansible_user=ubuntu
ansible_ssh_private_key_file=~/.ssh/id_rsa
ansible_python_interpreter=/usr/bin/python3
```

▶️ How to Run

From the root directory of the project (ansible-k8s/), execute the main playbook.
This single command will build the entire cluster:

```ansible
ansible-playbook -i inventory.ini site.yml
```