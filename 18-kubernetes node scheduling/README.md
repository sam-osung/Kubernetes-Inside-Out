# Kubernetes Node Scheduling Explained
## Labels, NodeSelector, NodeAffinity, Taints, and Tolerations

In this guide we will explore one of the most important Kubernetes topics: **Pod Scheduling**.

Scheduling determines **which node a Pod runs on inside the cluster**.

If you do not understand scheduling, you will often face problems like:

- Pods stuck in `Pending`
- Critical workloads running on the wrong nodes
- Resource imbalance across nodes
- Poor cluster utilization
- Lack of workload isolation

This is not just theory.  
This is **critical knowledge for running production clusters**.

---

# What You Will Learn

By the end of this guide you will understand:

- How Kubernetes chooses a node for a Pod
- How Pods request specific nodes
- How nodes block unwanted Pods
- How to isolate workloads using scheduling
- How to debug scheduling problems in production

---

# The Core Scheduling Mental Model

Before diving deeper, remember this rule:

> **Pods control where they want to go.**  
> **Nodes control who is allowed to come.**

This explains the entire scheduling system.

| Component | Controlled By | Purpose |
|------|------|------|
| Labels | Node | Metadata |
| nodeSelector | Pod | Choose node |
| nodeAffinity | Pod | Advanced selection |
| Taints | Node | Blocks Pods |
| Tolerations | Pod | Allows Pods past taints |

---

# How Kubernetes Scheduling Works

When a Pod is created, the Kubernetes scheduler performs the following process:

1. Find all available nodes
2. Filter nodes using scheduling rules
3. Score remaining nodes
4. Choose the best node
5. Bind the Pod to that node

If no node passes the filtering stage, the Pod remains:

```
Pending
```

This is where most scheduling problems happen.

---

# PART 1 — Node Labels

## What is a Label?

A **label** is simple key-value metadata attached to Kubernetes objects.

Labels can exist on:

- Nodes
- Pods
- Deployments
- Services
- Many other Kubernetes resources

For scheduling we focus mainly on **node labels**.

Example label:

```
role=db
```

Meaning:

> This node is intended for database workloads.

---

## Important Rule

Labels **do nothing by themselves**.

They do not:

- attract Pods
- block Pods
- influence scheduling

They only make nodes **selectable later**.

---

## Add a Label to a Node

List nodes:

```bash
kubectl get nodes
```

Add a label:

```bash
kubectl label node worker-1 role=db
```

Verify labels:

```bash
kubectl get nodes --show-labels
```

---

# PART 2 — nodeSelector (Basic Node Scheduling)

`nodeSelector` allows a Pod to run **only on nodes with specific labels**.

It is the **simplest scheduling rule**.

Example idea:

```
Run this Pod only on nodes with role=db
```

---

## Apply the Example

This repository already contains the manifest.

Apply it:

```bash
kubectl apply -f pod-nodeselector.yaml
```

Verify where the Pod landed:

```bash
kubectl get pods -o wide
```

Look at the **NODE column**.

---

## If No Node Matches

If no node has the required label, the Pod will remain:

```
Pending
```

Debug with:

```bash
kubectl describe pod <pod-name>
```

You will see:

```
node(s) didn't match node selector
```

---

# PART 3 — Node Affinity (Advanced Scheduling)

Node Affinity is a more powerful version of `nodeSelector`.

It allows more complex scheduling logic.

Examples:

- Run on DB **OR** INFRA nodes
- Avoid DEV nodes
- Run only on GPU nodes

---

## Apply the Node Affinity Example

This repository includes an affinity example.

Apply it:

```bash
kubectl apply -f pod-node-affinity.yaml
```

Verify scheduling:

```bash
kubectl get pods -o wide
```

---

# PART 4 — Taints (Nodes Blocking Pods)

Until now we saw:

Pods choosing nodes.

Now we flip the perspective.

Nodes can **reject Pods**.

---

## Add a Taint to a Node

Example:

```bash
kubectl taint node worker-2 dedicated=db:NoSchedule
```

Format:

```
key=value:effect
```

Example breakdown:

| Part | Meaning |
|------|------|
| dedicated | key |
| db | value |
| NoSchedule | effect |

---

## Verify the Taint

Run:

```bash
kubectl describe node worker-2
```

You should see:

```
Taints:
dedicated=db:NoSchedule
```

---

## What Happens Now?

If you create a normal Pod (without toleration):

```
kubectl apply -f plain-pod.yaml
```

That Pod **cannot run on worker-2**.

Even if:

- The node has free CPU
- The node has free memory
- The node has the correct labels

The taint blocks the Pod.

---

# PART 5 — Tolerations (Pods Accepting Taints)

To run on a tainted node, a Pod must declare a **toleration**.

A toleration means:

> This Pod is allowed to run on nodes with this taint.

---

## Apply the Toleration Example

```bash
kubectl apply -f pod-toleration.yaml
```

Verify scheduling:

```bash
kubectl get pods -o wide
```

---

## Important Clarification

A toleration **does NOT force the Pod onto that node**.

It only means:

```
The Pod is allowed to run there.
```

The scheduler may still choose another node.

---

# PART 6 — Real Production Pattern (Labels + Taints)

In real clusters we combine **labels and taints**.

Example node configuration:

```
worker-3
label: role=db
taint: dedicated=db:NoSchedule
```

---

## Step 1 — Prepare the Node

```bash
kubectl label node worker-3 role=db
kubectl taint node worker-3 dedicated=db:NoSchedule
```

---

## Step 2 — Deploy the Database Pod

Apply the manifest already included in the repository:

```bash
kubectl apply -f pod-with-select-tolerate.yaml
```

This Pod contains:

- a `nodeSelector` targeting `role=db`
- a `toleration` for `dedicated=db`

---

## Result

Now scheduling behaves correctly:

✔ Pod is **allowed on the node** because it tolerates the taint  
✔ Pod is **targeted to the node** because of the label  

Other workloads cannot run there.

This creates **proper workload isolation**.

---

# Golden Rule

Always remember:

```
Labels do not restrict.
Taints restrict.
```

---

# PART 7 — Types of Taint Effects

Kubernetes supports three taint effects.

### NoSchedule

Pods cannot be scheduled on the node.

Existing Pods remain.

---

### PreferNoSchedule

Soft rule.

Scheduler will try to avoid the node.

---

### NoExecute

Strongest rule.

It can:

- block new Pods
- evict existing Pods

---

# PART 8 — Control Plane Scheduling

Most clusters include this taint:

```
node-role.kubernetes.io/control-plane:NoSchedule
```

This prevents workloads from running on control-plane nodes.

To allow Pods there, you must include a toleration.

---

# PART 9 — Debugging Scheduling Problems

If a Pod is stuck in:

```
Pending
```

Run:

```bash
kubectl describe pod <pod-name>
```

Look under the **Events** section.

Common errors include:

```
node(s) didn't match node selector
```

```
node(s) didn't match node affinity
```

```
node(s) had taint that the pod didn't tolerate
```

These messages explain exactly why scheduling failed.

---

# Final Comparison

| Feature | Controlled By | Purpose |
|------|------|------|
| Label | Node | Metadata |
| nodeSelector | Pod | Basic node targeting |
| nodeAffinity | Pod | Advanced targeting |
| Taint | Node | Blocks Pods |
| Toleration | Pod | Allows Pods past taints |

---

# Final Mental Model

Think of scheduling like **airport security**.

Pods = passengers  
Nodes = countries

Pod rules say:

```
I want to go to this country.
```

Node rules say:

```
Only certain people can enter.
```

So scheduling becomes:

```
nodeSelector / nodeAffinity → where I want to go
taints / tolerations → whether I am allowed there
```

Once you understand this relationship, Kubernetes scheduling becomes **predictable instead of confusing**.