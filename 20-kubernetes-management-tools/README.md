# Kubernetes Management Tools – Beginner-Friendly Guide

Welcome! This guide covers **Kubernetes management tools** that make cluster monitoring, debugging, and day-to-day operations easier. While `kubectl` is powerful, using a management tool with a graphical interface improves visibility and reduces errors.

---

## 🎯 Learning Goals

By the end of this guide, you should be able to:

- Use Portainer to manage Kubernetes clusters via a web UI  
- Use Lens/OpenLens to manage clusters on your desktop  
- Compare tools and choose the best for your workflow  
- Understand how GUI-based management improves cluster visibility  

---

## ⚡ Why Use Kubernetes Management Tools?

Kubernetes clusters can be complex, with multiple nodes, pods, and services running simultaneously. Management tools provide:

- Real-time **cluster health visualization**  
- **Resource usage monitoring**  
- Easy debugging of failed pods and deployments  
- Deployment and service management without typing long CLI commands  

Popular options include **Portainer, Lens, OpenLens, Rancher, and K9s**.  

---

## 1️⃣ Portainer

Portainer is a **web-based Kubernetes and Docker management tool**. It allows you to manage clusters from a single interface.

**Key Features:**

- Deploy applications using a GUI  
- Monitor container health and logs  
- Role-based access control for teams  
- Multi-cluster management  

**Steps to Use Portainer:**

1. Install Portainer as a **pod in your Kubernetes cluster** or as a **standalone Docker container**.  
   [Official Installation Guide](https://docs.portainer.io/start/install)  

2. Open your browser and navigate to the Portainer URL.  

3. Create an **admin user** and log in.  

4. Add your Kubernetes cluster by providing **kubeconfig** or connecting via the Kubernetes API.  

5. Explore your cluster visually: check **pod status, deployments, services, and node resources**.  

6. Use the **deployment wizard** to deploy or update workloads without CLI commands.  

> Portainer is ideal for beginners or teams who want simplicity with access to advanced Kubernetes features.


# Setting Up Portainer on AWS EKS

This guide provides a full, production-ready setup of Portainer Community Edition (CE) on AWS EKS, including gp2/gp3 persistent storage, LoadBalancer exposure, TLS, and Helm installation.

---

## 1️⃣ Create an EKS Cluster with OIDC Enabled

```bash
eksctl create cluster \
  --name app-cluster \
  --region us-east-1 \
  --version 1.34 \
  --with-oidc \
  --managed
```

Check cluster:

```bash
kubectl get nodes
kubectl get sc
```

---

## 2️⃣ Install AWS EBS CSI Driver

```bash
eksctl create addon \
  --name aws-ebs-csi-driver \
  --cluster app-cluster \
  --region us-east-1
```

Check StorageClass:

```bash
kubectl get sc
```

(Optional) Set storageclass as default:

```bash
kubectl patch storageclass gp2 -p '{"metadata": {"annotations":{"storageclass.kubernetes.io/is-default-class":"true"}}}'
```

---

## 3️⃣ Install Portainer Using Helm

Add the Helm repository and update:

```bash
helm repo add portainer https://portainer.github.io/k8s/
helm repo update
```

Install Portainer:

```bash
helm install portainer portainer/portainer \
  --namespace portainer \
  --create-namespace
```

Check pods and PVC:

```bash
kubectl get pods -n portainer
kubectl get pvc -n portainer
```

---

## 4️⃣ Upgrade Portainer for Production-Ready Setup - with LoadBalancer

```bash
helm upgrade --install --create-namespace -n portainer portainer portainer/portainer \
  --set service.type=LoadBalancer \
  --set tls.force=true \
  --set image.tag=lts
```

Watch pods and service:

```bash
kubectl get pods -n portainer -w
kubectl get svc -n portainer -w
```

---

## 5️⃣ Access Portainer

Get the LoadBalancer URL:

```bash
kubectl get svc -n portainer
```

Open in browser:

```text
https://<EXTERNAL-IP>:9443
```

* Accept self-signed TLS certificate
* Login with admin user

---

## 6️⃣ Add Kubernetes Cluster to Portainer

* Choose **Kubernetes environment**
* Portainer Agent gathers cluster metrics and manages workloads

---

## 7️⃣ Manage and Restart Portainer

Restart safely:

```bash
kubectl rollout restart deployment portainer -n portainer
```

Check logs:

```bash
kubectl logs -n portainer -l app.kubernetes.io/name=portainer
```

PVC ensures persistent data remains intact.

---

## 8️⃣ Optional Enhancements

* Use **Ingress Controller + cert-manager** for a custom domain and valid TLS
* Switch PVC to **gp3** for better IOPS/performance
* Enable **Role

---

## 2️⃣ Lens & OpenLens

Lens is known as the **Kubernetes IDE**. It’s a **desktop application** that allows you to:

- Visualize cluster topology  
- Monitor logs, events, and metrics  
- Manage multiple clusters from one interface  

**Important:** Lens recently moved to a **paid model**, but **OpenLens** provides the original Lens experience for free.

### OpenLens

OpenLens is a **free, community-maintained fork of Lens**:

- Cross-platform desktop IDE  
- Fully functional cluster management  
- Monitors logs, events, and resource usage  

**Download:** [OpenLens Releases](https://github.com/MuhammedKalkan/OpenLens/releases)

**Steps to Use OpenLens:**

1. Download the binary for your OS.  

2. Launch the application – no installation required.  

3. Add your Kubernetes cluster by importing your **kubeconfig**.  

4. Explore your cluster: view **nodes, pods, deployments, and services**.  

5. Manage multiple clusters simultaneously and switch between them easily.  

> OpenLens is perfect for students and admins wanting the Lens experience for free.

---

## 3️⃣ Tool Comparison

| Tool       | Type          | Free | Notes |
|------------|--------------|------|-------|
| Portainer  | Web GUI      | ✅   | Beginner-friendly, manages Docker & Kubernetes, great for teams |
| Lens       | Desktop IDE  | ❌   | Paid now, advanced cluster management |
| OpenLens   | Desktop IDE  | ✅   | Free, community fork of Lens, fully functional |
| K9s        | CLI          | ✅   | Powerful for advanced users preferring terminal management |

**Recommendation for Beginners:**

- **Portainer** → Simple web GUI  
- **OpenLens** → Desktop IDE experience for free  
- **K9s** → CLI-based management for advanced users  

---

## ✅ Summary

Kubernetes management tools:

- Simplify cluster administration  
- Improve visibility into cluster health  
- Reduce errors and save time  
- Make multi-cluster management easier  

> Practice by deploying Portainer or using OpenLens on a test cluster. Explore the UI, monitor resources, and try deploying, scaling, and debugging pods to see which tool fits your workflow best.

---
