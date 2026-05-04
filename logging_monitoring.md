# Fluent Bit & Monitoring Deployment Guide

---

## How It Works (For Students)

Think of your Kubernetes cluster as a factory with many machines (pods) running 24/7. You need two things:

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
| **Fluent Bit** | Collects logs from every pod and ships them to Elasticsearch | A postman that picks up letters from every house |
| **Elasticsearch** | Stores and indexes all logs so you can search them | A filing cabinet you can search instantly |
| **Prometheus** | Scrapes metrics (CPU, memory, errors) from pods every 15s | A health monitor taking readings every 15 seconds |
| **Grafana** | Visualises Prometheus metrics as dashboards and charts | The screen showing all the health readings as graphs |
| **Alertmanager** | Sends email/Slack alerts when something goes wrong | An alarm that calls you when a reading goes critical |
| **Node Exporter** | Exposes host-level metrics (disk, CPU) from each node | A sensor attached to each machine in the factory |

---

All manifests live on the `feature/fluent-bit-logging` branch. Do **not** merge to `main` — apply directly from the branch.

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

| PVC | Size | Used by |
|-----|------|---------|
| `grafana` | 10Gi | Grafana |
| `prometheus-data` | 10Gi | Prometheus |
| `alertmanager-data` | 5Gi | Alertmanager |

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
