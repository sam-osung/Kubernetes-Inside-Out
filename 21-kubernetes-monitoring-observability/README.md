# Kubernetes Monitoring & Observability – Production-Ready Guide (Slack-Only Alerts + Loki Logging)

This guide covers production-ready Kubernetes monitoring using **Prometheus**, **Grafana**, **Alertmanager**, and **Loki**, with optional **Datadog** integration. The focus is Slack-only alerts, full observability, and practical setup.

Monitoring is essential for:

* Observing cluster and application performance in real-time
* Detecting problems early
* Ensuring reliability

Prometheus + Grafana provides **full control and open-source flexibility**, while Datadog offers a **managed SaaS alternative**.

---

## 1️⃣ Prometheus – The Monitoring Engine

**Prometheus** collects metrics from all parts of your cluster and stores them as **time-series data**.

**Key features:**

* Collects metrics from nodes, pods, applications, and custom metrics
* Allows trend analysis, anomaly detection, and historical comparisons
* Scrapes metrics dynamically from Kubernetes using service discovery
* Integrates with **Alertmanager** for notifications

**Prometheus Components:**

| Component         | Role                                                                     |
| ----------------- | ------------------------------------------------------------------------ |
| Prometheus Server | Scrapes metrics, stores time-series, executes queries                    |
| Exporters         | Expose metrics from nodes/apps (e.g., Node Exporter, kube-state-metrics) |
| Alertmanager      | Evaluates alert rules, routes alerts to Slack                            |
| Pushgateway       | Handles short-lived jobs not running long enough to be scraped           |

---

## 2️⃣ PromQL – Querying Metrics

Prometheus uses **PromQL** for metric aggregation and alerting. Examples:

```bash
sum(rate(container_cpu_usage_seconds_total[1m])) by (pod)  # CPU usage per pod
sum(container_memory_usage_bytes) by (pod)                 # Memory usage per pod
increase(kube_pod_container_status_restarts_total[10m])   # Pod restart counts
```

PromQL powers dashboards and alert conditions.

---

## 3️⃣ Installing kube-prometheus-stack (KPS)

```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update
helm install prometheus prometheus-community/kube-prometheus-stack \
  -n monitoring \
  --create-namespace \
  -f alertmanager-values.yaml
```

**Important Notes:**

* The **full kube-prometheus-stack values** are in `values.yaml`. You can edit it to customize **resources, persistence, scraping intervals, dashboards, and alerting rules**.
* KPS comes with **200+ preconfigured alerting rules**. Each rule can be **enabled or disabled** in `values.yaml`.
* Example custom rules can be defined separately in `prometheus-rule.yaml`. This allows **adding or testing alerts independently**.

---

## 4️⃣ Exposing Prometheus, Grafana, and Alertmanager

**Prometheus:**

```bash
kubectl patch svc prometheus-kube-prometheus-prometheus -n monitoring -p '{"spec": {"type": "LoadBalancer"}}'
kubectl get svc prometheus-kube-prometheus-prometheus -n monitoring
```

Example endpoint:
`http://<PROMETHEUS_EXTERNAL_IP>:9090`

**Grafana:**

```bash
kubectl patch svc prometheus-grafana -n monitoring -p '{"spec": {"type": "LoadBalancer"}}'
kubectl get svc prometheus-grafana -n monitoring
```

Example endpoint:
`http://<GRAFANA_EXTERNAL_IP>`
username: `admin`
password: `prom-operator`

**Alertmanager:**

```bash
kubectl patch svc prometheus-kube-prometheus-alertmanager -n monitoring -p '{"spec": {"type": "LoadBalancer"}}'
kubectl get svc prometheus-kube-prometheus-alertmanager -n monitoring
```

Example endpoint:
`http://<ALERTMANAGER_EXTERNAL_IP>:9093`

✅ All three services are exposed via **LoadBalancer** for production-ready access.

---

## 5️⃣ Connecting Prometheus to Grafana

* Already configured in kube-prometheus-stack
* Optional manual step: **Configuration → Data Sources → Add Prometheus URL**

Example Prometheus URL:
`http://<PROMETHEUS_EXTERNAL_IP>:9090`

---

## 6️⃣ Importing Kubernetes Dashboards

* Kube-prometheus-stack comes with preconfigured dashboards
* You can import Grafana dashboards 315 (cluster overview) and 14584 (advanced workloads) from:
  [Grafana Dashboards](https://grafana.com/grafana/dashboards/)

Steps:

1. Open Grafana
2. Click **Create → Import**
3. Enter the dashboard ID
4. Select Prometheus as the data source
5. Click **Import**

---

## 7️⃣ Installing Loki & Promtail for Logging

```bash
helm repo add grafana https://grafana.github.io/helm-charts
helm repo update
helm install loki grafana/loki-stack -n monitoring -f loki-values.yaml
```

**Example `loki-values.yaml`:**


**Import Loki into Grafana:**

1. Go to **Configuration → Data Sources → Add data source**
2. Select **Loki**
3. URL: `http://loki.monitoring.svc.cluster.local:3100`
4. Click **Save & Test**

**Example Loki query:**

```logql
{job="kubernetes-pods"} |= "error"
```

This fetches all logs from pods containing the word `error`.

---

## 8️⃣ PrometheusRule – Production Alerts

**Purpose:** Defines alerting conditions for cluster health.

* Monitors thresholds such as Node CPU/Memory high, Pod Memory high, Pod restart spikes, Deployment replica mismatch, Disk space low
* Triggers alerts → Prometheus → Alertmanager → Slack

```bash
kubectl apply -f prometheus-rule.yaml
```

---

## 9️⃣ AlertmanagerConfig – Slack Alerts

**Purpose:** Routes Prometheus alerts to Slack.

* Configures grouping, wait intervals, and repeat intervals
* Sends alerts to a specific Slack channel via webhook

```bash
kubectl apply -f alertmanager-config.yaml
```

---

## 🔟 Sample Test Pod

**Purpose:** Triggers alerts for testing alerting rules.

* Example: Memory-hogging pod stresses cluster
* Helps validate **PodMemoryHigh** alerts

```bash
kubectl apply -f mem-hog.yaml
```

---

## 1️⃣1️⃣ Metrics vs Logs

| Aspect      | Metrics               | Logs                        |
| ----------- | --------------------- | --------------------------- |
| Type        | Numeric (CPU, memory) | Textual (errors, events)    |
| Granularity | Aggregated            | Detailed                    |
| Purpose     | Monitoring & alerting | Troubleshooting & debugging |

Logs complement metrics for full observability.

---

## 1️⃣2️⃣ Datadog – Managed SaaS Alternative

**Datadog** provides an integrated cloud-based monitoring platform:

* Metrics, logs, traces, and alerts in one place
* Prebuilt Kubernetes dashboards and integrations
* Official website: [https://www.datadoghq.com](https://www.datadoghq.com)
* Simplified setup using Helm:

```bash
helm repo add datadog https://helm.datadoghq.com
helm repo update
helm install datadog-agent datadog/datadog \
    --set datadog.apiKey=<YOUR_DATADOG_API_KEY> \
    --set datadog.site='datadoghq.com'
```

**Comparison:**

| Feature          | Prometheus + Grafana | Datadog           |
| ---------------- | -------------------- | ----------------- |
| Open Source      | Yes                  | No                |
| Metrics          | Yes                  | Yes               |
| Logs             | Optional             | Integrated        |
| Traces           | Optional             | Integrated        |
| Alerts           | Alertmanager         | Integrated in UI  |
| Setup Complexity | Manual               | SaaS / Simplified |

---

## ✅ Summary Workflow

1. **Prometheus** scrapes metrics → evaluates **PrometheusRule** alerts
2. **Alertmanager** routes alerts → Slack (**AlertmanagerConfig**)
3. **Grafana** visualizes metrics, dashboards, and Loki logs
4. Use **test pods** to validate alerts
