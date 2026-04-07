# Kubernetes Architecture & Distributions

This guide explains the **internal architecture of a Kubernetes cluster** and the difference between **managed and self-managed Kubernetes distributions**.

Understanding Kubernetes architecture is extremely important because it helps you troubleshoot:

- Cluster failures
- Access problems
- Scheduling issues
- Production incidents

Instead of treating Kubernetes like a black box, you will understand **how the system actually works internally**.

---

# What This Guide Covers

We will explain the following core concepts:

- Control Plane vs Worker Nodes
- The Kubernetes API Server
- The Scheduler
- The Controller Manager
- etcd (the cluster database)
- What runs on worker nodes
- Managed vs Self-Managed Kubernetes clusters

By the end of this guide, you should clearly understand **how the Kubernetes cluster itself is structured**.

---

# Kubernetes Architecture Overview

A Kubernetes cluster is composed of machines that are divided into **two logical groups**.

```
Kubernetes Cluster
│
├── Control Plane
│
└── Worker Nodes
```

These two parts have **very different responsibilities**.

| Component | Responsibility |
|------|----------------|
Control Plane | Manages the cluster |
Worker Nodes | Run application workloads |

Understanding this separation is extremely important for **operations and troubleshooting**.

---

# What Is a Node?

Before going further, we must define the word **Node**.

A node is simply a **machine that participates in the Kubernetes cluster**.

It can be:

- A virtual machine
- A physical server
- A cloud instance

Nodes are categorized as either:

- Control Plane nodes
- Worker nodes

---

# Control Plane

The **Control Plane** is the **brain of the Kubernetes cluster**.

It **does not run your application containers**.

Instead, it manages **how and where applications run**.

Responsibilities of the Control Plane include:

- Deciding what should run in the cluster
- Deciding where workloads should run
- Monitoring the cluster state
- Maintaining the desired system state

A helpful analogy:

If Kubernetes were a company:

| Role | Component |
|-----|-----------|
Management | Control Plane |
Workers | Worker Nodes |

The control plane makes the **decisions**.

Worker nodes perform the **actual work**.

---

# Worker Nodes

Worker nodes are where your **applications actually run**.

This is where Kubernetes runs:

- Pods
- Containers
- Services
- Application workloads

Examples of workloads that run on worker nodes include:

- Web servers
- Backend APIs
- Databases
- Background jobs
- Worker processes

So the separation is simple:

```
Control Plane → decision making
Worker Nodes → workload execution
```

When troubleshooting, one of the first questions engineers ask is:

> Is this a control plane issue or a worker node issue?

---

# Control Plane Components

Inside the control plane, several components work together.

The four most important components are:

- kube-api-server
- etcd
- kube-scheduler
- kube-controller-manager

These components form the **core of Kubernetes**.

---

# kube-api-server

The **API Server** is the **front door of the Kubernetes cluster**.

Every interaction with Kubernetes goes through the API server.

Examples of commands that communicate with the API server include:

```
kubectl apply
kubectl get pods
kubectl delete deployment
```

These commands do **not talk directly to nodes**.

Instead, they communicate with the **API Server**.

The API Server is responsible for:

- Receiving requests
- Validating requests
- Updating cluster state
- Returning cluster information

Everything interacts through the API server, including:

- kubectl
- dashboards
- automation tools
- internal Kubernetes components

Because of this, the API server acts as the **central communication hub of the cluster**.

---

# etcd – The Cluster Database

etcd is the **database used by Kubernetes**.

It is a distributed **key-value store** that holds the entire cluster state.

Examples of information stored in etcd include:

- Pods
- Deployments
- Services
- Namespaces
- Nodes
- Configurations

If someone asks:

> Where does Kubernetes store its state?

The answer is:

```
etcd
```

However, users **never interact with etcd directly**.

All communication goes through the API server.

The flow looks like this:

```
User → API Server → etcd
```

Because etcd contains the cluster's state, its health is extremely important.

If etcd becomes unavailable, the cluster cannot function correctly.

---

# kube-scheduler

The **Scheduler** decides **where Pods should run**.

When you create a Pod or Deployment, Kubernetes creates a Pod object.

Initially, the Pod is **not assigned to any node**.

The scheduler examines:

- Available worker nodes
- Resource availability
- Scheduling constraints
- Policies and rules

After evaluating these conditions, the scheduler selects the **best node** for the Pod.

Important clarification:

The scheduler **does not run containers**.

It only **decides placement**.

---

# kube-controller-manager

The **Controller Manager** runs a set of controllers.

Controllers are loops that constantly compare:

```
Desired State
vs
Current State
```

If the two do not match, controllers attempt to fix the difference.

Example:

If you declare:

```
replicas: 3
```

But only **2 Pods are running**, the controller will create another Pod.

Controllers are responsible for maintaining system consistency.

They handle tasks like:

- Maintaining replica counts
- Replacing failed pods
- Detecting node failures
- Responding to cluster events

Because of controllers, Kubernetes can automatically correct problems.

This is why Kubernetes is known as a **self-healing system**.

---

# Control Plane Summary

The control plane components have clear responsibilities.

| Component | Responsibility |
|------|----------------|
API Server | Entry point to the cluster |
etcd | Cluster database |
Scheduler | Chooses where Pods run |
Controller Manager | Maintains the desired state |

If you understand these four components, you already understand **the core of Kubernetes architecture**.

---

# What Runs on Worker Nodes?

Worker nodes contain components responsible for running workloads.

Two important components run on worker nodes:

- Container Runtime
- kubelet

---

# Container Runtime

The container runtime is responsible for actually running containers.

Examples include:

- containerd
- CRI-O

The runtime pulls images and executes containers.

---

# kubelet

The **kubelet** is the node agent that runs on every worker node.

Its responsibilities include:

- Communicating with the API server
- Receiving instructions about Pods
- Ensuring containers are running
- Reporting node status back to the control plane

The control plane **does not start containers directly**.

Instead, the control plane communicates with kubelets.

The flow looks like this:

```
Control Plane → kubelet → container runtime → container starts
```

When Pods fail to start, engineers often inspect **kubelet logs** for troubleshooting.

---

# Managed vs Self-Managed Kubernetes

Kubernetes clusters can be deployed in two main ways.

- Self-Managed Kubernetes
- Managed Kubernetes

These approaches differ in **who is responsible for operating the cluster infrastructure**.

---

# Self-Managed Kubernetes

In self-managed Kubernetes, your team operates the entire cluster.

Responsibilities include:

- Installing Kubernetes
- Managing control plane nodes
- Upgrading Kubernetes versions
- Managing certificates
- Backing up etcd
- Maintaining high availability
- Managing worker node lifecycle

Tools often used include:

```
kubeadm
```

Self-managed clusters offer **maximum control**, but they also require significant operational effort.

If something fails in the control plane, your team must fix it.

---

# Managed Kubernetes

Managed Kubernetes shifts control plane management to a cloud provider.

The provider operates the cluster infrastructure.

They handle tasks such as:

- Running API servers
- Maintaining etcd
- Performing upgrades
- Ensuring high availability
- Applying security patches

Your team focuses on:

- Worker nodes
- Applications
- Workloads

A popular example is:

```
Amazon EKS
```

When using EKS:

- You do not access control plane servers
- You do not manage etcd
- You interact with the cluster through `kubectl`

Managed Kubernetes significantly reduces operational complexity.

---

# Quick Comparison

| Feature | Self-Managed | Managed |
|------|---------------|---------|
Control Plane | You manage it | Cloud provider manages it |
Cluster Upgrades | Manual | Provider handled |
Operational Complexity | High | Lower |
Control | Full control | Less control |

---

# Mental Model for Kubernetes

When you deploy an application, visualize the following sequence.

Your command goes to:

```
kubectl
   ↓
API Server
   ↓
etcd stores the state
   ↓
Scheduler chooses a node
   ↓
kubelet on the node starts containers
   ↓
Controllers keep monitoring and fixing
```

If you keep this mental model in mind, Kubernetes becomes much easier to understand.

---

# Key Takeaways

Important ideas to remember:

- A Kubernetes cluster has **two main parts**: Control Plane and Worker Nodes
- The control plane manages the cluster
- Worker nodes run application workloads
- The API server is the central communication hub
- etcd stores the cluster state
- The scheduler decides where Pods run
- Controllers maintain the desired system state
- kubelet executes workloads on worker nodes
- Kubernetes clusters can be **self-managed or managed**

Understanding this architecture will make future Kubernetes concepts much easier to learn and troubleshoot.