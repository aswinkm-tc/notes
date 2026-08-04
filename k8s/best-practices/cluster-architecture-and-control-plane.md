# Cluster Architecture and Control Plane

## Highly available etcd
Quorum Math: etcd uses Raft consensus. Quorum required is `Q = \[ N/2 \] + 1`.
* A 3-node cluster tolerates 1 failure Q=2.
* A 5-node cluster tolerates 2 failures Q=3.
* 6 nodes is a anti-pattern: It provides the same fault tolerance as 5 nodes (tolerates 2 failures, Q=4), but increases network write overhead during consensus rounds.

**Disk I/O Sensitivity:** etcd writes WAL (Write-Ahead Logs) to disk synchronously (fdatasync). High disk latency (>10) stalls etcd, causing leader election churn, which leads to kube-apiserver request timeouts and cascading API failures across the cluster.

**Database Maintenance:** By default, etcd has a 2GB soft / 8GB hard storage limit. Event churn, large objects, and revisions cause write amplification. Production clusters require scheduled auto-compaction (--auto-compaction-retention) and defragmentation (etcdctl defrag) to prevent hitting NOSPACE errors.

## Control Plane Components
```
                       ┌─────────────────────────┐
                       │   etcd (Distributed)    │
                       └────────────▲────────────┘
                                    │ gRPC
                       ┌────────────┴─────────────┐
                       │    kube-apiserver        │
                       │ (Stateless, horizontal)  │
                       └▲───────────▲────────────▲┘
                        │           │            │
      ┌─────────────────┴─┐   ┌─────┴─────────┐  │
      │kube-scheduler     │   │kube-controller│  │
      │(Leader-elected)   │   │-manager       │  │
      └───────────────────┘   └───────────────┘  │
                                                 │
                                    ┌────────────┴─────────┐
                                    │ Workers (Kubelet/CNI)│
                                    └──────────────────────┘
```
### kube-apiserver (Stateless Horizontal Scaling):
The only component that speaks directly to etcd.

API Priority & Fairness (APF): Replaced legacy max-inflight limits in recent Kubernetes releases. Categorizes traffic into flow schemas and priority levels so heavy worker node heartbeats or automated controller loops don't starve human kubectl operators or critical system callers.

### kube-controller-manager & kube-scheduler (Active-Passive / Leader Election):
Run in active-passive HA using lease objects (coordination.k8s.io/v1). Only the leader executes reconciliation loops or node scheduling evaluations.

### Self-Hosted vs. Cloud Managed Trade-Offs:
**Managed (EKS/GKE/AKS):** Cloud provider handles etcd topology, APF tuning, master node auto-healing, and master-to-worker network tunnels (Konnectivity / API Server Load Balancers).

**Self-Hosted (kubeadm, RKE2, Talos):** You manage control-plane load balancers (HAProxy/Keepalived), PKI certificate rotation (/etc/kubernetes/pki), snapshot orchestration, and raw kernel tuning (sysctl network limits) for API nodes.


## Cluster Sizing: Monolith vs. Multi-Cluster Fleet
### The Single Large Cluster (Consolidated):
Pros: Higher resource utilization, centralized monitoring/ingress, lower control plane overhead cost.

Cons: Huge blast radius (e.g., CNI daemonset crash takes down everything), large etcd footprint, complex RBAC/tenant isolation, API server throttling.

### The Multi-Cluster Strategy (Fleet):
Pros: Small blast radius, natural isolation (compliance, env, region), simpler upgrades (can drain/kill whole clusters via GitOps).

Cons: Overhead cost of multiple control planes, fragmented observability, cross-cluster service discovery challenges (requires Service Mesh / Submariner / Multi-Cluster Services API).


