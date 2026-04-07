# EKS IAM & Kubernetes RBAC Step-by-Step Guide

This document covers everything from creating IAM groups and roles, mapping them to EKS, and defining Kubernetes RBAC.

---

## PART 1 – Create IAM groups and users (Console)

1. Open AWS Console → IAM

2. Create groups:

   * `qa-group`
   * `devops-group`

   **IAM → User groups → Create group**

3. Create users:

   * `bob` → QA
   * `alice` → DevOps

4. Add users to groups:

   * `bob` → `qa-group`
   * `alice` → `devops-group`

---

## PART 2 – Create IAM roles (Bridge to Kubernetes)

1. Create `qa-role` and `devops-role`:

   * IAM → Roles → Create role
   * Trusted entity type → AWS account → Your account
   * Do **not attach any permission yet**

⚠️ At this stage, roles have **no AWS or Kubernetes permissions**, only identities.

---

## PART 3 – Allow groups to assume the roles

1. For `qa-group`: Add inline policy:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": "sts:AssumeRole",
      "Resource": "arn:aws:iam::<ACCOUNT_ID>:role/qa-role"
    }
  ]
}
```

2. For `devops-group`, same policy but `Resource` points to `devops-role`.

> This **only allows switching identity**, does **not** grant access to AWS services or Kubernetes.

---

## PART 4 – (Optional) AWS service permissions

* Example for `devops-role`: Attach `ReadOnlyAccess` or `AdministratorAccess`.
* Affects **AWS services** only, **not Kubernetes**.

---

## PART 5 – Let EKS recognize the IAM roles

Edit `aws-auth` ConfigMap:

```bash
kubectl -n kube-system edit configmap aws-auth
```

Add mapping:

```yaml
mapRoles: |
  - rolearn: arn:aws:iam::<ACCOUNT_ID>:role/qa-role
    username: qa-role
    groups:
      - qa-group
  - rolearn: arn:aws:iam::<ACCOUNT_ID>:role/devops-role
    username: devops-role
    groups:
      - devops-group
```

> This allows IAM roles to **authenticate** with Kubernetes. Permissions are still zero.

---

## PART 6 – Kubernetes RBAC (Apply from GitHub)

Clone the repo containing all RBAC manifests:

```bash
git clone https://github.com/YOUR_REPO/eks-rbac-demo.git
cd eks-rbac-demo/rbac
```

### QA – list pods only

Apply QA role and binding:

```bash
kubectl apply -f qa-pod-reader-role.yaml
kubectl apply -f qa-rolebinding.yaml
```

### DevOps – list pods, services, deployments

Apply ClusterRole and ClusterRoleBinding:

```bash
kubectl apply -f devops-reader-clusterrole.yaml
kubectl apply -f devops-clusterrolebinding.yaml
```

Or apply all at once:

```bash
kubectl apply -f .
```

> The ClusterRole and ClusterRoleBinding files allow the `devops-group` to access cluster-wide resources.

---

## PART 7 – Quick test

```bash
kubectl auth can-i list pods --as-group=qa-group -n default
kubectl auth can-i list services --as-group=qa-group -n default
```

Expected output:

```
yes
no
```

---

## PART 8 – How the user logs in

### Bob (QA user)

```bash
aws configure
```

Then either:

```bash
aws sts assume-role \
  --role-arn arn:aws:iam::<ACCOUNT_ID>:role/qa-role \
  --role-session-name qa
```

Or use a profile:

```bash
aws eks update-kubeconfig --name my-cluster --profile qa
```

Check access:

```bash
kubectl get pods
```

Kubernetes sees group=`qa-group` → applies RBAC.

### Alice (DevOps user)

```bash
aws eks update-kubeconfig --name my-cluster --role-arn arn:aws:iam::<ACCOUNT_ID>:role/devops-role
kubectl get pods,svc,deployments --all-namespaces
```

---

## PART 9 – Key clarifications

* IAM policies (**AdministratorAccess, sts:AssumeRole**) **do not grant Kubernetes permissions**.
* Kubernetes RBAC is separate.
* AWS managed policies only control **AWS API access** (S3, EC2, EKS API, etc.).
* `aws-auth` ConfigMap maps IAM → Kubernetes identity.
* Pod ServiceAccounts are different and unrelated to user login.

---

## PART 10 – Golden rule

```
IAM decides who can talk to the cluster.
RBAC decides what they can do inside the cluster.
```
