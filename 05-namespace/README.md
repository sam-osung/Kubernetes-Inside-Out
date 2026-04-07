# Kubernetes Namespaces – Logical Separation & Environment Isolation

This guide explains why Kubernetes namespaces exist and how to use them to achieve logical separation and environment isolation.

---

# Why Namespaces Matter

Namespaces provide:

- Isolation – workloads in one namespace don’t interfere with another
- Organization – group resources by team, project, or environment
- Resource quotas – control CPU, memory, and storage per namespace
- Access control – apply RBAC rules per namespace

Default scenarios:

- Dev environment: `dev` namespace
- QA environment: `qa` namespace
- Production: `prod` namespace

**Key takeaway:** Namespaces are virtual clusters inside a physical cluster.

---

# Creating Namespaces

## Imperative Method

```bash
kubectl create namespace dev
kubectl get namespaces
```

## Declarative Method

Apply pre-created manifests:

```bash
kubectl apply -f prod-namespace.yaml
kubectl get namespaces
```

> You will now see `default`, `kube-system`, `kube-public`, `dev`, and `prod`.

---

# Running Pods in Namespaces

### Imperative Method

```bash
kubectl run dev-nginx --image=nginx --restart=Never -n dev
kubectl get pods -n dev
kubectl get pods -n prod
```

### Declarative Method

```bash
kubectl apply -f nginx-pod-dev.yaml
kubectl get pods -n dev
kubectl get pods -n prod
```

> Pods in `dev` namespace are isolated from `prod`.

---

# Executing Commands Inside Pods

```bash
kubectl exec -it dev-nginx -n dev -- /bin/bash
# Inside container:
hostname
exit
```

---

# Deleting Pods and Namespaces

Delete a Pod:

```bash
kubectl delete pod dev-nginx -n dev
```

Delete the namespace entirely:

```bash
kubectl delete namespace dev
```

> Deleting a namespace deletes all resources inside it.

---

# Summary

Namespaces provide:

- Logical separation
- Environment isolation
- Organizational clarity

Best practices:

- Use `-n` or `--namespace` to target a specific namespace
- Isolate dev, QA, and production environments
- All Kubernetes resources can live inside namespaces for better management