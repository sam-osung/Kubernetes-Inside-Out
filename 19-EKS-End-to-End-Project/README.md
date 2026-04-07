# 3-Tier Kubernetes Application Deployment on EKS

This README guides you through deploying a **production-ready 3-tier app** (Frontend + Backend + MySQL) on **AWS EKS**, using **NGINX Ingress** and **Cert-Manager** for TLS.

---

## 1️⃣ Create AWS EKS Cluster

```bash
eksctl create cluster \
--name app-cluster \
--region us-east-1 \
--nodegroup-name workers \
--node-type c7i-flex.large \
--nodes 4 \
--managed \
--with-oidc
```

This command will:

Create the EKS control plane

Create a managed node group

Configure OIDC identity provider automatically


```bash
aws eks update-kubeconfig --name app-cluster --region us-east-1
```

---

## 2️⃣ Clone the Repository

```bash
git clone <your-repo-url>
cd <repo-directory>/app-manifests
```

All manifests are in `/app-manifests`:

| File | Purpose |
|-----|-----|
| `mysql-init-configmap.yaml` | Contains MySQL init script to create database and tables on first run |
| `mysql-statefulset.yaml` | Deploys MySQL StatefulSet with PVC for persistent storage |
| `backend-deployment.yaml` | Backend Deployment + ClusterIP Service |
| `frontend-configmap.yaml` | Frontend environment variables (API base URL) |
| `frontend-deployment.yaml` | Frontend Deployment + ClusterIP Service |
| `cluster-issuer.yaml` | Cert-Manager ClusterIssuer (Let's Encrypt) |
| `certificate.yaml` | TLS Certificate using ClusterIssuer |
| `ingress.yaml` | Ingress with TLS for frontend and backend |

---

## 3️⃣ Install the EBS CSI Driver

EKS uses the AWS EBS CSI Driver to dynamically provision volumes from Amazon Elastic Block Store (EBS) for PersistentVolumeClaims.

Without it: PVCs cannot dynamically provision EBS storage.

Install the driver:

```bash
eksctl create addon \
  --name aws-ebs-csi-driver \
  --cluster app-cluster \
  --region us-east-1
```

### Verify the Driver Installation

Check the addon:

```bash
eksctl get addon --cluster app-cluster --region us-east-1
```

Verify the driver pods:

```bash
kubectl get pods -n kube-system | grep ebs
```

Verify the CSI driver is registered:

```bash
kubectl get csidrivers
```

Verify available StorageClasses:

```bash
kubectl get storageclass
```

---

## 4️⃣ Install NGINX Ingress Controller

```bash
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update
helm install ingress-nginx ingress-nginx/ingress-nginx \
  --namespace ingress-nginx \
  --create-namespace
```

Verify pods:

```bash
kubectl get pods -n ingress-nginx
kubectl get svc -n ingress-nginx
```

* Note the **LoadBalancer DNS** for Route 53 setup later.

---

## 5️⃣ Install Cert-Manager

```bash
helm repo add jetstack https://charts.jetstack.io
helm repo update

helm install cert-manager jetstack/cert-manager \
  --namespace cert-manager --create-namespace \
  --version v1.14.2 \
  --set installCRDs=true
```

Verify:

```bash
kubectl get pods -n cert-manager
```

---

## 6️⃣ MySQL StatefulSet and Init Script

The `mysql-init-configmap.yaml` contains:

```sql
USE school;

CREATE TABLE student (
  id INT AUTO_INCREMENT PRIMARY KEY,
  name VARCHAR(40),
  roll_number INT,
  class VARCHAR(16)
);

CREATE TABLE teacher (
  id INT AUTO_INCREMENT PRIMARY KEY,
  name VARCHAR(40),
  subject VARCHAR(40),
  class VARCHAR(16)
);
```

* Runs **once** on first initialization of MySQL PVC.
* Creates `school` database and tables.

Apply StatefulSet:

```bash
kubectl apply -f mysql-statefulset.yaml
```

Check pod:

```bash
kubectl get pods
kubectl exec -it mysql-0 -- mysql -h 127.0.0.1 -u root -p
```

---

## 7️⃣ Backend Deployment

Apply backend Deployment:

```bash
kubectl apply -f backend-deployment.yaml
```

* ClusterIP service exposes backend internally (`backend:3500`).
* Frontend calls backend via Ingress `api.your-domain` path.

---

## 8️⃣ Frontend Deployment

Set environment variables in `frontend-configmap.yaml`:

```env
REACT_APP_API_BASE_URL=https://api.your-domain
```

Apply frontend config and deployment:

```bash
kubectl apply -f frontend-configmap.yaml
kubectl apply -f frontend-deployment.yaml
```

---

## 9️⃣ Cert-Manager ClusterIssuer

`cluster-issuer.yaml` defines Let's Encrypt issuer.

---

## 🔟 TLS Certificate

`certificate.yaml` requests a certificate.

---

## 1️⃣1️⃣ Ingress with TLS

`ingress.yaml` routes:

* `your-domain` → frontend
* `api.yourdomain` → backend

---

## 1️⃣2️⃣ Route 53 DNS Setup

1. Get LoadBalancer DNS:

```bash
kubectl get svc -n ingress-nginx
```

2. Create **Records** in Route 53:

* your-domain        A Record → Ingress LoadBalancer
* api.your-domain   CNAME    → your-domain

After propagation, frontend and backend API are accessible via HTTPS.

---

## 1️⃣3️⃣ Verify Deployment

**MySQL:**

```bash
kubectl exec -it mysql-0 -- mysql -h 127.0.0.1 -u root -p
```

**Backend / Frontend Pods:**

```bash
kubectl get pods
kubectl get svc
```

**Browser:** `https://your-domain` → frontend loads → `api.your-domain` calls reach backend → backend talks to MySQL.

---

## ✅ Summary

Production flow:

create cluster → install storage driver → clone repo → install NGINX ingress → install Cert-Manager → apply manifests → configure DNS

All components are **scalable, secure, and TLS-enabled**.

Change `your-domain` to your real domain.