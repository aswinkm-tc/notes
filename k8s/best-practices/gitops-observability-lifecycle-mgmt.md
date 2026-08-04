# GitOps, Observability & Lifecycle Management
## GitOps Engine Architecture: ArgoCD Pull vs. Push Pipelines
```
Push Model (Legacy CI/CD)                 GitOps Pull Model (ArgoCD / Flux)
┌──────────────────────────┐             ┌──────────────────────────┐
│ Jenkins / GitLab CI      │             │ ArgoCD Controller        │
│ holds cluster admin keys │             │ (Runs INSIDE cluster)    │
├──────────────────────────┤             ├──────────────────────────┤
│ Directly pushes via      │             │ Continuously pulls Git;  │
│ `kubectl apply` over net │             │ detects state drift;     │
│ (Security Risk).         │             │ reconciles `etcd` state. │
└──────────────────────────┘             └──────────────────────────┘
```
**The Pull Pattern:** ArgoCD runs an operator inside the cluster. It polls Git (or receives webhooks), compares the declared state in Git against the live state in etcd, and auto-reconciles drift.

**Key Advantage:** Cluster API credentials never leave the cluster control plane (eliminating external CI pipeline access to cluster secrets).

**Application repo vs. Infra repo separation:** Standard practice requires separating source code from deployment manifests. Pushing a code change triggers CI to build an image tag, update the deployment repo via a Pull Request (PR), which ArgoCD then reconciles.

##  Zero-Downtime Cluster Upgrades
When upgrading a production cluster (e.g., from K8s 1.28 -> 1.29), you must execute upgrades without dropping user traffic.

### Upgrade Sequence (Self-Hosted / kubeadm):
1. Control Plane First: Upgrade kube-apiserver, kube-controller-manager, and kube-scheduler one node at a time (kubeadm upgrade apply).
2. Worker Node Rolling Upgrade Loop:
    * Cordon Node: Mark as unschedulable (kubectl cordon node-1).
    * Drain Node: Evict active pods (kubectl drain node-1 --ignore-daemonsets --delete-emptydir-data). PDBs (PodDisruptionBudgets) are checked here to guarantee capacity before eviction.
    * Upgrade Host Components: Upgrade kubelet, kubectl, and containerd packages.
    * Uncordon Node: Mark node as ready (kubectl uncordon node-1).

### Managed Cloud (EKS/GKE):
1. Control plane is upgraded asynchronously via cloud API.
2. Node pools are upgraded via Blue/Green Node Pools (spin up a new node pool running target version, cordon/drain old pool, terminate old pool) or cloud-managed rolling node updates.

## Production Observability Stack
A modern 2026 telemetry architecture standardizes on OpenTelemetry (OTel) for data collection:
```
┌──────────────────────────────────────────────────────────┐
│ Applications (Auto-instrumented via OTel SDKs)           │
└────────────────────────────┬─────────────────────────────┘
                             │ OTLP Protocol (gRPC/HTTP)
                             ▼
┌──────────────────────────────────────────────────────────┐
│ OpenTelemetry Collector (DaemonSet / Deployment)         │
└────────┬───────────────────┬───────────────────┬─────────┘
         │ Prometheus        │ OTLP              │ Loki
         ▼                   ▼                   ▼
  ┌────────────────┐   ┌─────────────┐     ┌─────────────┐
  │ VictoriaMetrics│   │ Tempo/Jaeger│     │ Grafana Loki│
  │ / Thanos       │   │ (Traces)    │     │ (Logs)      │
  └────────────────┘   └─────────────┘     └─────────────┘
```

**Metrics Engine:** Prometheus Operator for local scrape targets, federated to Thanos or VictoriaMetrics for long-term historical storage and global multi-cluster views.

**Logs:** Grafana Loki or Vector shipping structured JSON logs directly from container stdout/stderr.

**Traces:** OpenTelemetry Collector shipping spans to Grafana Tempo or Jaeger.

