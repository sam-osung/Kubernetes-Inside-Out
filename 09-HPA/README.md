# Kubernetes Horizontal Pod Autoscaler (HPA) – Beginner Friendly Guide

Horizontal Pod Autoscaler (HPA) automatically scales the number of Pods in a Kubernetes workload based on metrics such as CPU or memory usage.

This guide demonstrates how to configure HPA using YAML manifests and how to simulate CPU load to observe scaling behavior.

---

# What is Horizontal Pod Autoscaler?

Horizontal Pod Autoscaler (HPA) automatically increases or decreases the number of Pods in:

- Deployments
- ReplicaSets
- StatefulSets

It uses metrics such as:

- CPU usage
- Memory usage
- Custom metrics

This allows applications to handle fluctuating workloads without manual scaling.

---

# Why HPA is Important

Example scenario:

- Your application is running with **3 Pods**
- Traffic suddenly increases
- CPU usage spikes
- Pods become overloaded

Without autoscaling, users may experience:

- Slow responses
- Application errors

HPA automatically:

- Adds Pods when demand increases
- Removes Pods when demand decreases

---

# How HPA Works

HPA continuously checks resource metrics and compares them with a target threshold.

Example:

| Setting | Value |
|-------|-------|
| Deployment Replicas | 1 |
| CPU Target | 50% |

If:

- Average CPU **> 50%** → HPA **adds Pods**
- Average CPU **< 50%** → HPA **removes Pods**

HPA keeps replicas within:

- **minReplicas**
- **maxReplicas**

---

# Prerequisites

HPA requires **Metrics Server**.

Check if it is installed:

```bash
kubectl get deployment metrics-server -n kube-system
```

If not installed:

```bash
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml
```

Metrics Server collects CPU and memory usage from Pods.

---

# Step 1 – Create the Deployment

Apply the deployment manifest:

```bash
kubectl apply -f nginx-hpa-deployment.yaml
```

Verify Pods:

```bash
kubectl get pods -n dev -o wide
```

Important configuration inside the Deployment:

```yaml
resources:
  requests:
    cpu: 100m
  limits:
    cpu: 500m
```

CPU explanation:

| Value | Meaning |
|------|------|
| 1000m | 1 CPU core |
| 100m | 0.1 CPU |
| 500m | 0.5 CPU |

HPA calculates scaling based on **CPU request**, not limit.

---

# Step 2 – Create the HPA Resource

Apply the HPA manifest:

```bash
kubectl apply -f nginx-hpa.yaml
```

Check the autoscaler:

```bash
kubectl get hpa -n dev
```

---

# HPA Configuration Breakdown

Example configuration:

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
spec:
  minReplicas: 1
  maxReplicas: 5
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 50
```

Explanation:

| Field | Meaning |
|------|------|
| minReplicas | Minimum number of Pods |
| maxReplicas | Maximum number of Pods |
| averageUtilization | Target CPU utilization |

Example:

CPU request = **100m**  
Target = **50%**

Desired CPU usage per Pod = **50m**

If CPU exceeds this → **HPA scales up**.

---

# Step 3 – Simulate CPU Load

To observe autoscaling, generate CPU load inside the Pod.

List Pods:

```bash
kubectl get pods -n dev
```

Exec into the Pod:

```bash
kubectl exec -it <nginx-hpa-pod-name> -n dev -- /bin/sh
```

Inside the container run:

```bash
while true; do :; done
```

Explanation:

- `while true` → infinite loop
- `:` → shell no-operation command that still consumes CPU
- `done` → closes the loop

This loop continuously consumes CPU cycles and triggers the autoscaler.

---

# Step 4 – Observe Autoscaling

Watch Pods:

```bash
kubectl get pods -n dev -o wide --watch
```

Watch the autoscaler:

```bash
kubectl get hpa -n dev --watch
```

Expected scaling:

```
1 → 2 → 3 → 4 → 5 Pods
```

As Pods increase, CPU load is distributed.

---

# Step 5 – Stop the Load

Stop the loop:

```
CTRL + C
```

Exit the container:

```bash
exit
```

Once CPU usage drops, HPA gradually scales the Deployment back down.

---

# Viewing Detailed Metrics

```bash
kubectl describe hpa nginx-hpa -n dev
```

Example output:

```
Metrics: resource cpu on pods (current / target)

120m / 50%
```

Interpretation:

- Current usage: **120m**
- Target: **50% of request**

Since usage is higher, HPA scales up.

---

# Important HPA Behavior

Scaling is **not instant**.

Kubernetes uses stabilization windows to prevent constant scaling up and down (called **thrashing**).

This keeps workloads stable in production environments.

---

# Key Takeaways

- HPA automatically scales Pods horizontally
- Works with CPU, memory, or custom metrics
- Requires **Metrics Server**
- Deployments must define **resource requests**
- YAML configuration is preferred for production

CPU unit recap:

| CPU | Meaning |
|----|----|
| 100m | 0.1 CPU |
| 500m | 0.5 CPU |

HPA targets **percentage of requested CPU**, not limits.

---

# Autoscaling Flow

```
User Traffic
     |
     v
Pods Handle Requests
     |
     v
CPU Usage Increases
     |
     v
Metrics Server Collects Metrics
     |
     v
HPA Compares to Target
     |
     v
Deployment Scales Pods
```

---

# Real Production Use Cases

HPA is commonly used for:

- Web APIs
- Microservices
- Backend processing services
- High traffic web applications
- Event-driven systems

---

# Final Summary

Horizontal Pod Autoscaler enables Kubernetes workloads to automatically scale based on demand.

Benefits:

- Improved performance
- Automatic scaling
- Efficient resource utilization
- Reduced operational overhead

HPA is a **core capability for production Kubernetes environments**.