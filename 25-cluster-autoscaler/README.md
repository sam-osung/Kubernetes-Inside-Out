# Kubernetes Cluster Autoscaler on EKS -- Complete Beginner-Friendly Guide

Hello everyone! Today we are doing a deep dive into **Kubernetes Cluster
Autoscaler (CA)** on Amazon EKS. By the end of this guide, you'll
understand:

-   Why node groups are required for autoscaling
-   The difference between HPA and Cluster Autoscaler
-   How to create an EKS cluster with OIDC enabled
-   How to set up IAM roles for service accounts (IRSA)
-   How Cluster Autoscaler discovers node groups via tags
-   How to install and configure Cluster Autoscaler with Helm
-   How to test autoscaling with real pods

------------------------------------------------------------------------

## 1️⃣ Theory: What is Cluster Autoscaler?

Cluster Autoscaler (CA) is a Kubernetes component that automatically
adjusts the **number of nodes** in your cluster.

It performs two main functions:

### Scaling Up

-   Adds nodes when pods are pending due to insufficient resources.\
-   Example: Deploy a pod needing 4 CPUs, but your cluster only has 2
    free → CA launches a new node to schedule it.

### Scaling Down

-   Removes nodes when they are underutilized.\
-   Example: Node mostly idle, pods can reschedule → CA drains &
    terminates it to save cost.

> Important: CA manages **nodes**, not pods. Pods are scheduled by the
> Kubernetes scheduler.

------------------------------------------------------------------------

## 2️⃣ HPA vs Cluster Autoscaler

  Feature   Horizontal Pod Autoscaler     Cluster Autoscaler
  --------- ----------------------------- ------------------------
  Scales    Pods (replicas)               Nodes
  Trigger   CPU, memory, custom metrics   Pending pods
  Scope     Per workload                  Entire cluster
  Purpose   Application scaling           Infrastructure scaling

### Example Scenario

-   HPA scales app from 4 → 10 replicas if CPU spikes.\
-   Cluster has only enough capacity for 6 replicas → 4 pods Pending.\
-   Cluster Autoscaler detects this → adds nodes → all pods scheduled.

**Key takeaway:**\
HPA = scales workloads; CA = scales infrastructure.

------------------------------------------------------------------------

## 3️⃣ Step 1 -- Create the EKS Cluster (with OIDC and Node Groups)

We will use **eksctl** with OIDC enabled and node group tags for
auto-discovery.

Create `cluster-config.yaml`:

``` yaml
apiVersion: eksctl.io/v1alpha5
kind: ClusterConfig

metadata:
  name: my-eks-cluster
  region: us-east-1

iam:
  withOIDC: true

managedNodeGroups:
  - name: workers
    instanceType: c7i-flex.large
    desiredCapacity: 4
    minSize: 2
    maxSize: 10
    tags:
      k8s.io/cluster-autoscaler/enabled: "true"
      k8s.io/cluster-autoscaler/my-eks-cluster: "owned"
```

Create the cluster:

``` bash
eksctl create cluster -f cluster-config.yaml
```

After creation you will have:

-   EKS cluster `my-eks-cluster`
-   OIDC enabled
-   Node group `workers`
-   Auto-discovery tags configured

------------------------------------------------------------------------

## 4️⃣ Step 2 -- Using IRSA (Pod Identity)

Before creating the IAM service account, **the IAM policy must exist in
your AWS account**.

Cluster Autoscaler requires permissions to interact with Auto Scaling
Groups.

Create the policy file:

`cluster-autoscaler-policy.json`

``` json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "autoscaling:DescribeAutoScalingGroups",
        "autoscaling:DescribeAutoScalingInstances",
        "autoscaling:DescribeLaunchConfigurations",
        "autoscaling:DescribeTags",
        "autoscaling:SetDesiredCapacity",
        "autoscaling:TerminateInstanceInAutoScalingGroup",
        "ec2:DescribeLaunchTemplateVersions"
      ],
      "Resource": "*"
    }
  ]
}
```

Create the policy:

``` bash
aws iam create-policy \
  --policy-name ClusterAutoscalerPolicy \
  --policy-document file://cluster-autoscaler-policy.json
```

Now create the IAM ServiceAccount using IRSA:

``` bash
eksctl create iamserviceaccount \
  --name cluster-autoscaler \
  --namespace kube-system \
  --cluster my-eks-cluster \
  --attach-policy-arn arn:aws:iam::<ACCOUNT_ID>:policy/ClusterAutoscalerPolicy \
  --approve \
  --override-existing-serviceaccounts
```

Now the Cluster Autoscaler pod can assume this IAM role securely.

------------------------------------------------------------------------

## 5️⃣ Step 3 -- Install Cluster Autoscaler Using Helm

Add the autoscaler Helm repository:

``` bash
helm repo add autoscaler https://kubernetes.github.io/autoscaler
```

Update Helm repositories:

``` bash
helm repo update
```

Install Cluster Autoscaler:

``` bash
helm install cluster-autoscaler autoscaler/cluster-autoscaler \
  --namespace kube-system \
  --set autoDiscovery.clusterName=my-eks-cluster \
  --set awsRegion=us-east-1 \
  --set rbac.serviceAccount.create=false \
  --set rbac.serviceAccount.name=cluster-autoscaler \
  --set extraArgs.skip-nodes-with-local-storage=false \
  --set extraArgs.skip-nodes-with-system-pods=false \
  --set extraArgs.expander=random \
  --set replicaCount=1
```

------------------------------------------------------------------------

## 6️⃣ Step 4 -- Verify Installation

``` bash
kubectl get pods -n kube-system | grep cluster-autoscaler

kubectl logs -f deployment/cluster-autoscaler-aws-cluster-autoscaler -n kube-system
```

You should see logs showing node group discovery and scaling activity.

------------------------------------------------------------------------

## 7️⃣ Step 5 -- Test Auto‑Scaling

Deploy a workload that requires high CPU:

``` yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: cpu-burn-deployment
spec:
  replicas: 6  # enough to trigger scale-up
  selector:
    matchLabels:
      app: cpu-burn
  template:
    metadata:
      labels:
        app: cpu-burn
    spec:
      containers:
        - name: burn
          image: vish/stress
          args: ["-cpus", "1"]  # stress 1 CPU per pod
          resources:
            requests:
              cpu: "1000m"  # 1 CPU, fits on c7i-flex.large
            limits:
              cpu: "1000m"
```

``` bash
kubectl apply -f test.yaml
```

This demo shows how Cluster Autoscaler automatically adds nodes to your EKS cluster when pods cannot be scheduled due to resource constraints.

---

## What happened

1. You created a deployment with **6 pods** requesting **1 CPU each**.
2. Your cluster originally had **2 nodes**, each **2 vCPU**. Only 4 pods could fit → **2 pods were Pending**.
3. **Cluster Autoscaler** detected unschedulable pods and automatically **provisioned new nodes**.
4. New nodes became **Ready**, and the pending pods were scheduled.

---

## How to confirm

Run the following commands:

```bash
kubectl get pods -o wide
kubectl get nodes
```

Later, unused nodes may scale down automatically.

------------------------------------------------------------------------

## 8️⃣ Best Practices

-   Always use **IRSA**
-   Tag node groups for **auto-discovery**
-   Match **Cluster Autoscaler version** with Kubernetes version
-   Configure safe **min/max node limits**
-   Monitor **Cluster Autoscaler logs**

------------------------------------------------------------------------

## Final Mental Model

    EKS Cluster with OIDC
            ↓
    Node Groups with Autoscaler Tags
            ↓
    IAM Service Account (IRSA)
            ↓
    Helm Installed Cluster Autoscaler
            ↓
    Automatic Node Scaling

Cluster Autoscaler adjusts the **infrastructure**, while Kubernetes
schedules the **pods**.