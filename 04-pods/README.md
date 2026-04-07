# Kubernetes Pods – Concepts, Single vs Multi-container, Imperative & Declarative Creation

This module explains Pods, the smallest unit in Kubernetes, and how to create them using imperative and declarative methods.

---

# Why Pods Matter

- Kubernetes does **not deploy containers directly**.
- Kubernetes deploys **Pods**, which wrap one or more containers.
- Flow: `image → container → Pod → Kubernetes`.
- Containers always run **inside Pods**, never directly in Kubernetes.

> **Key concept:** Pods are the smallest deployable units and represent a single logical application unit. Containers in the same Pod share networking and storage.

---

# Single-container vs Multi-container Pods

✅ **Single-container Pod**

- Most common scenario
- Example: one nginx container or one backend API container
- Simple and straightforward

✅ **Multi-container Pod**

- Runs multiple tightly-coupled containers
- Containers share:
  - Network namespace (same IP)
  - Storage volumes
  - Lifecycle (scheduled and terminated together)
- Use cases: sidecar containers for logging, metrics, proxies, etc.
- **Important:** Do not use Pods like Docker Compose; multi-container Pods are for tightly-coupled containers only.

---

# Creating Pods in Kubernetes

There are two approaches:

1. **Imperative** – create resources directly via `kubectl` commands
2. **Declarative** – define resources in YAML files and apply them

> In production, declarative is preferred; imperative is useful for learning.

---

## Part 1 – Imperative Pod Creation

Create a simple nginx Pod:

```bash
kubectl run nginx-pod --image=nginx --restart=Never
kubectl get pods
kubectl describe pod nginx-pod
```

- `--restart=Never` ensures this is a **raw Pod**, not a Deployment.
- `kubectl describe` shows events, container info, IP address, and node placement.

### Exec into the Pod

```bash
kubectl exec -it nginx-pod -- /bin/bash
# inside the container:
hostname
exit
```

### Delete the Pod

```bash
kubectl delete pod nginx-pod
```

> Imperative Pods are **not self-healing**.

---

## Part 2 – Declarative Pod Creation


```bash
kubectl apply -f nginx-pod.yaml
kubectl get pods
kubectl describe pod nginx-pod
```

> Declarative Pods are **reproducible, version-controllable, and preferred in production**.

---

## Part 3 – Inspecting Pods and Containers

- List containers in a Pod:

```bash
kubectl get pod nginx-pod -o jsonpath='{.spec.containers[*].name}'
```

- Get detailed YAML of a Pod:

```bash
kubectl get pod nginx-pod -o yaml
```

- View logs:

```bash
kubectl logs nginx-pod
```

- For multi-container Pods:

```bash
kubectl logs nginx-pod -c <container-name>
```

---

## Part 4 – Multi-container Pod Example

```bash
kubectl apply -f multi-containerpod.yaml
```

```yaml
spec:
  containers:
    - name: app
      image: nginx
    - name: sidecar
      image: busybox
      command: ["sh", "-c", "while true; do echo hello; sleep 5; done"]
```

- Both containers share Pod IP and lifecycle
- If the Pod dies, all containers die
- Scheduled together on the same node

---

## Part 5 – Operational Tips

- Imperative Pods are good for learning and testing.
- Declarative Pods are preferred for reproducibility.
- Raw Pods are **not self-healing**.
- Most real workloads use **Deployments**, **StatefulSets**, or **Jobs** to create Pods.

---

# Key Takeaways

- Pods are the **smallest deployable unit** in Kubernetes.
- Single-container Pods are most common; multi-container Pods are for tightly-coupled containers.
- Declarative YAML is the recommended way to create Pods in production.
- All Kubernetes controllers (Deployments, StatefulSets, Jobs) manage Pods for self-healing and scaling.
- Always remember: **image → container → Pod → Kubernetes**.