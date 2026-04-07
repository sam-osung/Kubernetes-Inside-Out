# Kubernetes Troubleshooting for Beginners – Practical Guide

This guide is designed for **hands-on learning** and beginner-friendly debugging. It covers common pod issues:

- `ImagePullBackOff`  
- `CrashLoopBackOff`  
- `OOMKilled`  
- Node taints and tolerations  
- Node affinity  
- Pod scheduling problems  

It gives you **all the commands you need** to reproduce, debug, and fix problems.

---

## Learning Goals

By the end of this guide, you will be able to:

- Debug **ImagePullBackOff**  
- Debug **CrashLoopBackOff**  
- Debug **OOMKilled**  
- Understand **Node taints and Tolerations**  
- Understand **Node affinity and labels**  
- Fix **Pending pods** due to scheduling issues  

---

## Primary Tools

```bash
kubectl get pods
kubectl describe pod <pod-name>
kubectl logs <pod-name>
kubectl logs <pod-name> --previous
kubectl exec -it <pod-name> -- sh
```

> `kubectl describe pod` is your best friend. Always read the **Events** section carefully — it usually tells exactly what went wrong.

---

## 1️⃣ ImagePullBackOff

**Meaning:** Kubernetes cannot pull the container image.

```bash
kubectl apply -f bad-image.yaml
kubectl get pods
kubectl describe pod bad-image
```

**Common Causes & Fixes:**

- Wrong image or tag → correct the `image:` field  
- Private registry → create a pull secret:

```bash
kubectl create secret docker-registry regcred \
  --docker-server=<registry-url> \
  --docker-username=<username> \
  --docker-password=<password>
```

Reference the secret in your pod:

```yaml
imagePullSecrets:
- name: regcred
```

> Beginner Tip: The **Events** section almost always tells the exact problem.

---

## 2️⃣ CrashLoopBackOff

**Meaning:** Container starts and immediately exits repeatedly.

```bash
kubectl apply -f crash-demo.yaml
kubectl get pods
kubectl logs crash-demo
kubectl logs crash-demo --previous
```

**Debug Trick:** Temporarily override the container command to inspect:

```yaml
command: ["sh", "-c", "sleep 3600"]
```

```bash
kubectl exec -it crash-demo -- sh
```

> Usually a **problem with the application**, not Kubernetes itself.

---

## 3️⃣ OOMKilled – Out of Memory

**Meaning:** Container exceeded memory limit.

```bash
kubectl apply -f oom-demo.yaml
kubectl describe pod oom-demo
```

**Fixes:**

- Increase `resources.limits.memory`  
- Optimize application memory usage  
- Always set `requests` and `limits`:

```yaml
resources:
  requests:
    memory: 100Mi
  limits:
    memory: 300Mi
```

> Requests help the scheduler place pods; limits enforce runtime memory usage.

---

## 4️⃣ Pending Pods – Scheduling Problems

Pods stay `Pending` if the scheduler cannot find a suitable node.

### 4.1 Node Taints

**Taints:** Mark a node so that only pods that tolerate the taint can schedule there.

```bash
kubectl get nodes
kubectl taint nodes <node-name> dedicated=infra:NoSchedule
kubectl describe node <node-name>
```

Pod without toleration will **stay Pending**:

```bash
kubectl apply -f normal-pod.yaml
```

**Add Toleration in the Pod:**

```yaml
spec:
  tolerations:
  - key: "dedicated"
    operator: "Equal"
    value: "infra"
    effect: "NoSchedule"
```

```bash
kubectl apply -f taint-test.yaml
```

> Tolerations **allow pods to schedule on tainted nodes**; they don’t force it.

---

### 4.2 Node Affinity

**Node Affinity:** Schedule pods based on node labels.

Label a node:

```bash
kubectl label node <node-name> disktype=ssd
kubectl get nodes --show-labels
```

```bash
kubectl apply -f affinity-demo.yaml
```

> Pod will only schedule on nodes labeled `disktype=ssd`.

---

### 4.3 Taints vs Affinity

| Concept                | Behavior                            |
|------------------------|-------------------------------------|
| Taints & Tolerations   | Nodes repel pods unless tolerated   |
| Node Affinity          | Pods select nodes to prefer/require |

> Taints protect nodes; affinity guides pod placement.

---

## 5️⃣ Other Scheduling Issues

- **Insufficient CPU/Memory:** Reduce `requests` or add nodes  
- **Pod stuck on PVC:** Check StorageClass & PVC status  
- **Probe failures:** Increase `initialDelaySeconds` or correct paths  

---


## Real Troubleshooting Flow

1. Check pod status:

```bash
kubectl get pods
```

2. Describe pod:

```bash
kubectl describe pod <pod-name>
```

3. Check logs:

```bash
kubectl logs <pod-name>
kubectl logs <pod-name> --previous
```

4. Exec into pod if running:

```bash
kubectl exec -it <pod-name> -- sh
```

5. Inspect node taints, labels, resource usage, and events.

---

## ✅ Summary

| Issue                | Root Cause                     |
|----------------------|--------------------------------|
| ImagePullBackOff     | Image or registry problem      |
| CrashLoopBackOff     | Application startup problem    |
| OOMKilled            | Memory exceeded                |
| Pending pods         | Scheduling issues: taints, tolerations, affinity, resource requests, volumes |

> Your best friend is always:

```bash
kubectl describe pod <pod-name>
```

> Always read **Events** carefully — Kubernetes will tell you exactly what went wrong.