# Kubernetes Resource Quotas – Beginner-Friendly Deep & Practical Guide

Resource Quotas in Kubernetes allow you to control how much CPU, memory, and other resources a namespace can consume. This guide walks you through creating quotas, applying them to deployments, and understanding quota exceeded errors.

---

# Why Resource Quotas Matter

In a shared cluster:

- Multiple teams may deploy applications simultaneously  
- One team could accidentally request too many resources  
- This can **starve other workloads** and affect cluster stability  

ResourceQuotas act as a **control layer** at the **namespace level**, ensuring fair resource usage and preventing resource starvation.

---

# What You Will Learn

By the end of this guide, you will know how to:

- Set ResourceQuotas for a namespace  
- Limit CPU, memory, number of Pods, PVCs, and other objects  
- Define `requests` and `limits` inside containers  
- Interpret quota exceeded errors and fix them  

---

# Part 1 – Requests vs Limits

## Requests

- Minimum resources a container needs to run  
- Used by the scheduler for placement  

Example:

```yaml
resources:
  requests:
    cpu: 100m
    memory: 128Mi
```

## Limits

- Maximum resources a container is allowed to use  
- CPU exceeding → throttled  
- Memory exceeding → container killed (OOMKilled)  

Example:

```yaml
resources:
  limits:
    cpu: 500m
    memory: 256Mi
```

**Units recap**:

| Resource | Example |
|----------|---------|
| CPU      | 1 CPU = 1000m, 500m = 0.5 CPU |
| Memory   | 128Mi, 256Mi, 1Gi |

---

# Part 2 – Create a Namespace for dev

Create a new namespace:

```bash
kubectl create namespace dev
```

✅ This is a logical boundary where quotas will be applied.

---

# Part 3 – Create a ResourceQuota

Apply the quota manifest:

```bash
kubectl apply -f dev-quota.yaml
```

Verify:

```bash
kubectl get resourcequota -n quota-demo
kubectl describe resourcequota dev-quota  -n dev
```

**Quota Details**:

- Limits total CPU and memory requests  
- Limits total CPU and memory usage  
- Limits number of pods and PVCs  

**Example Hard Limits in `dev-quota.yaml`**:

```yaml
requests.cpu: "500m"
requests.memory: "512Mi"
limits.cpu: "1"
limits.memory: "1Gi"
pods: "3"
persistentvolumeclaims: "2"
```

> Quotas are **strict limits** at the namespace level, not per pod.

---

# Part 4 – Create a Deployment with Requests and Limits

Apply the deployment:

```bash
kubectl apply -f nginx-deploy.yaml
```

Verify pods:

```bash
kubectl get pods -n dev
```

Check quota usage:

```bash
kubectl describe resourcequota dev-quota -n dev
```

**Observation**:

- Requests and limits from all pods are summed  
- Quota ensures **total usage does not exceed limits**

---

# Part 5 – Trigger Quota Exceeded Error

Increase deployment replicas to exceed quota:

```bash
# Edit nginx-deploy.yaml
replicas: 3
kubectl apply -f nginx-deploy.yaml
```

Check pods:

```bash
kubectl get pods -n dev
kubectl describe deployment nginx-deploy -n dev
```

You will see errors like:

```
exceeded quota: demo-quota, requested: requests.cpu=200m
```

**Important**:

- Quota errors **do not crash the cluster**  
- They prevent object creation until usage is reduced or quota is increased

---

# Part 6 – Fixing Quota Exceeded Errors

Option 1 – Reduce workload:

- Reduce replicas, requests, or limits in your deployment  
- Re-apply manifest:

```bash
kubectl apply -f nginx-deploy.yaml
```

Option 2 – Adjust the quota:

```bash
kubectl edit resourcequota dev-quota -n dev
```

- Increase allowed resources, then apply deployment again  

---

# Part 7 – Why ResourceQuotas Prevent Starvation

Without quotas:

- One namespace can consume all resources  
- Other teams suffer  

With quotas:

- Boundaries are guaranteed  
- Cluster usage becomes predictable  
- Platform teams enforce fair usage  

---

# Part 8 – Object Quotas (Pods, Services, ConfigMaps, Secrets)

ResourceQuotas can also limit object counts:

```yaml
pods: "3"
services: "2"
configmaps: "5"
secrets: "5"
```

Benefits:

- Prevent runaway automation  
- Maintain cluster hygiene  
- Not limited to CPU and memory  

---

# Part 9 – Final Summary

- ResourceQuotas **work at namespace level**  
- Limit **resource usage** and **object counts**  
- Requests → used for scheduling and quota accounting  
- Limits → runtime enforcement  
- Quotas prevent resource starvation and ensure fairness  
- Quota exceeded → reduce usage or increase quota  

✅ Applying quotas correctly helps you **think like a real Kubernetes platform engineer**, not just a YAML copier.

---

You can also experiment quota with **pod-fail.yaml, pod-fit.yaml & limit-range.yaml**.
