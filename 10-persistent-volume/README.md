# Kubernetes Persistent Storage – Beginner-Friendly Guide

Kubernetes separates **Pods** from **storage** to ensure data persists beyond Pod lifecycles.

---

# What Persistent Storage Means

Data persists even if a Pod is:

- Deleted
- Restarted
- Rescheduled to another node

By default, Pods are **ephemeral**. If a Pod dies, its container filesystem is destroyed and all data inside disappears.

**Solution:** Separate application from storage using:

- **PersistentVolume (PV)**
- **PersistentVolumeClaim (PVC)**

---

# Ephemeral Pod Example

Apply a demo Pod:

```bash
kubectl apply -f pod.yaml
```

Enter the container:

```bash
kubectl exec -it demo-pod -- /bin/sh
```

Create a file:

```bash
echo "hello" > /usr/share/nginx/html/test.txt
```

Delete and recreate the Pod:

```bash
kubectl delete pod demo-pod
kubectl apply -f pod.yaml
```

Result: File is gone because the container filesystem was ephemeral.

---

# Kubernetes Storage Building Blocks

## PersistentVolume (PV)

Represents real storage in the cluster. Examples:

- Disk
- Directory on a node
- NFS share
- Cloud volume

PV is cluster-scoped and created by administrators or the platform.

## PersistentVolumeClaim (PVC)

A PVC is a request for storage.

Example:

> "I want 1Gi of storage with ReadWriteOnce access."

Pods **never use PVs directly**; they always use PVCs.

## Storage Chain

```
Pod → PVC → PV → Real Storage
```

---

# Creating a PersistentVolume

Apply PV manifest:

```bash
kubectl apply -f pv.yaml
kubectl get pv
```

**Note:** `hostPath` is used for learning, not production.

### Important PV Fields

- **Capacity:** e.g., 1Gi
- **AccessModes:** e.g., `ReadWriteOnce` (one node can mount)
- **ReclaimPolicy:** e.g., `Retain` (data persists after PVC deletion)

---

# Creating a PersistentVolumeClaim

Apply PVC manifest:

```bash
kubectl apply -f pvc.yaml
kubectl get pvc
```

If PV matches the request, PVC status becomes:

```
Bound
```

Inspect binding:

```bash
kubectl describe pvc demo-pvc
```

---

# Mounting PVC into a Pod

Apply Pod manifest that mounts storage:

```bash
kubectl apply -f pod-with-storage.yaml
```

Pod spec includes:

- **Volumes:** Define storage source (PVC)
- **VolumeMounts:** Define container mount path

---

# Verifying Persistence

Enter Pod:

```bash
kubectl exec -it demo-pod -- /bin/sh
```

Create file:

```bash
echo "persistent data" > /usr/share/nginx/html/index.html
```

Delete and recreate Pod:

```bash
kubectl delete pod demo-pod
kubectl apply -f pod-with-storage.yaml
```

Result: File persists because it is stored on:

```
PV → hostPath → /mnt/demo-data
```

---

# Mental Model

```
Pod (temporary)
   ↓
PVC (storage request)
   ↓
PV (actual storage)
   ↓
Disk / Folder / Cloud Volume
```

---

# Real Use Case

Databases require persistent storage:

- MySQL
- PostgreSQL
- Redis

Architecture:

```
Database Pod → PVC → PV
```

---

# StatefulSets

Used for workloads needing stable identity and storage:

- Databases
- Message brokers
- Distributed systems

Features:

- Stable Pod names (db-0, db-1)
- Stable network identity
- Stable storage

---

# Why One PVC per StatefulSet is Wrong

Problems if multiple Pods share a PVC:

1. **Storage Isolation:** Each replica must have its own disk
2. **Access Mode Limits:** `ReadWriteOnce` blocks multiple mounts
3. **Identity Mismatch:** StatefulSet guarantees unique data per Pod

---

# Correct Approach: volumeClaimTemplates

StatefulSets use **volumeClaimTemplates** to create a PVC per Pod.

Deploy StatefulSet:

```bash
kubectl apply -f statefulset.yaml
kubectl get pvc
```

Example PVCs:

```
data-web-0 data-web-1
```

Mapping:

```
web-0 → data-web-0
web-1 → data-web-1
```

---

# Why volumeClaimTemplates Works

| Feature                 | Benefit                          |
|-------------------------|----------------------------------|
| One volume per Pod       | Safe storage isolation           |
| Stable storage           | Survives Pod restarts            |
| Automatic PVC creation   | No manual management             |
| Correct mapping          | Pod index maps to PVC index      |

---

# StorageClass Behavior

If no StorageClass is specified, Kubernetes uses the **default StorageClass**:

```bash
kubectl get storageclass
```

---

# Dynamic Provisioning Flow

```
StatefulSet → volumeClaimTemplates → PVC → StorageClass → Dynamic PV → Real Storage
```

---

# Deleting StatefulSets

```bash
kubectl delete statefulset web
```

- Pods are deleted
- PVCs remain
- Recreated Pods reconnect to the same PVCs

---

# Production Architecture Comparison

**Minikube / Learning:**

```
PV → hostPath → Local Node Directory
```

**EKS / Production:**

```
StatefulSet → volumeClaimTemplates → PVC per Pod → StorageClass (EBS CSI) → Dynamic PV → Amazon EBS
```

Each replica receives:

- Its own disk
- Its own identity
- Its own persistent data

---

# Real Production Use Cases

Used for:

- MySQL / PostgreSQL
- Redis
- Kafka
- Elasticsearch

Pattern:

```
StatefulSet + volumeClaimTemplates + Cloud StorageClass (gp3 / EBS)
```

This is the standard design for **stateful workloads in Kubernetes**.