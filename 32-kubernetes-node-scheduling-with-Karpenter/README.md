# Karpenter on EKS — Full Step-by-Step Beginner-to-Production Guide

---

## 1️⃣ What is Karpenter?

Karpenter is a **dynamic Kubernetes node provisioning system**:

* It **automatically launches EC2 instances** when pods cannot be scheduled due to insufficient resources.
* Runs as a **controller pod in Kubernetes** and communicates with AWS.
* Unlike Cluster Autoscaler, it **does not rely on Auto Scaling Groups (ASGs)**, and can select the **exact instance type and size** based on pod requests.
* Considers **CPU, memory, GPU, architecture, cost, and availability zones** to provision nodes optimally.

**Example:**

* Pod requests **2 CPUs and 4GB memory**.
* Cluster has a **t3.small node (1 CPU, 2GB)** → pod cannot schedule.
* Karpenter selects **t3.medium (2 CPU, 4GB)** and launches it immediately.

> Key idea: Nodes are provisioned **“just-in-time”** instead of scaling entire node groups, reducing idle resources and cost.

---

## 2️⃣ Why Karpenter Was Created

Before Karpenter:

* Cluster Autoscaler scales **entire node groups** → often over-provisioning resources.
* Scaling is **slower** (2–4 minutes per node).
* Managing multiple node groups for CPU, GPU, memory, and spot workloads is **complex**.

Karpenter solves these problems by:

* Launching **individual nodes dynamically**.
* Selecting **optimal instance types automatically**.
* Supporting **multi-NodePool setups** for workload isolation.
* Scaling **faster** and **more cost-efficiently**.

**Example Scenario:**

* ML workload requires **GPU**. Cluster has CPU nodes only → Cluster Autoscaler cannot help.
* Karpenter sees **GPU requirement** → launches GPU-enabled instance automatically.

---

## 3️⃣ Cluster Autoscaler vs Karpenter

| Feature        | Cluster Autoscaler                  | Karpenter                                                              |
| -------------- | ----------------------------------- | ---------------------------------------------------------------------- |
| Scales         | Node groups                         | Individual nodes                                                       |
| Instance types | Predefined in ASG                   | Dynamically selected based on pod requirements                         |
| Speed          | Moderate (2–4 mins to launch nodes) | Fast (launches immediately when pod pending)                           |
| Efficiency     | Medium                              | High (optimal bin packing, less wasted resources)                      |
| Flexibility    | Limited                             | Very high (multi-instance types, spot/on-demand, GPU/CPU/memory-heavy) |

> Insight: Karpenter is ideal for **heterogeneous workloads** where each pod may have different resource requirements.

---

## 4️⃣ What Makes Karpenter Stand Out

* **Dynamic instance selection:** Launches the **smallest instance that fits pending pods** → reduces cost.
* **Fast scaling:** Nodes are provisioned immediately → reduces pod pending time.
* **Better bin packing:** Pods are efficiently packed → reduces idle resources.
* **Workload isolation:** Labels and taints allow separating workloads into different NodePools.
* **Supports mixed instance types:** Spot, on-demand, GPU, CPU, memory-heavy.

**Example:**

* Batch job → **high CPU NodePool**
* ML model training → **GPU NodePool**
* Background jobs → **spot NodePool**

---

## 5️⃣ Requirements Before Installing Karpenter

| Requirement         | Why                                                                                                |
| ------------------- | -------------------------------------------------------------------------------------------------- |
| **EKS cluster**     | Kubernetes control plane for pods                                                                  |
| **OIDC enabled**    | Required for IAM Roles for Service Accounts (IRSA) so Karpenter pods can assume AWS roles securely |
| **IAM permissions** | Karpenter requires permission to launch EC2 instances, attach SGs, and assign IAM roles            |
| **Subnets**         | Nodes need networking; Karpenter must know which subnets are allowed                               |
| **Security groups** | Nodes need proper SGs attached; tagging ensures Karpenter can discover them                        |
| **Helm**            | Simplifies installation of Karpenter controller and associated CRDs                                |

---

## 6️⃣ Bootstrap Node Requirement

Even though Karpenter provisions nodes dynamically, you need **at least one node to run the Karpenter controller**:

```text
Bootstrap NodeGroup (1 small node)
↓
Install Karpenter controller
↓
Karpenter provisions future nodes dynamically
```

**Example:** t3.small (1 CPU, 2GB RAM)
Once Karpenter runs, it automatically provisions nodes for workloads.

---

## 7️⃣ Creating EKS Cluster with eksctl and Discovery Tags

**File: `cluster-config.yaml`**


> **Explanation:**
>
> * `metadata.name / metadata.region` → Cluster name and region
> * `iam.withOIDC` → Enables IRSA for Karpenter controller
> * `managedNodeGroups` → Defines a bootstrap node to run Karpenter controller
> * `vpc.subnets.private.tags` → Marks subnets where Karpenter can provision nodes


```bash
eksctl create cluster -f cluster-config.yaml
```

---

## 8️⃣ Manually Tag Security Groups

Karpenter needs tagged SGs for nodes:

```bash
aws ec2 create-tags \
  --resources sg-0abc12345example \
  --tags Key=karpenter.sh/discovery,Value=karpenter-demo
```

> **Why:** Karpenter attaches newly launched EC2 nodes to SGs to join cluster networking.

---

## 9️⃣ Karpenter IAM Setup (Service Account + Node Role + Instance Profile)

### 9️⃣a Create IAM Policy for Karpenter

**File: `KarpenterControllerPolicy.json`**

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": [
                "ec2:CreateLaunchTemplate",
                "ec2:CreateFleet",
                "ec2:RunInstances",
                "ec2:DescribeInstances",
                "ec2:DescribeLaunchTemplates",
                "ec2:DescribeInstanceTypes",
                "ec2:TerminateInstances",
                "ec2:DescribeSubnets",
                "ec2:DescribeSecurityGroups",
                "iam:PassRole"
            ],
            "Resource": "*"
        }
    ]
}
```


```bash
aws iam create-policy \
  --policy-name KarpenterControllerPolicy \
  --policy-document file://KarpenterControllerPolicy.json
```

> **Why:** Allows Karpenter to launch EC2 instances, manage SGs/subnets, and use node instance role.

### 9️⃣b Create Karpenter IAM Service Account

```bash
eksctl create iamserviceaccount \
  --cluster my-eks-cluster \
  --namespace karpenter \
  --name karpenter \
  --attach-policy-arn arn:aws:iam::<ACCOUNT_ID>:policy/KarpenterControllerPolicy \
  --approve \
  --override-existing-serviceaccounts
```

> **Why:** Links the Karpenter controller pod to an IAM role so it can call AWS APIs.

### 9️⃣c Create Node Instance Role & Instance Profile (For EC2 Nodes)

**Step 1 — Name the role:**

* Role name: `KarpenterNodeRole`
* Instance profile name: `KarpenterNodeInstanceProfile` (must match Helm flag)

**Step 2 — Create IAM Role in AWS Console / CLI:**

* Trusted entity: **EC2**
* Attach policies:

  * `AmazonEKSWorkerNodePolicy` → Allows node to join cluster
  * `AmazonEC2ContainerRegistryReadOnly` → Pull images from ECR
  * `AmazonEKS_CNI_Policy` → Networking support

**Step 3 — Create Instance Profile:**

* Name: `KarpenterNodeInstanceProfile`
* Attach `KarpenterNodeRole` to it.

> **Why:** Nodes need this to register with EKS, pull images, and use networking.

---

## 🔟 Install Karpenter via Helm

```bash
helm repo add karpenter https://charts.karpenter.sh
helm repo update

kubectl create namespace karpenter

helm install karpenter karpenter/karpenter \
  --namespace karpenter \
  --version 0.30.0 \
  --set serviceAccount.create=false \
  --set serviceAccount.name=karpenter \
  --set clusterName=my-eks-cluster \
  --set clusterEndpoint=$(aws eks describe-cluster --name my-eks-cluster --query "cluster.endpoint" --output text) \
  --set aws.defaultInstanceProfile=KarpenterNodeInstanceProfile
```

> **Breakdown:**
>
> * `serviceAccount.create=false` → Use the service account created earlier
> * `aws.defaultInstanceProfile` → Nodes launched by Karpenter will use this IAM profile

---

## 1️⃣1️⃣ Karpenter Resource Files Overview

| Resource     | Purpose                                                                 |
| ------------ | ----------------------------------------------------------------------- |
| EC2NodeClass | Defines EC2 node properties: AMI family, subnets, SGs, instance profile |
| NodePool     | Node template: labels, taints, CPU/memory requirements, instance types  |
| Provisioner  | Optional; defines scaling strategies and rules (max/min limits)         |

---

## 1️⃣2️⃣ Creating an EC2NodeClass

**File: `ec2nodeclass.yaml`**


> **Explanation:**
>
> * `subnetSelector & securityGroupSelector` → Limits nodes to allowed subnets/SGs
> * `instanceProfile` → Assigns IAM role to EC2 nodes
> * `amiFamily` → AL2 for Kubernetes compatibility

**Command:**

```bash
kubectl apply -f ec2nodeclass.yaml
```

---

## 1️⃣3️⃣ Minimal NodePool Example

**File: `minimal-nodepool.yaml`**


```bash
kubectl apply -f minimal-nodepool.yaml
```

---

## 1️⃣4️⃣ Deploy a Test Pod

**File: `test-deployment.yaml`**


```bash
kubectl apply -f test-deployment.yaml
```

---

## 1️⃣5️⃣ Labels, Taints, and Tolerations in Karpenter

* Traditional Kubernetes → `kubectl label node` & `kubectl taint node`
* Karpenter → Defined in **NodePool templates**, nodes are pre-configured

---

## 1️⃣6️⃣ NodePools for Different Workloads - 

```bash
kubectl apply -f high-cpu-nodepool.yaml
kubectl apply -f gpu-nodepool.yaml
kubectl apply -f spot-nodepool.yaml
```

---

## 1️⃣7️⃣ Deploy a Pod With Tolerations

**File: `cpu-intensive-pod.yaml`**


```bash
kubectl apply -f cpu-intensive-pod.yaml
```

---

## 1️⃣8️⃣ Mental Model

```text
Pods
 ↓
Calculate optimal instance
 ↓
Launch node dynamically (with Node Instance Role + Instance Profile)
 ↓
Apply taints & labels
 ↓
Schedule pods
```

> Multi-NodePool setup allows **optimized, isolated, dynamically-scaled nodes** per workload type.
