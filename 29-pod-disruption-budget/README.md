# Kubernetes PodDisruptionBudget (PDB) Deep Dive

In this guide we will explore **PodDisruptionBudget (PDB)** the way real **SREs and platform engineers** understand it.

This will be **very practical**.

We will:

- Create a **3-node Kubernetes cluster**
- Deploy an application with **6 replicas**
- Drain a node **without PDB**
- Observe what happens
- Introduce a **PodDisruptionBudget**
- Repeat the drain
- Force a **drain failure**
- Add **pod anti-affinity**
- Understand how this enables **safe cluster autoscaling**

By the end you will understand:

- `minAvailable`
- `maxUnavailable`
- pod anti-affinity
- scheduler weights
- how autoscalers respect disruption budgets

---

# PART 0 — The Real Problem PDB Solves

In real production systems you run things like:

- APIs
- authentication services
- payment services
- background workers
- message brokers

But Kubernetes nodes **do not live forever**.

Nodes disappear because:

- cluster upgrades
- OS patching
- cluster autoscaler scale-down
- infrastructure failures
- manual maintenance using `kubectl drain`

When a node is drained, Kubernetes **evicts the pods** running on it.

However by default Kubernetes only asks:

> **"Can I move this pod somewhere else?"**

It does **NOT ask**:

> **"Will the application remain healthy if I move too many pods at once?"**

That responsibility belongs to **you**.

This is exactly what **PodDisruptionBudget solves**.

---

# Important Truth About PDB

A **PodDisruptionBudget does NOT protect against failures.**

It does **NOT stop disruptions caused by:**

- node crashes
- kernel panic
- sudden VM termination
- hardware failures

PDB only protects against **voluntary disruptions**.

Examples:

- `kubectl drain`
- cluster autoscaler removing nodes
- rolling node upgrades
- manual maintenance

This is why it is called:

**Pod Disruption Budget**

---

# Real World Example

Imagine you run **6 API pods**.

Your load balancer can safely tolerate losing **2 pods at a time**.

If Kubernetes evicts **4 pods simultaneously**, your users start seeing:

```
500 Internal Server Errors
```

So you define a rule:

> Never allow more than 2 pods to be unavailable.

That rule is exactly what **PDB enforces**.

---

# PART 1 — Create a 3-Node Cluster (Kind)

First we create a cluster with **three worker nodes**.

Create the cluster using the configuration file already included in this repository.

```
kind create cluster --config kind-3nodes.yaml
```

Verify the nodes:

```
kubectl get nodes
```

You should see something similar to:

```
control-plane
worker
worker2
worker3
```

We will only perform experiments on **worker nodes**.

---

# PART 2 — Deploy an Application With 6 Replicas

Next we deploy a simple application with **six replicas**.

Apply the deployment manifest already provided.

```
kubectl apply -f app.yaml
```

Verify the pods:

```
kubectl get pods -o wide
```

Observe how Kubernetes distributes pods across nodes.

Ideally you should see pods spread across the **three workers**.

---

# PART 3 — Experiment 1 (Without PDB)

Now we simulate **node maintenance**.

First list nodes:

```
kubectl get nodes
```

Pick one worker node.

Example:

```
kind-worker
```

---

## Step 1 — Cordon the Node

```
kubectl cordon kind-worker
```

This marks the node as:

```
SchedulingDisabled
```

Meaning:

- New pods will **not be scheduled here**
- Existing pods **remain running**

---

## Step 2 — Drain the Node

Now drain it.

```
kubectl drain kind-worker --ignore-daemonsets
```

Explanation:

`kubectl drain kind-worker`

→ start evicting pods from the node

`--ignore-daemonsets`

→ do not remove daemonset pods such as `kube-proxy`

---

## Watch the Evictions

Run:

```
kubectl get pods -w
```

You will observe multiple pods terminating and being recreated elsewhere.

---

## Important Observation

Because **no PDB exists**, Kubernetes is allowed to evict **as many pods as it wants**.

If all 6 pods were on that node, Kubernetes could remove **all 6 at once**.

Your application has **no protection**.

---

## Step 3 — Make Node Schedulable Again

```
kubectl uncordon kind-worker
```

The node can now accept pods again.

---

# PART 4 — Introducing PodDisruptionBudget

A **PodDisruptionBudget limits how many pods may be unavailable during voluntary disruption**.

There are two ways to express the rule.

You can use:

- `minAvailable`
- `maxUnavailable`

You must choose **one**, not both.

---

# PART 5 — minAvailable vs maxUnavailable

## minAvailable

Defines the **minimum number of pods that must remain available**.

Example:

```
minAvailable: 4
```

If you have 6 pods:

You can only evict **2 pods**.

Because:

```
6 - 2 = 4
```

---

## maxUnavailable

Defines the **maximum number of pods allowed to be unavailable**.

Example:

```
maxUnavailable: 2
```

This means Kubernetes may only disrupt **2 pods simultaneously**.

---

## Subtle Difference

When replica counts change dynamically (for example with **HPA**):

`minAvailable` adapts naturally.

`maxUnavailable` remains a **fixed limit**.

Because of this:

Production APIs usually prefer **minAvailable**.

---

# PART 6 — Create the PodDisruptionBudget

Apply the PDB manifest included in this repository.

```
kubectl apply -f pdb.yaml
```

Check the PDB status:

```
kubectl get pdb
```

You should see something like:

```
NAME       MIN AVAILABLE   MAX UNAVAILABLE   ALLOWED DISRUPTIONS
demo-pdb   4               N/A               2
```

Meaning:

At most **2 pods may be disrupted**.

---

# PART 7 — Drain the Node Again (With PDB)

First ensure the node is schedulable.

```
kubectl uncordon kind-worker
```

Now repeat the process.

### Cordon

```
kubectl cordon kind-worker
```

### Drain

```
kubectl drain kind-worker --ignore-daemonsets
```

Now observe:

```
kubectl get pods -w
```

---

## What Happens Now?

We have:

```
replicas = 6
minAvailable = 4
```

So Kubernetes can only evict:

```
2 pods
```

Once 2 pods are evicted:

Available pods = **4**

The scheduler must **wait until new pods become Ready** before continuing.

Evictions become:

- controlled
- slower
- safe

This is the **real power of PDB**.

---

# PART 8 — Simulate a Drain Failure

Now we intentionally break the system.

Update the PDB so that **all pods must remain available**.

Reapply the updated configuration:

```
kubectl apply -f pdb.yaml
```

Now attempt to drain the node again.

```
kubectl drain kind-worker --ignore-daemonsets
```

You will see something like:

```
eviction would violate the disruption budget
```

---

## Why Did This Fail?

Because the policy says:

```
minAvailable = 6
```

If Kubernetes evicts even **one pod**:

Available pods become **5**.

But the rule requires **6**.

So Kubernetes blocks the eviction.

This is a **very common real-world mistake**.

Teams accidentally configure PDB in a way that **blocks node maintenance**.

---

# PART 9 — Why Pod Anti-Affinity Is Needed

PDB alone is **not enough**.

Imagine this scenario:

All 6 pods are scheduled on **one node**.

If that node is drained:

- all replicas disappear simultaneously
- the service temporarily collapses

Even with a PDB.

To solve this we introduce **pod anti-affinity**.

Anti-affinity tells the scheduler:

> Do not place my replicas on the same node.

---

# PART 10 — Anti-Affinity Rules

Kubernetes supports **two types of anti-affinity rules**.

---

## requiredDuringSchedulingIgnoredDuringExecution

A **hard rule**.

If the scheduler cannot satisfy it:

```
the pod will NOT be scheduled
```

---

## preferredDuringSchedulingIgnoredDuringExecution

A **soft rule**.

The scheduler **tries to satisfy it**, but may break it if necessary.

This is safer for small clusters.

---

# PART 11 — Deploy Anti-Affinity Version

Apply the updated deployment with anti-affinity rules.

```
kubectl apply -f app-with-antiaffinity.yaml
```

Check pod distribution:

```
kubectl get pods -o wide
```

You should now see pods **more evenly distributed across nodes**.

---

# Understanding the Anti-Affinity Rule

The rule targets pods with label:

```
app=demo
```

The scheduler then tries to avoid placing those pods on the same:

```
kubernetes.io/hostname
```

Which effectively means:

```
avoid putting replicas on the same node
```

---

# What Does Weight Mean?

Anti-affinity preferences have weights from:

```
1 – 100
```

Higher weight means:

```
stronger scheduling preference
```

Example:

```
weight: 100
```

This tells the scheduler:

> strongly prefer spreading these pods.

---

# PART 12 — When To Use Required vs Preferred

Use **required** when:

- strict placement rules are necessary
- database leader/follower systems
- control plane style workloads

Use **preferred** when:

- cluster size is small
- node availability is limited
- availability matters more than strict placement

In most real systems:

```
preferred rules are safer
```

---

# PART 13 — How PDB Helps Cluster Autoscaler

Cluster Autoscaler removes nodes when they are underutilized.

Before removing a node it simulates:

```
Can I drain this node safely?
```

During that simulation the autoscaler respects:

```
PodDisruptionBudget
```

If draining the node would violate a PDB:

The autoscaler refuses to remove the node.

---

## Real Example

If you set:

```
minAvailable: 6
```

for a deployment with **6 replicas**,

then the autoscaler can **never remove any node** hosting those pods.

This is how teams accidentally block **cluster scale-down**.

---

# PART 14 — How PDB and Anti-Affinity Work Together

Anti-affinity ensures:

```
pods are spread across nodes
```

PDB ensures:

```
only a safe number of pods may be disrupted
```

Together they guarantee:

- replicas are not concentrated
- node maintenance is safe
- autoscaling works correctly
- rolling upgrades do not break services

---

# Final Mental Model

If you remember nothing else, remember this:

```
Pod Anti-Affinity decides WHERE pods are placed.

PodDisruptionBudget decides HOW MANY pods may be removed during maintenance.
```

Understanding this relationship is **critical for running reliable Kubernetes clusters**.