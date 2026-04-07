# Amazon EKS Access Entries Practical Guide (Account-Trust + Group Inline Policy)

This guide provides **step-by-step instructions** for setting up **Amazon EKS Access Entries** for multiple teams using **Account Trust + Group Inline Policy** approach. It includes **IAM group and role creation**, attaching policies, and mapping them to **namespace or cluster-wide access**, with brief explanations for each command.

---

## 1. Overview

Teams:

| Team   | Access Scope                 |
| ------ | ---------------------------- |
| QA     | Namespace-scoped (`staging`) |
| DevOps | Cluster-wide admin           |

Key Concepts:

* **IAM Group:** Organizes users (`qa-group`, `devops-group`)
* **IAM Role:** Contains permissions; can be assumed by users (`qa-role`, `devops-role`)
* **Trust Policy:** Role trusts the AWS account
* **Inline Policy on Group:** Allows users to assume the role
* **Access Entry:** Authenticates the IAM role with EKS
* **Access Policy:** Defines RBAC inside EKS (authorization)
* **Access Entry Type:** `STANDARD` for normal users; `SERVICE` for automation/service accounts

---

## 2. Create IAM Groups

1. AWS Console → IAM → **User Groups** → **Create group**
2. Name: `qa-group` → skip attaching policies → Create
3. Repeat for `devops-group`

---

## 3. Add Users to Groups

1. IAM → User Groups → Select `qa-group` → **Add users**
2. IAM → User Groups → Select `devops-group` → **Add users**

---

## 4. Create IAM Roles (Account Trust)

1. IAM → Roles → Create Role → Trusted entity: AWS account (root) → Role name: `qa-role` / `devops-role`
2. Role trust policy:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": { "AWS": "arn:aws:iam::123456789012:root" },
      "Action": "sts:AssumeRole"
    }
  ]
}
```

* The role trusts the account, so anyone allowed can assume it.

---

## 5. Attach Inline Policy to Groups

**DevOps Group Example:**

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": "sts:AssumeRole",
      "Resource": "arn:aws:iam::123456789012:role/devops-role"
    }
  ]
}
```

* Repeat for `qa-group` with `qa-role`.
* This allows **group members to assume the role**.

---

## 6. Create EKS Access Entries (Authentication)

```bash
aws eks create-access-entry \
--cluster-name my-cluster \
--principal-arn arn:aws:iam::123456789012:role/qa-role \
--type STANDARD

aws eks create-access-entry \
--cluster-name my-cluster \
--principal-arn arn:aws:iam::123456789012:role/devops-role \
--type STANDARD
```

**Explanation:**

* `aws eks create-access-entry`: Registers role ARN with EKS for authentication
* `--cluster-name`: Target EKS cluster
* `--principal-arn`: IAM role being mapped
* `--type STANDARD`: Normal user/role access (`SERVICE` type is for automation)

---

## 7. Common Access Policies in EKS (Authorization)

| Policy Name                   | Scope            | Description                 | Use Case                    |
| ----------------------------- | ---------------- | --------------------------- | --------------------------- |
| `AmazonEKSClusterAdminPolicy` | Cluster-wide     | Full admin privileges       | DevOps, cluster operators   |
| `AmazonEKSViewPolicy`         | Namespace        | Read-only access            | QA team, auditors           |
| `AmazonEKSNetworkPolicy`      | Namespace        | Manage networking           | Security teams              |
| `AmazonEKSServicePolicy`      | Service accounts | Pod access to AWS resources | Automation/service accounts |

### Associate Access Policies

**QA Team — Namespace-Scoped Read-Only**

```bash
aws eks associate-access-policy \
--cluster-name my-cluster \
--principal-arn arn:aws:iam::123456789012:role/qa-role \
--policy-arn arn:aws:eks::aws:cluster-access-policy/AmazonEKSViewPolicy \
--access-scope type=namespace,namespaces=staging
```

**DevOps Team — Cluster-Wide Admin**

```bash
aws eks associate-access-policy \
--cluster-name my-cluster \
--principal-arn arn:aws:iam::123456789012:role/devops-role \
--policy-arn arn:aws:eks::aws:cluster-access-policy/AmazonEKSClusterAdminPolicy \
--access-scope type=cluster
```

**Explanation:**

* `associate-access-policy`: Maps IAM role to RBAC policy in EKS
* `--access-scope`: Defines namespace or cluster-wide permissions

---

## 8. Verify Access

```bash
# QA User
aws sts assume-role --role-arn arn:aws:iam::123456789012:role/qa-role --role-session-name qa-session
aws eks update-kubeconfig --name my-cluster --region us-east-1
kubectl get pods -n staging   # ✅ works
kubectl get pods -n dev       # ❌ denied

# DevOps User
aws sts assume-role --role-arn arn:aws:iam::123456789012:role/devops-role --role-session-name devops-session
aws eks update-kubeconfig --name my-cluster --region us-east-1
kubectl get pods --all-namespaces  # ✅ works
```

> `aws sts assume-role`: Get temporary credentials for the role
> `aws eks update-kubeconfig`: Configures `kubectl` to use those credentials

---

## 9. Summary Table

| Team   | IAM Group    | IAM Role    | Access Entry Type | Policy                      | Scope             | Method                              |
| ------ | ------------ | ----------- | ----------------- | --------------------------- | ----------------- | ----------------------------------- |
| QA     | qa-group     | qa-role     | STANDARD          | AmazonEKSViewPolicy         | staging namespace | Account-trust + group inline policy |
| DevOps | devops-group | devops-role | STANDARD          | AmazonEKSClusterAdminPolicy | cluster-wide      | Account-trust + group inline policy |

---

## 10. Visual Flow

```text
IAM Group (qa-group / devops-group)
      │
      ▼
Inline policy: sts:AssumeRole -> qa-role / devops-role
      │
      ▼
Users explicitly assume role (manual or automation)
      │
      ▼
EKS Access Entry maps role ARN
      │
      ▼
Cluster / namespace access via RBAC
```

> ✅ This approach uses only Account Trust + Group Inline Policy and works fully with EKS.

---

