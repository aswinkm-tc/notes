# Compute, Scheduling & Autoscaling
## Pod Scheduling & Topology Constraints
### Node Affinity vs. Pod Anti-Affinity:
**nodeAffinity:** Directs pods to specific hardware (e.g., ARM64 vs x86, GPU nodes).

**podAntiAffinity:** Prevents pods of the same deployment from landing on the same node/rack. Warning: Hard requiredDuringSchedulingIgnoredDuringExecution pod anti-affinity scales poorly (computational cost is O(N^2) in kube-scheduler as pod count grows).

### Topology Spread Constraints: The modern, scalable alternative to Pod Anti-Affinity.
```yaml
topologySpreadConstraints:
- maxSkew: 1
  topologyKey: topology.kubernetes.io/zone
  whenUnsatisfiable: DoNotSchedule # or ScheduleAnyway
  labelSelector:
    matchLabels:
      app: payment-api
```
Guarantees that the difference in pod count between any two zones never exceeds maxSkew: 1.

## Resource Quality of Service (QoS) & OOM Killer Mechanics
When the node runs low on memory, the Linux kernel's Out-Of-Memory (OOM) Killer calculates an oom_score for each process:
> oom_score = memory_usage - oom_score_adj

Kubernetes translates Pod specs into three distinct QoS classes, setting oom_score_adj accordingly

```
  ┌─────────────────────────────────────────────────────────────┐
  │                      Guaranteed QoS                         │
  │  (Requests == Limits for ALL containers)                    │
  │  oom_score_adj = -997 (Last to be killed)                   │
  └──────────────────────────────▲──────────────────────────────┘
                                 │
  ┌──────────────────────────────┴──────────────────────────────┐
  │                      Burstable QoS                          │
  │  (Requests < Limits or Requests set without Limits)         │
  │  oom_score_adj = 2 to 999 (Killed based on % of request)    │
  └──────────────────────────────▲──────────────────────────────┘
                                 │
  ┌──────────────────────────────┴──────────────────────────────┐
  │                      BestEffort QoS                         │
  │  (No Requests, No Limits)                                   │
  │  oom_score_adj = 1000 (FIRST TO BE KILLED)                  │
  └─────────────────────────────────────────────────────────────┘
```

## Autoscaling in 2026: HPA + KEDA + Karpenter
### HPA (Horizontal Pod Autoscaler)
Queries metrics via metrics-server (metrics.k8s.io) or custom Prometheus metrics (custom.metrics.k8s.io).

Math formula:
```
Target Replicas = Current Replicas x ( Current Metric Value / Target Metric Value )
```

### KEDA (Kubernetes Event-driven Autoscaling)
* Extends standard HPA by introducing CRDs (ScaledObject, ScaledJob).
* Scales pods to/from zero based on external event sources (e.g., Kafka consumer group lag, RabbitMQ queues, Azure Service Bus) before CPU/Memory utilization actually spikes.

### Node Autoscaling: Karpenter vs. Legacy Cluster Autoscaler (CA)
```
Legacy Cluster Autoscaler                 Karpenter (Modern Standard)
┌──────────────────────────┐             ┌──────────────────────────┐
│ Evaluates unschedulable  │             │ Bypasses K8s NodeGroups/ │
│ Pods -> Modifies Cloud   │             │ Auto Scaling Groups (ASG)│
│ Auto Scaling Groups (ASG)│             │ directly.                │
├──────────────────────────┤             ├──────────────────────────┤
│ Takes 3-7 minutes to     │             │ Evaluates pending Pods & │
│ provision nodes.         │             │ launches exact fit EC2/  │
├──────────────────────────┤             │ Compute instances via    │
│ Slow consolidation;      │             │ Cloud API in seconds.    │
│ constrained by ASG types.│             │ Continuous bin-packing.  │
└──────────────────────────┘             └──────────────────────────┘
```

