# Networking, Ingress & Traffic Management
Kubernetes networking at scale moves away from legacy iptables rules toward eBPF, Gateway API, and sidecarless service meshes.

## CNI Evolution: iptables vs. IPVS vs. eBPF (Cilium)
```
Legacy Kube-Proxy (iptables)            Modern eBPF CNI (Cilium)
┌──────────────────────────────┐       ┌──────────────────────────────┐
│ Sequentially evaluates rules │       │ Kernel-level eBPF map lookups│
│ O(N) lookup time.            │       │ O(1) constant time lookup.   │
├──────────────────────────────┤       ├──────────────────────────────┤
│ High CPU overhead at 10k+    │       │ Bypasses socket layer        │
│ Services/Endpoints.          │       │ overhead via sockmap.        │
├──────────────────────────────┤       ├──────────────────────────────┤
│ Packet filter mode only.     │       │ Deep L3-L7 visibility, native│
│ Needs external tools for L7. │       │ WireGuard encryption, Hubble.│
└──────────────────────────────┘       └──────────────────────────────┘
```

* The Problem with iptables (kube-proxy default): Evaluates packet rules sequentially. At 10,000+ Services, updating and traversing O(N) iptables chains causes high kernel CPU utilization and packet delivery delays.
* IPVS (IP Virtual Server): Uses hash tables O(1) lookup time) instead of sequential rule lists, providing scaling for large clusters, but lacks L7 network insight.
* eBPF (Cilium): Replaces kube-proxy entirely. Runs sandboxed bytecode directly inside the Linux kernel on socket events. Eliminates TCP/IP stack traversal overhead for local pod-to-pod communication on the same node using eBPF sockmap.

## Ingress Controller vs. Kubernetes Gateway API
The traditional Ingress resource (networking.k8s.io/v1) is monolithic—a single manifest mixes infrastructure concerns (TLS certs, DNS) with application routing rules (paths, headers).

Kubernetes Gateway API breaks this into role-oriented, decoupled resources:
```
  ┌─────────────────────────────────────────┐
  │ GatewayClass (Infra Provider / Cilium)  │
  └────────────────────▲────────────────────┘
                       │ Defines implementation
  ┌────────────────────┴────────────────────┐
  │ Gateway (Cluster Admin / Ops)           │ -> Sets IP, VIP, TLS Certs, Ports
  └────────────────────▲────────────────────┘
                       │ Binds routes via Label Selectors
  ┌────────────────────┴────────────────────┐
  │ HTTPRoute / GRPCRoute (App Developers)   │ -> Defines /api/v1 -> service-a
  └─────────────────────────────────────────┘
```

**Role Separation:** Cluster Admins manage Gateway (listening ports, TLS certs, external IPs); App teams manage HTTPRoute (path routing, header manipulation, traffic splitting).

**Cross-Namespace Routing:** Allows an HTTPRoute in namespace-app to attach to a Gateway in namespace-infra, enforcing clear security boundaries.

## Service Mesh Architecture: Sidecar vs. Ambient (Sidecarless)
### Sidecar Pattern (Istio Classic / Linkerd):
Injects an Envoy proxy container into every single Pod.

Pain Points: High memory/CPU footprint across thousands of pods, lifecycle coupling (proxy must start before app), complex pod initialization.

### Ambient Mesh (Istio Ambient / Cilium Service Mesh):
Decouples proxying into two layers:

**ztunnel (Zero-Trust Tunnel):** Runs as a node-level DaemonSet. Handles L4 secure mTLS tunnel between nodes.

**Waypoint Proxy:** Runs per-namespace or per-service as a standalone deployment. Handles L7 processing (HTTP routing, retries, authorization policies) only when explicitly required.
