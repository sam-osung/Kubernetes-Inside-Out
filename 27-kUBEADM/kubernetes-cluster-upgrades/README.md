# Kubernetes Cluster Upgrade Guide

## Overview

Upgrading a Kubernetes cluster is a critical operational task that ensures access to new features, security patches, and performance improvements. This guide provides a structured, production-ready approach to upgrading Kubernetes clusters using `kubeadm`.

---

## Kubernetes Versioning Explained

Kubernetes follows semantic versioning:

```
<major>.<minor>.<patch>
Example: 1.30.2
```

* **Major**: Significant architectural changes (rare)
* **Minor**: New features (released every ~3 months)
* **Patch**: Bug fixes and security updates (frequent)

### Upgrade Rules

* Upgrade **one minor version at a time** (e.g., 1.28 → 1.29 → 1.30)
* Skipping minor versions is **not supported**
* Patch upgrades are flexible and safe

---

## Version Skew Policy (Critical)

Kubernetes components must remain within supported version ranges:

| Component          | Supported Skew    |
| ------------------ | ----------------- |
| kube-apiserver     | Reference         |
| controller-manager | ≤ 1 minor behind  |
| scheduler          | ≤ 1 minor behind  |
| kubelet            | ≤ 2 minors behind |
| kubectl            | ≤ 2 minors behind |

---

## Upgrade Strategies

### 1. In-Place Upgrade (Not Recommended for Production)

* Upgrade all nodes at once
* Causes downtime

### 2. Rolling Upgrade (Recommended)

* Upgrade nodes one at a time
* Maintains availability

### 3. Blue-Green Deployment

* Create a new cluster
* Migrate workloads
* Higher cost but safest

---

## Pre-Upgrade Checklist

Before upgrading:

* Backup etcd:

  ```bash
  ETCDCTL_API=3 etcdctl snapshot save snapshot.db
  ```

* Verify cluster health:

  ```bash
  kubectl get nodes
  kubectl get pods -A
  ```

* Check OS version:

  ```bash
  cat /etc/os-release
  ```

* Ensure access to control plane config:

  ```bash
  /etc/kubernetes/admin.conf
  ```

---

## Step 1: Update Package Repository

Check current repo:

```bash
cat /etc/apt/sources.list.d/kubernetes.list
```

Example:

```bash
deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] \
https://pkgs.k8s.io/core:/stable:/v1.29/deb/ /
```

Update to next version:

```bash
nano /etc/apt/sources.list.d/kubernetes.list
```

Change to:

```bash
deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] \
https://pkgs.k8s.io/core:/stable:/v1.30/deb/ /
```

---

## Step 2: Upgrade Control Plane

### Upgrade kubeadm

```bash
sudo apt-mark unhold kubeadm
sudo apt-get update
sudo apt-get install -y kubeadm='1.30.x-*'
sudo apt-mark hold kubeadm
```

Verify version:

```bash
kubeadm version
```

### Plan Upgrade

```bash
kubeadm upgrade plan
```

### Check Available Patch Versions

```bash
sudo apt-cache madison kubeadm
```

### Apply Upgrade

```bash
sudo kubeadm upgrade apply v1.30.x
```

> Note: Certificates are automatically renewed unless disabled.

---

## Step 3: Upgrade Control Plane Node Components

### Drain Node

```bash
kubectl drain <node-name> --ignore-daemonsets
```

### Upgrade kubelet & kubectl

```bash
sudo apt-mark unhold kubelet kubectl
sudo apt-get update
sudo apt-get install -y kubelet='1.30.x-*' kubectl='1.30.x-*'
sudo apt-mark hold kubelet kubectl
```

### Restart kubelet

```bash
sudo systemctl daemon-reload
sudo systemctl restart kubelet
```

### Uncordon Node

```bash
kubectl uncordon <node-name>
```

---

## Step 4: Upgrade Worker Nodes

> Perform this **one node at a time** to maintain availability.

### Update Repository

```bash
nano /etc/apt/sources.list.d/kubernetes.list
```

Update version to target release.

### Upgrade kubeadm

```bash
sudo apt-mark unhold kubeadm
sudo apt-get update
sudo apt-get install -y kubeadm='1.30.x-*'
sudo apt-mark hold kubeadm
```

### Upgrade Node Configuration

```bash
sudo kubeadm upgrade node
```

### Drain Node (from control plane)

```bash
kubectl drain <node-name> --ignore-daemonsets
```

### Upgrade kubelet & kubectl

```bash
sudo apt-mark unhold kubelet kubectl
sudo apt-get update
sudo apt-get install -y kubelet='1.30.x-*' kubectl='1.30.x-*'
sudo apt-mark hold kubelet kubectl
```

### Restart kubelet

```bash
sudo systemctl daemon-reload
sudo systemctl restart kubelet
```

### Uncordon Node

```bash
kubectl uncordon <node-name>
```

---

## Step 5: Upgrade CNI Plugin

CNI plugins are not upgraded automatically.

Example (Calico):

```bash
kubectl apply -f https://docs.projectcalico.org/manifests/calico.yaml
```

Ensure compatibility with Kubernetes version.

---

## Post-Upgrade Validation

* Check nodes:

  ```bash
  kubectl get nodes
  ```

* Check system pods:

  ```bash
  kubectl get pods -n kube-system
  ```

* Verify cluster version:

  ```bash
  kubectl version
  ```

---

## Rollback Strategy (Important)

If upgrade fails:

1. Restore etcd snapshot
2. Downgrade kubeadm, kubelet, kubectl
3. Restart services

> Note: Kubernetes rollback is not always straightforward — backups are critical.

---

## Best Practices

* Always test upgrades in staging first
* Monitor workloads during upgrade
* Upgrade during low traffic periods
* Automate with CI/CD where possible
* Keep documentation updated

---

## Summary

* Upgrade **one minor version at a time**
* Follow **version skew policy**
* Use **rolling upgrades for production**
* Always **backup etcd before upgrading**

---

## References

* [https://kubernetes.io/docs/tasks/administer-cluster/kubeadm/kubeadm-upgrade/](https://kubernetes.io/docs/tasks/administer-cluster/kubeadm/kubeadm-upgrade/)

---

