# Fluent Bit & Monitoring Deployment Guide

---

## How It Works

You need two things:

- **Logs** — what happened? (errors, events, messages from each pod)
- **Metrics** — how is it performing? (CPU, memory, request rates)

This is what Fluent Bit and the monitoring stack do:

```
                        ┌─────────────────────────────────────────────────────┐
                        │              Kubernetes Cluster (EKS)               │
                        │                                                     │
                        │   ┌──────────┐  ┌──────────┐  ┌──────────┐        │
                        │   │  Pod A   │  │  Pod B   │  │  Pod C   │        │
                        │   │ (app)    │  │ (app)    │  │ (app)    │        │
                        │   └────┬─────┘  └────┬─────┘  └────┬─────┘        │
                        │        │ logs         │ logs         │ logs         │
                        │        └──────────────┴─────────────┘              │
                        │                       │                             │
                        │              ┌─────────▼─────────┐                 │
                        │              │    Fluent Bit      │                 │
                        │              │  (DaemonSet -      │                 │
                        │              │  runs on every     │                 │
                        │              │  node)             │                 │
                        │              └─────────┬─────────┘                 │
                        │                        │ ships logs                │
                        │                        ▼                           │
                        │              ┌──────────────────┐                  │
                        │              │  Elasticsearch   │ ◄── search &     │
                        │              │  (Elastic Cloud) │    view logs     │
                        │              └──────────────────┘                  │
                        │                                                     │
                        │        metrics (CPU, memory, requests)              │
                        │   ┌──────────────────────────────────────┐         │
                        │   │           Prometheus                 │         │
                        │   │  (scrapes metrics from all pods      │         │
                        │   │   every 15 seconds)                  │         │
                        │   └──────────────┬───────────────────────┘         │
                        │                  │                                  │
                        │       ┌──────────┴──────────┐                      │
                        │       ▼                     ▼                      │
                        │  ┌─────────┐        ┌──────────────┐               │
                        │  │ Grafana │        │ Alertmanager │               │
                        │  │(charts &│        │(sends alerts │               │
                        │  │dashbrd) │        │ via email)   │               │
                        │  └─────────┘        └──────────────┘               │
                        └─────────────────────────────────────────────────────┘
```

### Architecture Diagram

> Open `slides/fluent-monitoring-architecture.excalidraw` in [Excalidraw](https://excalidraw.com) or the VS Code Excalidraw extension to view the full interactive diagram.

![Architecture](slides/fluent-monitoring-architecture.excalidraw)

---

### What each piece does

| Component | Role | Simple Analogy |
|-----------|------|----------------|
| **Fluent Bit** | Collects logs from every pod and ships them to Elasticsearch | 
| **Elasticsearch** | Stores and indexes all logs so you can search them | 
| **Prometheus** | Scrapes metrics (CPU, memory, errors) from pods every 15s | 
| **Grafana** | Visualises Prometheus metrics as dashboards and charts | 
| **Alertmanager** | Sends email/Slack alerts when something goes wrong | 
| **Node Exporter** | Exposes host-level metrics (disk, CPU) from each node | 

---

## How Metrics and Dashboards Work

### Step 1 — Every pod exposes a `/metrics` endpoint

Each pod serves a plain HTTP page at `/metrics` with raw numbers:

```
http_requests_total{method="GET", status="200"} 1523
container_memory_usage_bytes{pod="api-gateway"} 87031808
process_cpu_seconds_total 0.42
```

Your pharma services, Fluent Bit, Node Exporter, and Kubernetes itself all expose this.

---

### Step 2 — Prometheus scrapes (pulls) those endpoints every 15s

Prometheus does **not** receive pushed data. It reaches out and **pulls** from each target on a fixed interval. This is set in `envs/monitoring/prometheus-values.yaml`:

```yaml
prometheusSpec:
  retention: 15d       # keeps data for 15 days
  retentionSize: "8GB" # hard cap on disk
```

---

### Step 3 — PodMonitor tells Prometheus what to scrape

`k8s/monitoring/fluent-bit-podmonitor.yaml` registers Fluent Bit as a scrape target:

```yaml
kind: PodMonitor
spec:
  namespaceSelector:
    matchNames:
      - dev                            # look in the dev namespace
  selector:
    matchLabels:
      app: fluent-bit                  # find pods with this label
  podMetricsEndpoints:
    - path: /api/v1/metrics/prometheus
      interval: 30s                    # scrape every 30s
```

Prometheus finds every pod labelled `app: fluent-bit` in `dev` and hits that path every 30 seconds. These two flags in the values make Prometheus pick up **all** monitors across the whole cluster:

```yaml
serviceMonitorSelectorNilUsesHelmValues: false
podMonitorSelectorNilUsesHelmValues: false
```

---

### Step 4 — Node Exporter and Kube State Metrics add host and cluster data

| Component | What it exposes |
|-----------|----------------|
| **Node Exporter** | CPU, RAM, disk I/O, network — per node (host level) |
| **Kube State Metrics** | Pod restarts, deployment replicas, PVC status — k8s object state |

Both expose `/metrics` and Prometheus scrapes them the same way.

---

### Step 5 — Prometheus stores everything as time-series

Each metric becomes a row timestamped every 15 seconds:

```
http_requests_total{pod="api-gateway"} 1523  @ t=0s
http_requests_total{pod="api-gateway"} 1601  @ t=15s
http_requests_total{pod="api-gateway"} 1688  @ t=30s
```

Stored on the **10Gi PVC** (`prometheus-data`) for up to **15 days**.

---

### Step 6 — Grafana queries Prometheus with PromQL and renders charts

Grafana does not store metrics. It fires **PromQL** queries at Prometheus and draws the results. Examples:

```promql
rate(http_requests_total[5m])      # request rate over last 5 minutes
container_memory_usage_bytes       # memory usage per container
100 - avg(rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100  # CPU %
```

---

### Step 7 — The Fluent Bit dashboard is auto-provisioned at startup

This block in `prometheus-values.yaml` handles it automatically:

```yaml
grafana:
  sidecar:
    dashboards:
      enabled: true
      searchNamespace: ALL    # watches ALL namespaces for dashboard ConfigMaps

  dashboards:
    default:
      fluent-bit:
        gnetId: 7752          # downloads dashboard #7752 from grafana.com
        revision: 1
        datasource: Prometheus
```

When Grafana starts:
1. The **sidecar container** watches for ConfigMaps labelled `grafana_dashboard: "1"` across all namespaces — any new one is auto-loaded
2. Dashboard `gnetId: 7752` is pulled from grafana.com and placed in the **Pharma** folder automatically

---

### Step 8 — Alertmanager fires when a rule is breached

Pre-built alert rules are enabled via `defaultRules`. When a rule triggers (e.g. a pod keeps crashing), Prometheus sends it to Alertmanager, which groups and routes it:

```yaml
route:
  group_wait: 30s          # wait 30s to group related alerts together
  repeat_interval: 12h     # don't spam — re-alert every 12 hours
  receiver: "email-notifications"

receivers:
  - name: "email-notifications"
    email_configs:
      - to: "devops@pharma.com"
        send_resolved: true  # also emails when the issue is fixed
```

---

### The Full Flow

```
Every 15s / 30s:
  Prometheus ──scrapes──► /metrics on each pharma pod
                           /metrics on Node Exporter  (host stats)
                           /metrics on Kube State Metrics  (k8s stats)
                           /metrics on Fluent Bit  (log pipeline stats)
        │
        │  stores as time-series (10Gi PVC · 15 day retention)
        ▼
  Grafana ──PromQL──► Prometheus ──returns data──► renders charts
        │
  Alertmanager ──rule fires──► groups alerts──► emails devops@pharma.com
```

> **Key insight:** Nothing is pushed. Prometheus is the active collector — it reaches out, pulls, stores, and evaluates rules on a fixed interval.

---

## What is `/metrics` and How Does Grafana Use PromQL?

### What is `/metrics`?

It is a plain HTTP page that any app can serve. If you `curl` it, you get plain text like this:

```
# HELP http_requests_total Total number of HTTP requests
# TYPE http_requests_total counter
http_requests_total{method="GET",  status="200"} 1523
http_requests_total{method="POST", status="500"} 12

# HELP process_memory_bytes Memory used by the process
# TYPE process_memory_bytes gauge
process_memory_bytes 87031808
```

That is it. No special protocol. Just numbers with labels, served at `http://<pod-ip>:<port>/metrics`.

Every pod in your cluster — api-gateway, auth-service, Fluent Bit, even Kubernetes itself — serves this page.

---

### The Scrape → Store → Query Flow

```
┌─────────────────────────────────────────────────────────┐
│                                                         │
│   api-gateway:8080/metrics  ◄──┐                        │
│   auth-service:8080/metrics ◄──┤                        │
│   fluent-bit:2020/metrics   ◄──┼── Prometheus           │
│   node-exporter:9100/metrics◄──┤   pulls every 15s      │
│   kube-state-metrics/metrics◄──┘                        │
│                                         │               │
│                                         │ stores rows   │
│                                         ▼               │
│                                  time-series DB         │
│                              (10Gi PVC on EKS)          │
│                                                         │
│   t=0s   http_requests_total{pod="api-gateway"} 1523    │
│   t=15s  http_requests_total{pod="api-gateway"} 1601    │
│   t=30s  http_requests_total{pod="api-gateway"} 1688    │
│                                         │               │
│   Grafana asks: "give me the last       │               │
│   5 mins of request rate"               │               │
│          │                              │               │
│          └──── PromQL query ───────────►│               │
│                                         │               │
│          ◄──── returns numbers ─────────┘               │
│                                                         │
│          Grafana draws the chart                        │
└─────────────────────────────────────────────────────────┘
```

---

### What is PromQL?

PromQL is the query language Grafana uses to ask Prometheus questions. Think of it like SQL but for metrics.

| Question | PromQL |
|----------|--------|
| How many requests per second right now? | `rate(http_requests_total[5m])` |
| How much memory is each pod using? | `container_memory_usage_bytes` |
| What is CPU % per node? | `100 - avg(rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100` |
| How many pod restarts today? | `kube_pod_container_status_restarts_total` |

---

### What Grafana Actually Does

Grafana stores **nothing**. It is purely a visualisation layer:

```
Student opens Grafana dashboard
        │
        ▼
Grafana reads the dashboard JSON
(which has PromQL queries baked in)
        │
        ▼
Grafana fires those queries at Prometheus
        │
        ▼
Prometheus looks up its time-series DB
and returns the matching numbers
        │
        ▼
Grafana draws those numbers as a line chart,
bar chart, gauge, or table
        │
        ▼
Student sees a live chart updating every 30s
```

---

### Your Fluent Bit Dashboard (gnetId: 7752)

In `prometheus-values.yaml`:

```yaml
dashboards:
  default:
    fluent-bit:
      gnetId: 7752       # downloaded automatically from grafana.com
      datasource: Prometheus
```

Dashboard `7752` contains pre-written PromQL queries like:

```promql
rate(fluentbit_input_records_total[5m])   # log lines ingested per second
rate(fluentbit_output_errors_total[5m])   # failed log shipments
fluentbit_input_bytes_total               # total bytes ingested
```

Grafana runs these every 30 seconds and draws the charts automatically. You never need to write the PromQL yourself — but understanding it means you can build your own panels for your own services.

---

All manifests live on the `feature/fluent-bit-logging` branch. Do **not** merge to `main` — apply directly from the branch.

---

## How Dashboards Are Working Right Now (Live State)

Your Grafana pod has **3 containers** running inside it:

```
Pod: monitoring-grafana
├── grafana-sc-dashboard    ← the sidecar watcher
├── grafana-sc-datasources  ← datasource watcher
└── grafana                 ← the actual Grafana app
```

There are **two separate streams** loading dashboards in:

---

### Stream 1 — 28 dashboards via the sidecar

```
28 ConfigMaps in the cluster
(all labelled grafana_dashboard: "1")
        │
        ▼
grafana-sc-dashboard container
watches ALL namespaces every 60s
        │
        ▼
copies JSON files into /tmp/dashboards/
        │
        ▼
sidecarProvider reads /tmp/dashboards/
and loads dashboards into Grafana live
```

The sidecar checks every 60 seconds. Any new ConfigMap labelled `grafana_dashboard: "1"` is picked up automatically — no restart needed.

---

### Stream 2 — Fluent Bit dashboard from grafana.com

```
prometheus-values.yaml  →  gnetId: 7752
        │
        ▼
Grafana init container fetches it from grafana.com at startup
        │
        ▼
Written to: /var/lib/grafana/dashboards/default/fluent-bit.json
        │
        ▼
Dashboard provider config reads that path:

  providers:
  - name: default
    folder: Pharma        ← appears under the Pharma folder in Grafana
    path: /var/lib/grafana/dashboards/default
        │
        ▼
Loaded into Grafana under the "Pharma" folder
```

---

### What You See in Grafana Right Now

| Folder | Dashboards | Source |
|--------|-----------|--------|
| **Pharma** | Fluent Bit (1) | Downloaded from grafana.com at pod startup → `/var/lib/grafana/dashboards/default/fluent-bit.json` |
| **General** | 28 Kubernetes dashboards | kube-prometheus-stack ConfigMaps → sidecar → `/tmp/dashboards/` |

---

### Key Difference Between the Two Paths

| | Sidecar (28 dashboards) | Provider (Fluent Bit) |
|-|------------------------|----------------------|
| Loaded from | ConfigMaps in k8s | File on disk |
| Refreshes | Every 60s — live | Only on pod restart |
| Stored in | `/tmp/dashboards/` | `/var/lib/grafana/dashboards/default/` |
| How to add more | Create a labelled ConfigMap | Add `gnetId` to `prometheus-values.yaml` |

---

### How to Add Your Own Dashboard

**Option A — Label a ConfigMap** (picked up live by the sidecar):

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: my-api-gateway-dashboard
  namespace: dev
  labels:
    grafana_dashboard: "1"    # sidecar watches for this label
data:
  api-gateway.json: |
    { ... grafana dashboard JSON ... }
```

**Option B — Add to `prometheus-values.yaml`** (fetched from grafana.com at startup):

```yaml
dashboards:
  default:
    kubernetes-overview:
      gnetId: 15661    # any dashboard ID from grafana.com/dashboards
      datasource: Prometheus
```

---

## Prerequisites

### 1. Ensure nodes have enough capacity
Use `t3.medium` instances (supports ~17 pods per node). If starting fresh, create the node group with:

```bash
aws eks create-nodegroup \
  --cluster-name pharma-dev-cluster \
  --nodegroup-name pharma-dev-node-group-medium \
  --scaling-config minSize=2,maxSize=6,desiredSize=6 \
  --instance-types t3.medium \
  --ami-type AL2023_x86_64_STANDARD \
  --node-role arn:aws:iam::873135413040:role/pharma-dev-eks-node-group-role \
  --subnets subnet-06d1320aa157ef2ac subnet-0101546f0638a244a
```

### 2. Install the EBS CSI Driver addon

```bash
aws eks create-addon \
  --cluster-name pharma-dev-cluster \
  --addon-name aws-ebs-csi-driver \
  --resolve-conflicts OVERWRITE
```

Wait for it to become `ACTIVE`:

```bash
aws eks describe-addon --cluster-name pharma-dev-cluster \
  --addon-name aws-ebs-csi-driver \
  --query 'addon.status' --output text
```

Then annotate the service account with the IAM role:

```bash
kubectl annotate serviceaccount ebs-csi-controller-sa -n kube-system \
  eks.amazonaws.com/role-arn=arn:aws:iam::873135413040:role/pharma-dev-ebs-csi-driver-role \
  --overwrite

kubectl rollout restart deployment ebs-csi-controller -n kube-system
```

> **Note:** Ensure the IAM role trust policy references the correct cluster OIDC ID.
> Check with: `aws eks describe-cluster --name pharma-dev-cluster --query 'cluster.identity.oidc.issuer' --output text`

### 3. Create the gp2-csi StorageClass

```bash
kubectl apply -f - <<'EOF'
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: gp2-csi
  annotations:
    storageclass.kubernetes.io/is-default-class: "true"
provisioner: ebs.csi.aws.com
volumeBindingMode: WaitForFirstConsumer
parameters:
  type: gp2
  fsType: ext4
EOF
```

---

## Deploy

Apply the following in order:

### 1. Update the ArgoCD project (adds `monitoring` and `kube-system` destinations)

```bash
git show origin/feature/fluent-bit-logging:argocd/projects/pharma-project.yaml | kubectl apply -f -
```

### 2. Create namespaces

```bash
git show origin/feature/fluent-bit-logging:k8s/namespaces.yaml | kubectl apply -f -
```

### 3. Create PVCs

```bash
git show origin/feature/fluent-bit-logging:k8s/monitoring/pvc.yaml | kubectl apply -f -
```

| PVC | Size | Used by | Managed by |
|-----|------|---------|------------|
| `grafana` | 10Gi | Grafana | This file (`pvc.yaml`) |
| `prometheus-data` | 10Gi | Prometheus | Helm (auto-created) |
| `alertmanager-data` | 5Gi | Alertmanager | Helm (auto-created) |

### 4. Apply ArgoCD apps

```bash
git show origin/feature/fluent-bit-logging:argocd/apps/dev/fluent-bit-app.yaml | kubectl apply -f -
git show origin/feature/fluent-bit-logging:argocd/apps/dev/monitoring-app.yaml | kubectl apply -f -
```

---

## Verify

```bash
# ArgoCD app sync status
kubectl get applications -n argocd fluent-bit-dev monitoring

# All monitoring pods
kubectl get pods -n monitoring

# PVCs are bound
kubectl get pvc -n monitoring
```

Expected state — all pods `Running`, all PVCs `Bound`:

| Component | Pods |
|-----------|------|
| Prometheus | 2/2 |
| Alertmanager | 2/2 |
| Grafana | 3/3 |
| Kube Prometheus Operator | 1/1 |
| Kube State Metrics | 1/1 |
| Node Exporter | 1/1 per node |
