# Vagrant Kubernetes Cluster

A local Kubernetes cluster setup using Vagrant, VMware Fusion, and Ansible. This project provisions a 3-node cluster (1 master + 2 workers) for learning and development purposes.

## Prerequisites

- [Vagrant](https://www.vagrantup.com/downloads)
- [VMware Fusion](https://www.vmware.com/products/fusion.html) (Pro or Player)
- [Vagrant VMware Utility](https://developer.hashicorp.com/vagrant/docs/providers/vmware/vagrant-vmware-utility)
- [Ansible](https://docs.ansible.com/ansible/latest/installation_guide/intro_installation.html)

### macOS Installation

```bash
# Install Vagrant
brew tap hashicorp/tap
brew install hashicorp/tap/hashicorp-vagrant

# Install VMware Utility
brew install --cask vagrant-vmware-utility

# Install Vagrant VMware plugin
vagrant plugin install vagrant-vmware-desktop

# Install Ansible
brew install ansible

# Start VMware Utility service
sudo launchctl load -w /Library/LaunchDaemons/com.vagrant.vagrant-vmware-utility.plist
```

## Cluster Architecture

| Node    | IP Address     | Role          | Resources      |
|---------|----------------|---------------|----------------|
| master  | 192.168.56.10  | Control Plane | 2 CPU, 2GB RAM |
| worker1 | 192.168.56.11  | Worker        | 2 CPU, 2GB RAM |
| worker2 | 192.168.56.12  | Worker        | 2 CPU, 2GB RAM |

## Quick Start

### 1. Clone the repository

```bash
git clone <repository-url>
cd vagrant-k8s-cluster
```

### 2. Start the cluster

```bash
vagrant up --provider=vmware_desktop
```

This will provision all 3 VMs and install the required packages via Ansible.

### 3. Initialize the master node

```bash
vagrant ssh master
```

```bash
sudo kubeadm init --apiserver-advertise-address=192.168.56.10 --pod-network-cidr=10.244.0.0/16
```

Save the `kubeadm join` command from the output.

### 4. Configure kubectl

On the master node:

```bash
mkdir -p $HOME/.kube
sudo cp /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

### 5. Install Flannel CNI

```bash
kubectl apply -f https://raw.githubusercontent.com/flannel-io/flannel/master/Documentation/kube-flannel.yml
```

### 6. Join worker nodes

SSH into each worker and run the join command:

```bash
vagrant ssh worker1
sudo kubeadm join 192.168.56.10:6443 --token <TOKEN> --discovery-token-ca-cert-hash <HASH>
```

```bash
vagrant ssh worker2
sudo kubeadm join 192.168.56.10:6443 --token <TOKEN> --discovery-token-ca-cert-hash <HASH>
```

### 7. Verify the cluster

On the master node:

```bash
kubectl get nodes
```

Expected output:

```
NAME      STATUS   ROLES           AGE   VERSION
master    Ready    control-plane   5m    v1.31.x
worker1   Ready    <none>          3m    v1.31.x
worker2   Ready    <none>          2m    v1.31.x
```

## Vagrant Commands

| Command                  | Description                    |
|--------------------------|--------------------------------|
| `vagrant up`             | Start all VMs                  |
| `vagrant halt`           | Stop all VMs                   |
| `vagrant destroy -f`     | Delete all VMs                 |
| `vagrant ssh <node>`     | SSH into a specific node       |
| `vagrant status`         | Show status of all VMs         |
| `vagrant provision`      | Re-run Ansible provisioning    |
| `vagrant reload`         | Restart VMs                    |

## Project Structure

```
vagrant-k8s-cluster/
├── Vagrantfile          # VM definitions and configuration
├── playbook.yml         # Ansible playbook for K8s setup
└── README.md
```

## What the Ansible Playbook Does

- Disables swap (required for Kubernetes)
- Loads required kernel modules (overlay, br_netfilter)
- Configures sysctl for networking
- Installs and configures containerd
- Installs kubeadm, kubelet, and kubectl
- Holds Kubernetes packages at current version
