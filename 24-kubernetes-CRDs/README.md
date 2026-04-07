In this guide, we explore **how Kubernetes evolves from its core primitives into full production platforms**. We cover everything from vanilla Kubernetes to CRDs, controllers, and operators, culminating in real-world platforms.

---


Real Kubernetes clusters often include:

- Certificates
- Databases
- Kafka topics
- Applications
- Tenants

These do not exist in Kubernetes by default. How does Kubernetes expose APIs for them?

We will cover:

1. What vanilla Kubernetes contains
2. Its limitations
3. CRDs (Custom Resource Definitions)
4. Why CRDs alone do nothing
5. Controllers implementing behavior
6. Operators and programming them
7. How production platforms are built

**Key takeaway:** Kubernetes is an extensible platform, not just a container orchestrator.

---

## — Vanilla Kubernetes

**Vanilla Kubernetes:** a cluster from the official project, **without extensions**.

**Characteristics:**

- No operators
- No CRDs
- No platform add-ons
- Provides only native APIs → kernel of the cloud-native OS

### Core Resources (API v1)

- Pod, Service, ConfigMap, Secret
- Namespace, Node
- PersistentVolume, PersistentVolumeClaim
- ServiceAccount, Event

**Example:**

```yaml
apiVersion: v1
kind: Pod
```

### Workload Controllers (apps/v1)

- Deployment
- ReplicaSet
- StatefulSet
- DaemonSet

**Behavior:** Ensure N replicas are running.

### Batch Workloads (batch/v1)

- Job
- CronJob

### Networking (networking.k8s.io/v1)

- Ingress, NetworkPolicy, IngressClass

### RBAC (rbac.authorization.k8s.io/v1)

- Role, RoleBinding, ClusterRole, ClusterRoleBinding

### Storage (storage.k8s.io/v1)

- StorageClass, CSIDriver, VolumeAttachment

### Autoscaling (autoscaling/v2)

- HorizontalPodAutoscaler

### Stability & Scheduling

- PriorityClass
- PodDisruptionBudget

**Key point:** All of these exist in **every cluster** by default — no CRDs needed.

---

## How to Verify Native Resources

Run:

```bash
kubectl api-resources
```

You will see groups like:

- v1
- apps
- batch
- networking.k8s.io
- rbac.authorization.k8s.io
- storage.k8s.io

External CRDs appear with other domains, e.g., `cert-manager.io`, `istio.io`.

---

## Limitation of Vanilla Kubernetes

Vanilla Kubernetes understands **infrastructure primitives**, but cannot natively manage higher-level concepts:

- Certificates
- Databases
- Kafka topics
- Applications
- Tenants

**Solution:** **Custom Resource Definitions (CRDs)**.

---

## What is a CRD?

**Custom Resource Definition (CRD):**

- Extends Kubernetes API with **new resource types**
- Example: Certificate, Database, App, Tenant
- CRDs **define structure** only, not behavior

**Analogy:** CRD = database schema

---

## Create Your Own CRD

**Apply the CRD manifest:**

```bash
kubectl apply -f app-crd.yaml
kubectl get crd
kubectl api-resources | grep app
```

**Result:** Kubernetes now knows about a new resource type: `App`.

---

### Create a Custom Resource


```bash
kubectl apply -f my-app.yaml
kubectl get apps
kubectl get app demo-app -o yaml
```

**Observation:** At this point:

- No Pod is created
- No Deployment is created
- CRD + Custom Resource = API only

---

## Controllers and Operators

**Controllers:**

- Watch the Kubernetes API
- Filter events for a resource type
- Reconcile desired state

**Operators:**

- Controllers built **specifically for CRDs**
- Implement reconciliation logic
- Manage creation/update/deletion of native resources
- Update `.status` field

**Who writes them?** Platform engineers or developers.  
**Where do they run?** As a Deployment or StatefulSet inside the Kubernetes cluster.

**Architecture:**

```
CRD → Operator (controller code) → Native resources → Status
```

**operator deployment example:**

```bash
apiVersion: apps/v1
kind: Deployment
metadata:
  name: demo-operator
  namespace: platform
spec:
  replicas: 1
  selector:
    matchLabels:
      app: demo-operator
  template:
    metadata:
      labels:
        app: demo-operator
    spec:
      containers:
        - name: operator
          image: myregistry/demo-operator:latest  # THIS IS THE OPERATOR IMAGE
```

---


## How Operators Work

**Step-by-step flow:**

1. User creates a Custom Resource  
2. API server stores it in etcd  
3. Operator detects the new resource  
4. Operator reads `.spec` fields (e.g., `image`, `replicas`)  
5. Operator creates/updates native resources (Deployments, Services, Secrets)  
6. Operator updates `.status` fields on the Custom Resource  
7. Loop repeats continuously → **reconciliation**  

**Important points:**

- Operators watch CRDs, not raw Pods  
- `.spec` defines **desired state**  
- `.status` reports **actual state**  
- Operators allow **declarative automation** for custom resources  

---

## Using cert-manager as real example

- CRDs: Certificate, Issuer, ClusterIssuer
- Operator interacts with external CAs
- Automates issuance and renewal
- `.spec` = desired state
- `.status` = reality

---

## Summary

- **Vanilla Kubernetes:** basic building blocks
- **CRDs:** define new API types
- **Custom Resources:** user requests
- **Operators:** implement behavior for CRDs
- **Production pattern:** CRD + Operator + Native resources

**Mental model:** 

```
CRDs define WHAT you want.
Operators/controllers define HOW it happens.
```

This is how Kubernetes becomes a **programmable internal cloud API**.
