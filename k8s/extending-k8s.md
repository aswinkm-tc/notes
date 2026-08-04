# Extending Kubernetes
Extending the Kubernetes API is fundamentally about choosing where state is stored, how the request pipeline is executed, and how much control you need over subresources and backend storage.

## Request Pipeline Architecture: CRDs vs. API Aggregation
When a request hits kube-apiserver, the routing path differs significantly based on whether the resource is managed natively by CRDs or proxied to an Extension API Server.  
```
                [ Incoming HTTP/2 Request ]
                              │
                              ▼
                   [ kube-apiserver ]
             Authentication & Authorization (RBAC)
                              │
              ┌───────────────┴───────────────┐
              ▼                               ▼
    [ CustomResourceDefinition ]      [ APIService (Aggregated) ]
              │                               │
              ▼                               ▼
  CRD Controller / apiextensions    mTLS Proxy (kube-aggregator)
  In-memory Schema Validation                 │
              │                               ▼
  Mutating / Validating Webhooks    [ Extension API Server ]
  (CEL Validation / Webhook)        (e.g., metrics-server, KubeVirt)
              │                               │
              ▼                               ▼
   Conversion Webhook (if needed)    Custom Business Logic & Validation
              │                               │
              ▼                               ▼
     etcd Storage Engine             Custom Storage Engine
    (/registry/mygroup/...)          (etcd, SQL, TSDB, Virtual In-Mem)
```
### The kube-aggregator Reverse Proxy
The Aggregation Layer runs directly inside kube-apiserver as the kube-aggregator module.

* You register an APIService object (e.g., v1beta1.metrics.k8s.io) pointing to a Kubernetes Service. 
* kube-apiserver terminates initial client TLS, authenticates and authorizes the request via RBAC, injects user header information (X-Remote-User, X-Remote-Group), and proxies the request over mTLS to your Extension API Server.  

### Architectural Comparison Matrix
| Architectural Dimension | Custom Resource Definitions (CRDs) | Aggregated API Server (APIService) |
|-|-|-|
| Control Plane Overhead | Zero  additional control plane binaries. Managed entirely inside core kube-apiserver (apiextensions-apiserver). | Requires deploying, scaling, and maintaining a custom API server binary + CA certificates. |
| Storage Backend | **Hardcoded to primary etcd**. Shares disk/memory bounds with core cluster objects. | **Arbitrary**. Can use external etcd, PostgreSQL, TSDB, or run entirely in-memory (stateless). |
| Custom Subresources | Restricted to /status and /scale. | **Unlimited**. Can implement /exec, /logs, /console, /restart (e.g., KubeVirt).|
| Protobuf Support | JSON on the wire (internal serialization handles etcd Protobuf). | Full native Protobuf gRPC/HTTP endpoint support. |
| Strategic Merge Patch | Only supports JSONPatch (application/json-patch+json) and MergePatch. | Supports Strategic Merge Patch (application/strategic-merge-patch+json).
| Declarative Validation | OpenAPI v3 schemas + server-side CEL (Common Expression Language) expressions. | Imperative Go logic baked directly into the custom API server handlers. |

### CRD Multi Versioning
When a CRD serves multiple versions (e.g., v1alpha1, v1beta1, v1) and a client reads or writes v1alpha1 while the stored version in etcd is v1, kube-apiserver delegates conversion to an external Conversion Webhook.

```
                        Client Requests (v1alpha1) ──► kube-apiserver
                                    │
                                    ▼ (Detects v1alpha1 != Stored v1)
                         [ Conversion Review ]
                                    │
                                    ▼ HTTP POST (JSON Payload)
                         [ Conversion Webhook ]
                                    │
                         ┌──────────┴──────────┐
                         │ Convert v1alpha1    │
                         │    to Hub (v1)      │
                         └──────────┬──────────┘
                                    │
                       ▼ HTTP 200 (Converted Object)
                    kube-apiserver ──► Persist v1 to etcd
 ```

**The ConversionReview Contract**
kube-apiserver sends a ConversionReview request payload containing an array of objects to convert and the desiredAPIVersion:
```json
{
  "apiVersion": "apiextensions.k8s.io/v1",
  "kind": "ConversionReview",
  "request": {
    "uid": "705ab4f5-6393-11e8-b7cc-42010a800002",
    "desiredAPIVersion": "stable.example.com/v1",
    "objects": [
      {
        "apiVersion": "stable.example.com/v1alpha1",
        "kind": "CronTab",
        "metadata": { "name": "my-cron" },
        "spec": { "schedule": "* * * * *" }
      }
    ]
  }
}
```
**Lossless Conversion Rule:** The webhook must preserve all data when converting down and up (e.g., v1 -> v1alpha1 -> v1). Data loss during round-trips breaks controller state reconciliation loops.

**Annotations as Fallbacks:** If a new field exists in v1 but has no structural equivalent in v1alpha1, conversion webhooks typically serialize the unmapped fields into a JSON string inside an annotation (e.g., [crontab.example.com/preserved-v1-data](https://crontab.example.com/preserved-v1-data)) to prevent dropping information when down-converting.

### Custom Storage Engines (API Aggregation)
The single biggest technical driver for using the API Aggregation Layer over CRDs is bypassing etcd.

#### Example: Metrics Server(metrics.k8s.io)
Problem: Metrics (pod CPU/Memory) are time-series datapoints generated every few seconds. Writing metrics to etcd would cause catastrophic storage bloat and rapid key compaction cycles.

Solution: metrics-server implements an Aggregated API Server. It holds metrics in an in-memory ring buffer scraped directly from Kubelet /stats/summary endpoints. It exposes GET /apis/metrics.k8s.io/v1beta1/pods without touching etcd at all.

#### Example: Custom Database Backends
By using apiserver builder libraries (such as k8s.io/apiserver), you implement the storage.Interface interface in Go:

```go
// Custom storage engine contract inside k8s.io/apiserver
type Interface interface {
    Create(ctx context.Context, key string, obj, out runtime.Object, ttl uint64) error
    Get(ctx context.Context, key string, opts GetOptions, objPtr runtime.Object) error
    List(ctx context.Context, key string, opts ListOptions, listObj runtime.Object) error
    Watch(ctx context.Context, key string, opts ListOptions) (watch.Interface, error)
}
```

By substituting etcd3.Store with a custom implementation, your API server can:

* Read/write directly from a Relational DB (PostgreSQL) for transactional compliance.

* Act as an API translation layer over non-Kubernetes backends (e.g., mapping kubectl get virtualmachines to a VMware vSphere API).

#### Decision Framework
```
Do you need custom subresources (beyond /status & /scale), 
non-etcd storage, or ultra-high write frequency?
              │
      ┌───────┴───────┐
     YES              NO
      │               │
      ▼               ▼
Use Aggregated     Does your validation/conversion logic require 
API Server         complex Go business logic beyond CEL & OpenAPI?
                      │
              ┌───────┴───────┐
             YES              NO
              │               │
              ▼               ▼
        Use CRDs with     Use Declarative CRDs 
       Webhooks (CEL)     (Validation + OpenAPI)

```


