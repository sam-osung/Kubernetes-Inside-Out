# Kubernetes Logging Guide (EFK Stack)

---

# 1. Introduction to Logging in Kubernetes

Logging is one of the **most critical components of operating modern distributed systems**, especially when running applications inside Kubernetes clusters.

In traditional computing environments, applications typically ran on **dedicated physical servers or virtual machines**. When something went wrong, engineers could simply:

* SSH into the server
* Navigate to the application log directory
* Inspect log files manually

Typical log locations looked like this:

```/var/log/syslog
/var/log/nginx/access.log
/var/log/mysql/error.log
```

These logs were **easy to locate** because:

* Applications were running on **static machines**
* Infrastructure was **predictable**
* Services rarely moved between hosts

However, Kubernetes changes this model dramatically.

---

# Why Logging Becomes Hard in Kubernetes

Kubernetes is designed around **ephemeral, dynamic infrastructure**. This means:

* Containers are created and destroyed frequently
* Pods may restart automatically
* Workloads may move between nodes
* Nodes themselves can scale up or down

Because of this, logs are no longer stored in **one predictable place**.

Instead, logs become **distributed across the entire cluster**.

For example:

* A microservice may run **10 replicas**
* Those replicas may run across **5 nodes**
* Each replica generates **separate log files**

This creates several operational challenges.

### Problem 1 — Logs Are Scattered Across Nodes

Each node stores logs locally. If an application runs on multiple nodes, logs become **fragmented across the cluster**.

### Problem 2 — Pods Are Ephemeral

Pods can be destroyed and recreated at any time.

If logs are stored only inside the container or node filesystem, they may **disappear when the pod is deleted**.

### Problem 3 — Debugging Distributed Systems Is Hard

Modern applications often consist of **many microservices**.

A single request may pass through:

```
API Gateway
   ↓
Authentication Service
   ↓
Payment Service
   ↓
Database
```

If something fails, engineers must trace logs across **multiple services and nodes**.

Without centralized logging, this becomes extremely difficult.

---

# How Logging Works in Kubernetes by Default

Kubernetes follows a **simple logging philosophy**:

Applications should write logs to:

```
stdout
stderr
```

The container runtime (Docker or containerd) captures these logs and stores them on the **node filesystem**.

Typical log directories include:

```
/var/log/pods
/var/lib/docker/containers
```

These directories contain **all container logs running on the node**.

Limitations:

* Logs are **only on the node**
* **Not centralized**
* Hard to search and correlate
* Lost if nodes are removed

---

# The Goal of Centralized Logging

A centralized logging system allows engineers to:

* Collect logs from **all nodes**
* Store logs in a **central system**
* **Search and filter logs quickly**
* Correlate logs across services
* Investigate incidents efficiently
* Retain logs for long-term analysis

Centralized logging in Kubernetes is typically achieved by deploying **log collectors** on every node, which process logs and forward them to storage.

---

# Logging Stacks in Kubernetes

* **ELK** (Elasticsearch, Logstash, Kibana) — heavy, Logstash handles parsing
* **EFK** (Elasticsearch, Fluentd, Kibana) — lighter, Fluentd handles parsing and forwarding
* **Fluent Bit Stack** — optimized for low memory and CPU usage
* **Loki Stack** — stores logs without full indexing, reduces storage cost

---

# EFK Stack on Kubernetes (Helm Installation)

EFK = **Elasticsearch + Fluent Bit + Kibana**
This stack allows you to **collect, store, and visualize logs** in Kubernetes.

---

## 1️⃣ Create Logging Namespace

```bash
kubectl create namespace logging
```

* Creates a dedicated namespace for all logging components.
* Keeps logging resources separate from application workloads.

---

## 2️⃣ Install Elasticsearch via Helm

```bash
helm repo add elastic https://helm.elastic.co
helm repo update

helm install elasticsearch \
  --namespace logging \
  --set replicas=1 \
  --set volumeClaimTemplate.storageClassName=gp2 \
  --set persistence.labels.enabled=true \
  elastic/elasticsearch
```

**Explanation:**

* **replicas=1** → single Elasticsearch node (good for testing).
* **volumeClaimTemplate** → automatically provisions persistent volumes for logs.
* **persistence.labels.enabled** → ensures PVCs are properly labeled for tracking and management.

---

### 3️⃣ Retrieve Elasticsearch Credentials

```bash
# Retrieve username
kubectl get secret elasticsearch-master-credentials -n logging -ojsonpath='{.data.username}' | base64 -d

# Retrieve password
kubectl get secret elasticsearch-master-credentials -n logging -ojsonpath='{.data.password}' | base64 -d
```

* Base64 decode to get plain text.
* Note them down; you’ll use them for Kibana and Fluent Bit.

---

## 4️⃣ Install Kibana via Helm

```bash
helm install kibana \
  --namespace logging \
  --set service.type=LoadBalancer \
  elastic/kibana
```

**Explanation:**

* **LoadBalancer** → exposes Kibana outside the cluster for easy access.
* Kibana connects automatically to Elasticsearch in the same namespace.

---

## 5️⃣ Install Fluent Bit via Helm

```bash
helm repo add fluent https://fluent.github.io/helm-charts
helm repo update

helm install fluent-bit fluent/fluent-bit \
  -n logging \
  -f fluentbit-values.yaml
```

**Explanation:**

* Fluent Bit **collects logs** from application pods and node system logs.
* Reads logs from `/var/log/containers/*.log`.
* Enriches logs with **Kubernetes metadata** (pod name, namespace, labels).
* Sends logs to **Elasticsearch** using the credentials retrieved earlier.

---

## 6️⃣ Fluent Bit Values Overview (`fluentbit-values.yaml`)


* Replace `<ELASTIC_USERNAME>` and `<ELASTIC_PASSWORD>` with the secrets retrieved in step 3.
* Flush interval and log level control how often logs are sent and verbosity.
* Kubernetes filter adds pod metadata for easy visualization in Kibana.

---

## 7️⃣ Access Kibana Dashboard

```bash
kubectl get svc -n logging
```

* Look for **Kibana LoadBalancer IP** or hostname.
* Open in browser: `http://<LOADBALANCER_IP>:5601`
* Log in using the **Elasticsearch username/password** from step 3.

---

## 8️⃣ Test Logging Pipeline

```bash
kubectl apply -f test-logger.yaml
```

# EFK Stack: Verify Logs & Create Data View in Kibana

## 1️⃣ Verify Elasticsearch Indices

Run the following to check your indices:

```bash
kubectl exec -n logging <elasticsearch-pod> -- \
curl -k -u elastic:<elasticsearch-password> https://localhost:9200/_cat/indices?v
```

You should see something like:

```text
health status index               uuid                   pri rep docs.count docs.deleted store.size pri.store.size
yellow open   logstash-2026.03.18 UdU7qXkNSOG1Y24Nw38Zfw   1   1        163            0    290.4kb        290.4kb
```

✅ This confirms logs are successfully indexed.

---

## 2️⃣ Make Logs Visible in Kibana

Kibana needs a **Data View** to know which indices to read.

### Step A: Create Data View

1. Go to: **Kibana → Stack Management → Data Views → Create Data View**
2. Fill in:

* **Name:** `logstash-logs`
* **Index pattern:** `logstash-*`
* **Time field:** `@timestamp`

3. Save the data view.

### Step B: Explore Logs in Discover

1. Navigate to **Discover**
2. Select your new data view (`logstash-logs`)
3. Set **time range** (top right) → **Last 15 minutes**
4. You should now see logs from your pods, including your `test-logger` pod.

---

## ✅ Tip

If you don’t see logs:

* Make sure Fluent Bit is running and configured correctly
* Check your index in Elasticsearch matches the pattern `logstash-*`
* Adjust Kibana time range to include recent logs

---

This ensures your

---

## 🔄 Final Log Flow

```
Application Pods
      │
      ▼
Fluent Bit DaemonSet
      │
      ▼
Elasticsearch StatefulSet
      │
      ▼
Kibana Dashboard
```

* Application logs → node log files → Fluent Bit → Elasticsearch → Kibana.
