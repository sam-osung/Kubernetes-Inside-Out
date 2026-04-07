# Kubernetes DaemonSet
## Running One Pod on Every Node

In this guide we will learn:

- What a **DaemonSet** is
- When to use a **DaemonSet**
- How it differs from a **Deployment**
- How to deploy a DaemonSet from this repository
- How to verify that it runs on every node

This repository already contains the DaemonSet manifest.

After cloning the repository, you only need to **apply the file**.

---

# 1. What is a DaemonSet?

A **DaemonSet** is a Kubernetes controller that ensures:

> Exactly **one Pod runs on every node** in the cluster (or on selected nodes).

Example:

If your cluster has:

```
3 nodes → 3 pods
5 nodes → 5 pods
10 nodes → 10 pods
```

You **do not set replicas manually**.

Kubernetes automatically schedules **one Pod per node**.

---

# 2. Simple Mental Model

Think of a DaemonSet as saying:

> “I want this Pod to exist on every node.”

Whenever a **new node joins the cluster**, Kubernetes will automatically create a new DaemonSet Pod on that node.

No scaling configuration is needed.

---

# 3. Real DevOps Use Cases

DaemonSets are extremely common in production Kubernetes clusters.

Typical examples include:

### Log Collection

Tools that collect logs from every node.

Examples:

- Fluent Bit
- Filebeat

They must run **on every node** to capture container logs.

---

### Monitoring Agents

Monitoring tools often run as DaemonSets.

Examples:

- Prometheus Node Exporter
- Datadog agent
- New Relic agent

These collect **node-level metrics**.

---

### Security Agents

Security tools that monitor node activity.

Examples:

- Falco
- Sysdig agents

These monitor **system calls and container activity**.

---

### Networking Components

Some networking components run on every node.

Examples:

- kube-proxy
- CNI networking plugins

These must run on each node to manage networking.

---

# 4. Deployment vs DaemonSet

| Feature | Deployment | DaemonSet |
|------|------|------|
| Replicas | You choose | Kubernetes decides |
| Pod placement | Anywhere in cluster | One per node |
| Typical usage | Applications, APIs | Node-level agents |

Example:

A **web API** uses a Deployment.

A **log collector** uses a DaemonSet.

---

# 5. Deploying the DaemonSet

This repository already includes the DaemonSet manifest.

After cloning the repository, apply it using:

```bash
kubectl apply -f daemonset.yaml
```

Kubernetes will automatically create **one Pod on each node**.

---

# 6. Verify the DaemonSet

Always verify that Kubernetes created the resources correctly.

---

## Check the DaemonSet

```bash
kubectl get daemonset
```

Example output:

```
NAME           DESIRED   CURRENT   READY
node-checker   3         3         3
```

Explanation:

| Column | Meaning |
|------|------|
| DESIRED | Number of nodes in the cluster |
| CURRENT | Number of pods created |
| READY | Pods currently running |

---

## Check Pod Placement

Run:

```bash
kubectl get pods -o wide
```

Look at the **NODE column**.

Example:

```
node-checker-abcde   Running   node-1
node-checker-fghij   Running   node-2
node-checker-klmno   Running   node-3
```

This confirms:

> One DaemonSet Pod is running on each node.

---

## Describe the DaemonSet

For more detailed information:

```bash
kubectl describe daemonset node-checker
```

This command shows:

- Desired number of nodes
- Current scheduled pods
- Events
- Scheduling errors (if any)

---

# 7. Automatic Scaling With Nodes

DaemonSets automatically react to cluster changes.

If a **new node joins the cluster**, Kubernetes will automatically create a new Pod for the DaemonSet on that node.

You do not need to modify replicas or configuration.

---

# 8. Control Plane Node Behavior

By default, DaemonSet Pods **do not run on control-plane nodes**.

Control-plane nodes usually have a taint like this:

```
node-role.kubernetes.io/control-plane
```

This prevents regular workloads from being scheduled there.

As a result, DaemonSets normally run **only on worker nodes**.

---

# 9. How Kubernetes Decides Where DaemonSets Run

Internally, Kubernetes follows these steps:

1. It looks at all nodes in the cluster.

2. It filters nodes using:

- Taints and tolerations
- Node selectors
- Node affinity rules

3. It schedules **exactly one Pod per allowed node**.

---

# 10. Key Takeaways

Important concepts to remember:

- A **DaemonSet runs one Pod per node**
- Kubernetes automatically handles scaling
- Used for **node-level services**
- Common for **logging, monitoring, networking, and security**
- Pods usually run only on **worker nodes**

---

# Final Thought

Whenever you hear:

> "This service must run on every node"

The correct Kubernetes solution is usually a:

**DaemonSet.**