# Kubernetes ServiceAccounts & RBAC – Beginner-Friendly Practical Guide

This guide covers **ServiceAccounts**, **pod identity**, and **RBAC** in Kubernetes. It explains how pods authenticate, how access is controlled, and how RBAC permissions work in real clusters.

---

# Why ServiceAccounts Matter

- Pods need an **identity** to talk to the Kubernetes API.  
- Unlike users who use `kubeconfig`, pods authenticate with **ServiceAccounts**.  
- ServiceAccount provides **authentication only**, not permissions.  
- Without it, pods cannot list resources, read secrets, or interact with the API.

---

# Understanding Pod Identity

- **Default ServiceAccount:** Every pod runs as `default` unless specified.  
- **Token Mounting:** Kubernetes automatically mounts a token inside `/var/run/secrets/kubernetes.io/serviceaccount/token`.  
- **Pod uses this token** to authenticate with the API server.

Check default ServiceAccounts:

```bash
kubectl get serviceaccounts -n default
kubectl describe serviceaccount default -n default
```

---

# Creating Your Own ServiceAccount

Apply the YAML:

```bash
kubectl apply -f sa.yaml
```

Verify:

```bash
kubectl get serviceaccounts -n dev
```

---

# Attaching a ServiceAccount to a Pod

Create a pod YAML with your ServiceAccount:

```bash
kubectl apply -f pod-with-sa.yaml
kubectl describe pod sa-demo-2 -n dev
```

Inside the pod:

```bash
kubectl exec -it sa-demo-2 -n dev -- /bin/sh
ls /var/run/secrets/kubernetes.io/serviceaccount
cat /var/run/secrets/kubernetes.io/serviceaccount/token
```

> ✅ Each ServiceAccount gets its **own token** mounted automatically.

---

# Testing API Access from Pod

Inside the pod:

```bash
apk add curl
TOKEN=$(cat /var/run/secrets/kubernetes.io/serviceaccount/token)
curl -k -H "Authorization: Bearer $TOKEN" \
  https://$KUBERNETES_SERVICE_HOST:$KUBERNETES_SERVICE_PORT/api/v1/pods
```

Most likely:

```
Forbidden
User "system:serviceaccount:dev:app-sa" cannot list resource "pods"
```

**Why?**  
ServiceAccount provides identity, **but no permissions yet**.

---

# RBAC Overview

Kubernetes security consists of two parts:

| Aspect           | Responsible For                   |
|-----------------|-----------------------------------|
| Authentication   | Who you are (ServiceAccount, kubeconfig, certificates, SSO) |
| Authorization    | What you can do (RBAC)           |

**ServiceAccount = authentication for pods**  
**RBAC = authorization for access control**

---

# Role – Namespace Scoped Permissions

Roles define **permissions inside a namespace**. They do **not grant access by themselves**.  

Apply Role:

```bash
kubectl apply -f role.yaml
```

Check your role:

```bash
kubectl get role -n dev
```

---

# RoleBinding – Attach Role to a ServiceAccount

Bind your ServiceAccount to the Role:

```bash
kubectl apply -f rolebinding.yaml
```

Check the binding:

```bash
kubectl get rolebinding -n dev
kubectl describe rolebinding read-pods-binding -n dev
```

---

# Verify Pod Access After RBAC

Inside the pod:

```bash
TOKEN=$(cat /var/run/secrets/kubernetes.io/serviceaccount/token)
curl -k -H "Authorization: Bearer $TOKEN" \
  https://$KUBERNETES_SERVICE_HOST:$KUBERNETES_SERVICE_PORT/api/v1/pods
```
or do:

```bash
kubectl auth can-i list pods --as=system:serviceaccount:dev:app-sa -n dev
```

> ✅ Now the pod can **list pods** in the `dev` namespace.  

Try other namespaces:

```bash
kubectl auth can-i list pods --as=system:serviceaccount:dev:app-sa -n (other-namespace)
```

> ❌ Access denied. RoleBinding only applies to **its namespace**.

---

# ClusterRole – Cluster-Wide Permissions

ClusterRoles define permissions **not tied to a namespace**.  

Apply ClusterRole:

```bash
kubectl apply -f clusterrole.yaml
```

- Typically used for resources like nodes, cluster-wide pods, or monitoring.

---

# ClusterRoleBinding – Attach ClusterRole Globally

Apply ClusterRoleBinding:

```bash
kubectl apply -f clusterrolebinding.yaml
```

- Gives ServiceAccount permissions **across all namespaces**  
- ⚠ Use carefully: breaks namespace isolation

---

# Key RBAC Concepts

| Field        | Meaning |
|-------------|---------|
| verbs       | Actions allowed (get, list, create, delete, patch, update) |
| resources   | Kubernetes objects (pods, secrets, deployments) |
| apiGroups   | API group the resource belongs to: <br>- Core: `""` <br>- Apps: `apps` <br>- Batch: `batch` <br>- Networking: `networking.k8s.io` |

---

# Role vs ClusterRole Recap

| Object           | Scope                        | Usage |
|-----------------|-------------------------------|-------|
| Role             | Namespace                     | Bind to SA or user inside that namespace |
| ClusterRole      | Cluster-wide                  | Bind to SA or user across all namespaces |
| RoleBinding      | Namespace                     | Attach Role or ClusterRole inside namespace |
| ClusterRoleBinding | Cluster-wide                | Attach ClusterRole across cluster |

---

# ServiceAccount vs User

| Identity        | Who it is              | Authentication | Authorization |
|-----------------|-----------------------|----------------|---------------|
| ServiceAccount  | Pod identity          | Token          | RBAC          |
| User / Group    | Human identity        | Kubeconfig / SSO | RBAC        |

---

# Complete Security Flow

**For Pods:**

```
Pod → ServiceAccount → Token → API Server → RBAC → Access Granted / Denied
```

**For Humans:**

```
kubectl → kubeconfig → Certificate / SSO → API Server → RBAC → Access Granted / Denied
```

> RBAC engine is the same, only identity differs.

---


---

✅ **Notes for Beginners:**

- **ServiceAccount** = pod identity (authentication)  
- **RBAC** = permission enforcement (authorization)  
- **Role/ClusterRole** = define permissions  
- **RoleBinding/ClusterRoleBinding** = attach permissions to identity  
- **Namespace isolation** = true boundary; ClusterRoleBinding bypasses it  
- **API groups** = tell Kubernetes which resources belong where  

---
