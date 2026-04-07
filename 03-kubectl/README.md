# kubectl & kubeconfig  
## Cluster Access, Contexts, and Managing Multiple Kubernetes Clusters

In this guide we will learn:

- What **kubectl** is
- How to **install kubectl** on Linux, macOS, and Windows
- What **kubeconfig** is
- What **clusters, users, and contexts** mean
- How to **switch between clusters**
- How this works with **local clusters and cloud clusters (like Amazon EKS)**

This topic is extremely important.

You can have multiple Kubernetes clusters running perfectly, but if you don't understand **kubectl and kubeconfig**, you will struggle to operate them.

---

# 1. What is kubectl?

`kubectl` is the **official command-line client for Kubernetes**.

It is the tool used to interact with a Kubernetes cluster.

Examples:

```bash
kubectl get pods
kubectl apply -f deployment.yaml
kubectl delete pod mypod
```

Important concept:

**kubectl does NOT talk directly to nodes.**

The communication flow is:

```
You
→ kubectl
→ Kubernetes API Server
→ Kubernetes Control Plane
```

The **API Server** is the entry point for all Kubernetes operations.

So remember:

> kubectl is just a client that sends requests to the Kubernetes API server.

It does **not run workloads** and **does not store data**.

---

# 2. Installing kubectl

kubectl must be installed on your machine before interacting with a cluster.

Installing kubectl **does not create a cluster**.

It only installs the **client**.

---

## Install kubectl on Windows

1. Download the kubectl binary from the official Kubernetes website.

2. Rename the file to:

```
kubectl.exe
```

3. Place it in a directory that is included in your **PATH**.

Verify installation:

```bash
kubectl version --client
```

---

## Install kubectl on macOS

Using Homebrew:

```bash
brew install kubectl
```

Verify:

```bash
kubectl version --client
```

---

## Install kubectl on Linux

```bash
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"

chmod +x kubectl

sudo mv kubectl /usr/local/bin/
```

Verify installation:

```bash
kubectl version --client
```

If a version is returned, kubectl is installed successfully.

---

# 3. Connecting kubectl to a Cluster

Now the important question:

When you run:

```bash
kubectl get nodes
```

How does kubectl know:

- Which cluster to talk to?
- Where the API server is?
- Which credentials to use?

The answer is:

**kubeconfig**

---

# 4. What is kubeconfig?

`kubeconfig` is a **configuration file used by kubectl**.

It tells kubectl:

- Which clusters exist
- How to authenticate
- Which cluster to connect to

By default the kubeconfig file is located at:

```
~/.kube/config
```

Inside this file are three important sections:

- clusters
- users
- contexts

---

# 5. Understanding Clusters, Users, and Contexts

## Cluster

A **cluster entry** defines a Kubernetes cluster.

It contains:

- The **API server endpoint**
- Certificate information

In simple terms:

> A cluster entry says:  
> "This is the Kubernetes cluster and this is where the API server lives."

---

## User

A **user entry** defines authentication.

It may include:

- Tokens
- Certificates
- Cloud authentication mechanisms

Example authentication methods:

- TLS certificates
- Bearer tokens
- AWS IAM authentication (EKS)

---

## Context

A **context combines**:

- Cluster
- User
- Optional namespace

So a context represents:

```
Context = Cluster + User (+ Namespace)
```

Meaning:

- Which cluster you talk to
- Which identity you use

Switching context changes **both cluster and authentication**.

---

# 6. Viewing Your Kubernetes Contexts

To see all available contexts:

```bash
kubectl config get-contexts
```

Example output:

```
CURRENT   NAME          CLUSTER       AUTHINFO
*         minikube      minikube      minikube-user
          kind-kind     kind-kind     kind-user
```

The `*` indicates the **active context**.

---

To see the **current context**:

```bash
kubectl config current-context
```

This is a **very important command** in real environments.

---

# 7. Switching Between Clusters

To change the active context:

```bash
kubectl config use-context <context-name>
```

Example:

```bash
kubectl config use-context minikube
```

Now kubectl communicates with the **Minikube cluster**.

If using kind, the context may look like:

```
kind-kind
```

Switch to it:

```bash
kubectl config use-context kind-kind
```

Switching contexts is how you change **which cluster kubectl controls**.

---

# 8. Real-World Cluster Environments

In real companies you rarely have just one cluster.

Typical environments include:

- Local cluster
- Development cluster
- Staging cluster
- Production cluster

Example:

```
Local → Minikube
Testing → kind cluster
Staging → Cloud cluster
Production → Cloud cluster
```

Your kubeconfig can store **all of them**.

Example kubeconfig contents:

```
clusters:
- minikube
- kind
- eks-production

contexts:
- minikube
- kind-kind
- eks-prod
```

Switching environments simply means switching **contexts**.

---

# 9. Local Clusters Automatically Update kubeconfig

Tools like:

- **Minikube**
- **kind**

automatically add entries to kubeconfig.

After creating a cluster, you can run:

```bash
kubectl config get-contexts
```

You will see new contexts automatically created.

Examples:

```
minikube
kind-kind
```

This means your kubeconfig has been updated.

---

# 10. Cloud Clusters (Example: Amazon EKS)

Cloud providers also update kubeconfig.

For example with Amazon EKS:

```bash
aws eks update-kubeconfig --region <region> --name <cluster-name>
```

This command adds the EKS cluster to your kubeconfig.

After running it, a new context appears:

```bash
kubectl config get-contexts
```

Example output:

```
minikube
kind-kind
arn:aws:eks:region:cluster/production
```

Now your laptop can control:

- Local clusters
- Testing clusters
- Cloud production clusters

All using **one kubectl installation**.

---

# 11. Verify Your Cluster Access

To verify your connection to a cluster:

```bash
kubectl get nodes
```

Example output:

```
NAME           STATUS   ROLES           AGE
minikube       Ready    control-plane   2h
```

Or with a kind cluster:

```
kind-control-plane   Ready    control-plane
kind-worker          Ready
kind-worker2         Ready
```

If nodes appear, your kubectl connection is working correctly.

---

# 12. Key Takeaways

Important things to remember:

- `kubectl` is the **client tool for Kubernetes**
- It communicates with the **API server**
- `kubeconfig` stores cluster access information
- kubeconfig contains **clusters, users, and contexts**
- A **context = cluster + user**
- Switching context changes which cluster you control
- Your laptop can manage **multiple clusters at once**

---

# Final Note

Before running any command in real environments, always check your active context:

```bash
kubectl config current-context
```

Many production outages happen because engineers believe they are connected to **staging**, while they are actually connected to **production**.

Always verify first.