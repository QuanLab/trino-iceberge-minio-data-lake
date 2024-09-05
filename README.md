# Kubernetes Cluster Installation

This guide provides instructions for setting up a Kubernetes cluster on the following servers:
- **Master Node**: `10.254.59.161`
- **Worker Node 1**: `10.254.59.162`
- **Worker Node 2**: `10.254.59.163`

## Prerequisites

- All nodes should be CentOS-based.
- Ensure the nodes have internet access.
- Run the steps below on all nodes unless specified otherwise.

## Step 1: Disable Swap and Set SELinux to Permissive

On all nodes, disable swap and set SELinux to permissive:

```bash
sudo swapoff -a
sudo setenforce 0
sudo sed -i 's/^SELINUX=enforcing$/SELINUX=permissive/' /etc/selinux/config
```

## Step 2: Install Kubernetes and Docker
Add Kubernetes Repository
On all nodes, add the Kubernetes repository:

```
cat <<EOF | sudo tee /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://pkgs.k8s.io/core:/stable:/v1.30/rpm/
enabled=1
gpgcheck=1
gpgkey=https://pkgs.k8s.io/core:/stable:/v1.30/rpm/repodata/repomd.xml.key
exclude=kubelet kubeadm kubectl cri-tools kubernetes-cni
EOF
```

Install Kubernetes Components
Install `kubelet`, `kubeadm`, and `kubectl` on all nodes:

```
sudo yum install -y kubelet kubeadm kubectl --disableexcludes=kubernetes
```

### Install Docker
Install Docker on all nodes:
```
sudo yum install -y yum-utils
sudo dnf config-manager --add-repo=https://download.docker.com/linux/centos/docker-ce.repo
sudo dnf remove -y runc
sudo dnf install -y docker-ce --nobest
```

### Start Docker
Enable and start Docker on all nodes:

```
sudo systemctl enable docker.service
sudo systemctl start docker.service

```

### Configure Docker
On all nodes, create or modify the Docker daemon configuration file `/etc/docker/daemon.json`:
```
cat <<EOF | sudo tee /etc/docker/daemon.json
{
  "exec-opts": ["native.cgroupdriver=systemd"],
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "100m"
  },
  "storage-driver": "overlay2"
}
EOF
```

### Restart Docker:
```
sudo systemctl restart docker
```

### Configure Networking
Enable IP forwarding:

```
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.ipv4.ip_forward = 1
EOF
sudo sysctl --system
```

## Step 3: Set Hostnames and Update `/etc/hosts`
On each node, set the appropriate hostname and update `/etc/hosts` to map the IP addresses:
```
hostnamectl set-hostname <hostname>
```

Add the following entries to /etc/hosts on each node:

```
10.254.59.161 master
10.254.59.162 worker1
10.254.59.163 worker2
```

## Step 4: Initialize Kubernetes on the Master Node

On the master node (10.254.59.161), pull the Kubernetes images:

```
sudo kubeadm config images pull
```
Initialize the cluster with the following command:

```
sudo kubeadm init --control-plane-endpoint=10.254.59.161 --pod-network-cidr=10.254.0.0/16
```

### Configure kubectl on the Master Node
After the initialization, run the following commands to configure kubectl:

```
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

Alternatively, if you are the root user, you can export the kubeconfig directly:

``` 
export KUBECONFIG=/etc/kubernetes/admin.conf
```

## Step 5: Deploy a Pod Network
Deploy a pod network using a network plugin like Calico, Flannel, or others. For example, to use Calico, run:
```
kubectl apply -f https://docs.projectcalico.org/v3.25/manifests/calico.yaml
```

Refer to the Kubernetes add-ons documentation for other options.

## Step 6: Join Worker Nodes to the Cluster
On each worker node (10.254.59.162 and 10.254.59.163), run the following command to join the cluster:

```
kubeadm join 10.254.59.161:6443 --token <token> \
    --discovery-token-ca-cert-hash sha256:<hash>
```
Replace <token> and <hash> with the values provided during the master node initialization.

## Step 7: Verify Cluster Status
On the master node, verify that all nodes have joined the cluster and are ready:

```
kubectl get nodes
```
You should see the master and worker nodes listed with the status Ready.

# Conclusion
Congratulations! You have successfully set up a Kubernetes cluster with one master and two worker nodes. You can now deploy applications on your cluster.