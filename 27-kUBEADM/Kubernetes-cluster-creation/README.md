# Kubernetes 1.29 Cluster Setup Guide with Calico, MetalLB, NGINX Ingress, Metrics Server, and Helm

## Introduction

This guide walks you through creating a Kubernetes cluster using kubeadm with Calico networking, MetalLB LoadBalancer, NGINX ingress, Metrics Server, and Helm. Every command includes detailed explanations of what it does, why it is needed, and what happens if it is skipped. This ensures beginners can understand every step of the setup process.

---

## 1. Prerequisites and System Preparation

### Disable Swap

```bash
sudo swapoff -a
sudo sed -i '/swap/d' /etc/fstab
```

**Explanation:**

* Kubernetes requires swap to be disabled for proper scheduling and performance.
* If not disabled, kubelet will refuse to start.

### Load Kernel Modules

```bash
sudo modprobe overlay
sudo modprobe br_netfilter
```

**Explanation:**

* Enables overlay networking and bridge traffic for pods.
* Required for container networking to work correctly.

### Persist Kernel Modules

```bash
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF
```

**Explanation:**

* Ensures modules are loaded on boot.

### Apply Sysctl Parameters

```bash
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables=1
net.ipv4.ip_forward=1
net.bridge.bridge-nf-call-ip6tables=1
EOF
sudo sysctl --system
```

**Explanation:**

* Enables packet forwarding and bridge traffic through iptables.
* Without this, pod-to-pod and pod-to-service networking may fail.

### Install Containerd and runc

```bash
sudo apt update
sudo apt install -y containerd
sudo mkdir -p /etc/containerd
containerd config default | sudo tee /etc/containerd/config.toml
sudo sed -i 's/SystemdCgroup = false/SystemdCgroup = true/' /etc/containerd/config.toml
sudo systemctl restart containerd
sudo systemctl enable containerd
```

**Explanation:**

* Containerd is the container runtime required by Kubernetes.
* SystemdCgroup ensures proper resource management by kubelet.

### Install crictl

```bash
VERSION="v1.28.0"
curl -LO https://github.com/kubernetes-sigs/cri-tools/releases/download/$VERSION/crictl-$VERSION-linux-amd64.tar.gz
sudo tar zxvf crictl-$VERSION-linux-amd64.tar.gz -C /usr/local/bin
rm crictl-$VERSION-linux-amd64.tar.gz
```

**Explanation:**

* crictl is used to interact with the container runtime (containerd).
* Useful for troubleshooting pod/container issues.

---

## 2. Install Kubernetes Components (Master Node)

### Add Kubernetes Repository

```bash
sudo mkdir -p /etc/apt/keyrings
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.29/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/k8s.gpg
echo "deb [signed-by=/etc/apt/keyrings/k8s.gpg] https://pkgs.k8s.io/core:/stable:/v1.29/deb/ /" | sudo tee /etc/apt/sources.list.d/kubernetes.list
sudo apt update
```

**Explanation:**

* Adds the official Kubernetes v1.29 repository.
* Ensures we can install specific versions reliably.


### Install Kubernetes Components

* list all available versions of kubeadm

```bash
apt-cache madison kubeadm
```

```bash
sudo apt install -y kubelet=1.29.15-1.1 kubeadm=1.29.15-1.1 kubectl=1.29.15-1.1
sudo apt-mark hold kubelet kubeadm kubectl
```

**Explanation:**

* Installs kubeadm (cluster bootstrap), kubelet (node agent), kubectl (CLI).
* Holding versions prevents accidental upgrades that may break cluster compatibility.

### Verify Installation

```bash
kubeadm version
kubectl version --client
```

**Explanation:**

* Confirms correct versions are installed.

---

## 3. Security Groups (Example for AWS VPC)

### Master Node Security Group Rules

* **Inbound:**

  * 6443 TCP → kube-apiserver (worker to master)
  * 2379-2380 TCP → etcd communication
  * 10250 TCP → kubelet API
  * 10251 TCP → kube-scheduler
  * 10252 TCP → kube-controller-manager
* **Outbound:** Allow all

### Worker Node Security Group Rules

* **Inbound:**

  * 10250 TCP → kubelet API
  * 30000-32767 TCP → NodePort services
* **Outbound:** Allow all

**Explanation:**

* Ensures necessary traffic between master and worker nodes for Kubernetes control plane and pods.

---

## 4. Initialize Kubernetes Master Node

```bash
sudo kubeadm init --apiserver-advertise-address=<MASTER_NODE_PRIVATE_IP> --pod-network-cidr=192.168.0.0/16
```

**Explanation:**

* Boots up the control plane on the master node.
* `--pod-network-cidr` must match the CNI plugin (Calico uses 192.168.0.0/16).
* Without this, pods will not be able to communicate across nodes.

### Configure kubectl for Master Node

```bash
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

**Explanation:**

* Sets up CLI access for the admin user.
* Without this, `kubectl` commands will fail.

---

## 5. Install Calico CNI

```bash
kubectl apply -f https://raw.githubusercontent.com/projectcalico/calico/v3.26.0/manifests/calico.yaml
```

**Explanation:**

* Calico provides pod networking, network policies, and IP management.
* Without a CNI, pods cannot communicate across nodes.

---

## 6. Join Worker Nodes

**Repeat the worker node preparation** (disable swap, load modules, install containerd, kubeadm/kubelet/kubectl **same version**). Then run:

```bash
sudo kubeadm join <MASTER_NODE_PRIVATE_IP>:6443 --token <TOKEN> --discovery-token-ca-cert-hash sha256:<HASH>
```

**Explanation:**

* Adds worker nodes to the cluster.
* Versions must match the master node.
* Without joining, nodes cannot schedule pods.

---

## 7. MetalLB LoadBalancer

### Install MetalLB

```bash
kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.14.5/config/manifests/metallb-native.yaml
```

### Confirm Pods are running

```bash
kubectl get pods -n metallb-system
```

### Configure IP Pool  (Pick a small unused range inside the 172.31.x.x network.)

```bash
cat <<EOF | kubectl apply -f -
apiVersion: metallb.io/v1beta1
kind: IPAddressPool
metadata:
  name: my-ip-pool
  namespace: metallb-system
spec:
  addresses:
  - 172.31.255.200-172.31.255.210
EOF
```
### Enable L2 advertisement

```bash
cat <<EOF | kubectl apply -f -
apiVersion: metallb.io/v1beta1
kind: L2Advertisement
metadata:
  name: l2-advert
  namespace: metallb-system
EOF
```

**Explanation:**

* Provides LoadBalancer IPs for services.
* Without MetalLB, LoadBalancer services remain pending.

---

## 8. NGINX Ingress Controller

```bash
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/main/deploy/static/provider/cloud/deploy.yaml
kubectl get pods -n ingress-nginx
kubectl get svc -n ingress-nginx
```

**Explanation:**

* Handles HTTP/S routing to services.
* Without ingress, external traffic cannot reach cluster services.

---

## 9. Metrics Server

```bash
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml
kubectl get deployment metrics-server -n kube-system
kubectl top nodes
kubectl top pods -A
```

**Explanation:**

* Collects CPU and memory metrics.
* Required for `kubectl top` and HPA.
* Without it, autoscaling cannot work.

---

## 10. Helm Installation

```bash
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
helm version
helm repo add stable https://charts.helm.sh/stable
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update
helm install my-nginx ingress-nginx/ingress-nginx -n ingress-nginx
```

**Explanation:**

* Helm simplifies app deployment and management.
* Without Helm, deployments require manual YAML management.

---

## Notes on Upgrading Kubernetes (e.g., v1.30)

* Change repository URL to `https://pkgs.k8s.io/core:/stable:/v1.30/deb/`
* Update version numbers in install commands, e.g., `1.30.x-00`
* Ensure kubeadm init, kubelet, and kubeadm versions **match** across master and worker nodes.
* Re-run `kubeadm upgrade plan` and `kubeadm upgrade apply <version>` for upgrades.

---

## ✅ Summary

* Master initialized with kubeadm, Calico CNI applied
* Worker nodes joined
* MetalLB providing LoadBalancer IPs
* NGINX ingress controller installed
* Metrics server installed
* Helm installed for package management

Cluster is now fully operational and ready for deployments.
