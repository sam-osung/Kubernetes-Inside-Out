# Kubernetes ETCD Backup & Restore — Production Guide

---

## 1. What is ETCD in Kubernetes

ETCD is a distributed, strongly consistent key-value store used by Kubernetes to store:

* Entire cluster state
* API objects (Pods, Deployments, Services, Nodes, Secrets, ConfigMaps)
* Cluster metadata
* Leader election data

Every `kubectl` command interacts with ETCD through the API server.

> Losing ETCD = Losing cluster state.

---

## 2. ETCD Architecture

### Single Node

* Simple, dev/test clusters
* Risky for production

### Multi-node (HA)

* 3 or 5 nodes
* Uses Raft consensus
* Backup strategy must consider quorum and which node to snapshot from

---

## 3. Why Simple YAML Backup is NOT Enough

```bash
kubectl get all -o yaml > backup.yaml
```

**Limitations:**

* No persistent volume data
* No secrets integrity
* No cluster state/history
* Cannot restore cluster fully

---

## 4. When to Backup ETCD

* Before cluster upgrades
* Before control-plane changes
* Before applying CRDs or major configs
* Scheduled backups every 6-24 hours

---

## 5. ETCD Deployment

In kubeadm clusters, ETCD runs as a static pod:

```
/etc/kubernetes/manifests/etcd.yaml
```

Kubelet watches this directory and manages the pod automatically.

---

## 6. ETCD Configuration

### Important Flags

* `--data-dir=/var/lib/etcd` → Stores WAL, snapshots, cluster data
* `--listen-client-urls=https://127.0.0.1:2379` → API server endpoint
* `--listen-peer-urls=https://<IP>:2380` → HA communication
* Certificates in `/etc/kubernetes/pki/etcd/` (`ca.crt`, `server.crt`, `server.key`)

---

## 7. ETCD Backup (Snapshot)

### Step 1: Set API Version

```bash
export ETCDCTL_API=3
```

### Step 2: Identify Certificates

```bash
kubectl describe pod etcd-<node> -n kube-system
```

### Step 3: Create Snapshot

```bash
etcdctl --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key \
  snapshot save /opt/etcd-backup.db
```

### Step 4: Verify Snapshot

```bash
etcdutl snapshot status /opt/etcd-backup.db --write-out=table
```

OR

```bash
etcdctl snapshot status /opt/etcd-backup.db --write-out=table
```

---

## 8. ETCD Restore

### Step 1: Restore Snapshot

```bash
ETCDCTL_API=3 etcdctl snapshot restore /opt/etcd-backup.db \
  --data-dir=/var/lib/etcd-restore
```

### Step 2: Inspect Restored Data

```bash
ls /var/lib/etcd-restore
```

Expected: `member/`

### Step 3: Modify ETCD Manifest

```bash
sudo nano /etc/kubernetes/manifests/etcd.yaml
```

* Update `--data-dir` to `/var/lib/etcd-restore`
* Fix volumeMounts if necessary

### Step 4: Restart Kubelet

```bash
sudo systemctl daemon-reload
sudo systemctl restart kubelet
```

### Step 5: Validate

```bash
kubectl get pods -n kube-system
kubectl describe pod etcd-<node> -n kube-system
kubectl get nodes
kubectl get pods -A
```

---

## 9. Automation Example

### Cron Job

```
0 */6 * * * /usr/local/bin/etcd-backup.sh
```

### Sample Backup Script

```bash
#!/bin/bash

export ETCDCTL_API=3
BACKUP_DIR="/opt/etcd-backups"
DATE=$(date +%Y-%m-%d-%H-%M)

etcdctl \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key \
  snapshot save ${BACKUP_DIR}/backup-${DATE}.db
```

### Remote Storage

* AWS S3
* GCS
* Azure Blob

---

## 10. Managed Kubernetes Backup

* No direct ETCD access in EKS, GKE, AKS
* Use **Velero** or cloud snapshots for full cluster backup

---

## 11. ETCD vs Persistent Volume Backups

| Component    | Tool              |
| ------------ | ----------------- |
| ETCD         | etcdctl           |
| PV Data      | Storage snapshots |
| Full Cluster | Velero            |

---

## 12. Security Best Practices

* Restrict access to ETCD certs and backup files
* Encrypt backups at rest
* Verify snapshots before restore

---

## 13. Common Mistakes

| Mistake                      | Impact              |
| ---------------------------- | ------------------- |
| Overwriting old data-dir     | Permanent data loss |
| Not verifying snapshot       | Restore failure     |
| Wrong cert paths             | Backup fails        |
| Restoring on running cluster | Corruption          |
| Ignoring PVs                 | Data loss           |

---

## 14. Conclusion

* ETCD is the brain of your cluster
* Regular, verified backups are mandatory
* Restore must be carefully planned
* Automate and secure backups for production readiness
