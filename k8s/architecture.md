# Kubernetes Core Mechanics
## kube-apiserver & Storage Pipeline

The API server is a stateless HTTP server built around an aggregated filter chain.

```
Request 
  │
  ▼
[ HTTP Handler Chain ]
  ├── Authentication (X.509, OIDC, Webhook)
  ├── Authorization (RBAC, ABAC, Webhook)
  ├── Mutating Admission Webhooks
  ├── OpenAPI / Schema Validation
  └── Validating Admission Webhooks
  │
  ▼
[ Internal Scheme Conversion ]
  └── Unversioned Internal Object Construction
  │
  ▼
[ Storage Layer Engine ]
  ├── Optimistic Concurrency Control Check (resourceVersion validation)
  └── Protobuf / JSON Serialization
  │
  ▼
[ etcd Backend ] (/registry/<group>/<resource>/<namespace>/<name>)
```

**Internal Object Representation***: API objects are decoded from external API versions (e.g., v1, v1beta1) into a single, unversioned internal type before processing, and then serialized back into the requested API version or Protobuf for etcd storage.

**Optimistic Concurrency Control (OCC)**: Updates rely on etcd's 64-bit monotonically increasing mod_revision. When writing, if the target object's resourceVersion doesn't match the current mod_revision in etcd, the write is rejected with a 409 Conflict.

### Why Unversioned Internal Object Conversion is Required?

In Kubernetes, API groups evolve across multiple versions over time (v1alpha1 -> v1beta1 -> v1). Clients might send requests using different API versions simultaneously.

Without an internal unversioned type, every component in kube-apiserver (admission plugins, validation logic, defaulting functions, storage encoders) would need `N x M` conversion functions to handle conversions between every possible pair of external versions.

```
Without Internal Version (N x M Complexity)
  v1alpha1  <─────────>  v1beta1
     ▲                      ▲
     │                      │
     └─────────> v1 <───────┘

With Hub-and-Spoke Internal Version (Linear O(N) Complexity)
  v1alpha1  ──┐          ┌──> v1alpha1
  v1beta1   ──┼─> [ Hub ] ──┼──> v1beta1
  v1        ──┘          └──> v1
```

**The "Hub-and-Spoke" Pattern**
kube-apiserver uses an internal unversioned struct as a Hub:
* Decode: Incoming request (v1beta1) -> Converted to Internal Version (Hub).
* Process: All internal business logic (Validating/Mutating webhooks, field defaulting, schema validation) operates strictly on the Internal Version.
* Encode: Internal Version -> Converted to the Storage Version (e.g., v1) before writing to etcd.This decouples your business logic from external API changes. When a new API version is added, developers only write two conversion functions: v1beta2 -> Internal and Internal -> v1beta2.

When kube-apiserver writes an object to etcd, it serializes the object using the Storage Version configured for that resource (for example, apps/v1).

However, before writing those bytes into etcd at key /registry/deployments/default/my-app, kube-apiserver prepends a magic header / envelope identifier to the byte stream:

```
┌────────────────────────────────┬────────────────────────────────────────────────────────┐
│ Magic Header Prefix            │ Serialized Protobuf/JSON Payload                       │
│ (e.g., "k8s\x00")              │                                                        │
├────────────────────────────────┼────────────────────────────────────────────────────────┤
│ Indicates format/versioning    │ Contains ObjectMeta + Spec + Status                    │
│ protocol used by storage engine│ AND explicit TypeMeta (APIVersion: "apps/v1", Kind)    │
└────────────────────────────────┴────────────────────────────────────────────────────────┘
```

When written to disk in etcd, the payload for apps/v1 Deployment includes its TypeMeta:

```json
{
  "kind": "Deployment",
  "apiVersion": "apps/v1",
  "metadata": { "name": "my-app", "namespace": "default" },
  "spec": { ... }
}
```
Even when stored using Protobuf (the default storage format for performance), the Protobuf schema for the external object explicitly encodes the apiVersion and kind fields into the binary payload.

#### Read Pipeline (etcd -> Client)
```GET /apis/apps/v1beta1/namespaces/default/deployments/my-app:```
1. Client requests: apps/v1beta1

2. kube-apiserver fetches raw bytes from etcd:
   Key: /registry/deployments/default/my-app
   
3. Storage Decoder reads payload header & TypeMeta:
   "Stored APIVersion is apps/v1, Kind is Deployment"
   
4. Storage Decoder deserializes raw bytes directly into:
   Internal Struct (pkg/apis/apps.Deployment)
   
5. API Server converts Internal Struct into requested version:
   Internal (pkg/apis/apps.Deployment)  ───►  External (apps/v1beta1.Deployment)
   
6. API Server serializes apps/v1beta1 struct to JSON & returns to client.

#### The StorageVersionHash Safety Net
If the preferred storage version for a cluster changes over time? (e.g., when a cluster is upgraded from Kubernetes 1.15 to 1.16, shifting storage from apps/v1beta1 to apps/v1).

Because etcd might contain old objects written as apps/v1beta1 alongside new objects written as apps/v1, kube-apiserver does not assume all objects in etcd are stored in the current preferred version.

Instead, the decoding engine operates dynamically:
```go
// Simplified conceptual code inside client-go / apiserver storage decoder
func (s *Store) Get(ctx context.Context, key string) (runtime.Object, error) {
    rawBytes := etcdClient.Get(key)
    
    // 1. Read the GVK header embedded in the raw bytes
    actualGVK := ExtractGVKFromPayload(rawBytes) 
    // -> Returns: gvk.GroupVersionKind{Group: "apps", Version: "v1", Kind: "Deployment"}

    // 2. Lookup the registered Go struct for this exact stored version
    versionedObj := Scheme.New(actualGVK) 

    // 3. Unmarshal raw bytes into that specific versioned struct
    s.Codec.Decode(rawBytes, versionedObj)

    // 4. Convert directly to the Internal Unversioned Type
    internalObj := Scheme.ConvertToInternal(versionedObj)

    return internalObj, nil
}
```


## The Watch Pipeline & controller-runtime Architecture

Operators don't polling-query the API server; they rely on level-triggered reconciliation driven by the client-go Informer architecture.

```
kube-apiserver (etcd watch)
      │
      │ HTTP/2 Streaming (Chunked Transfer / gRPC)
      ▼
   Reflector
      │  Populates
      ▼
  DeltaFIFO Queue
      │  Dequeues
      ▼
  Indexer / Local Store (In-Memory Cache)
      │
      ├──────────────────────┐
      ▼                      ▼
  WorkQueue              Reconciler (Reconcile Loop)
  (Keys only: ns/name)   (Reads ONLY from Indexer cache, writes to APIServer)
  ```

**Level-Triggered vs. Edge-Triggered**: The system reconciles toward current desired state, not historical delta events. If 5 update events occur while the controller is offline, the reconciler processes only the final state once reconnected.

**WorkQueue Deduplication**: The client-go rate-limiting WorkQueue deduplicates keys (namespace/name). If an item is added to the queue multiple times before processing begins, it is coalesced into a single key.

## Node Execution: Kubelet, CRI, CNI, CSI

kubelet converts declarative spec definitions into concrete Linux primitives (namespaces, cgroups, eBPF/iptables).

**SyncLoop Drivers**:

Pod Lifecycle Event Generator (PLEG): Periodically polls the container runtime (CRI) to inspect actual container states, compares them with the desired state in kubelet memory, and generates internal PodLifecycleEvent messages.

**Interface Contracts**:

CRI (Container Runtime Interface): gRPC contract (RunPodSandbox, CreateContainer, StartContainer).

CNI (Container Network Interface): Executable plugin contract invoked during sandbox creation (ADD, DEL, CHECK).

CSI (Container Storage Interface): Out-of-tree RPC execution managed by sidecars (external-provisioner, external-attacher, node-driver-registrar) interfacing with kubelet local sockets.

### CRI (Container Runtime Interface)
What it is: A set of gRPC services (RuntimeService and ImageService) exposed over a Unix domain socket on the node.

How it works: kubelet acts as the gRPC client; containerd or CRI-O acts as the server.

The Flow:

1. kubelet calls RunPodSandbox() to set up IPC, Network, and UTS Linux namespaces.

2. kubelet calls CreateContainer() and StartContainer() to launch application containers inside that sandbox via low-level OCI runtimes like runc or crun.

### CNI (Container Network Interface)

What it is: Unlike CRI (which is a long-running gRPC daemon), CNI was historically an executable binary execution contract (though modern CNIs use local daemons for state management).

How it works:

When CRI creates the Pod sandbox, CRI (not kubelet directly!) executes the CNI binary on the host passing environment variables (CNI_COMMAND=ADD, CNI_CONTAINERID=..., CNI_NETNS=...) and JSON configuration via standard input stdin.

The CNI plugin (e.g., Calico, Cilium) creates network interfaces (e.g., veth pairs), attaches one end inside the pod's network namespace, configures the IP address, and programs host routes or eBPF programs.

### CSI (Container Storage Interface)

What it is: An out-of-tree RPC specification splitting storage operations between Cluster-Wide Controllers and Per-Node Agents.

How it works:

Controller Operations (Attach/Detach): Managed by running out-of-tree CSI sidecars in the control plane. They call cloud APIs (e.g., AWS EC2 API) to attach a storage volume to a virtual machine host.

Node Operations (Mount/Format): Handled by a CSI Node DaemonSet on the target worker node. kubelet communicates with the CSI node plugin over a local Unix socket to format the attached block device (NodeStageVolume) and bind-mount it into the container's filesystem path (NodePublishVolume).

## Pod Lifecycle Event Generator(PLEG)
kubelet needs to know if container states match expectations. Instead of setting up individual polling threads for hundreds of containers, kubelet uses PLEG:
```
[ PLEG Loop (Runs every 1s) ]
        │
        ▼
1. Query CRI: Relist all Pods & Containers (`RuntimeService.ListPodSandbox`, `ListContainers`)
        │
        ▼
2. Compare current CRI states with previous PLEG cache state
        │
        ▼
3. Produce `PodLifecycleEvent` (e.g., ContainerStarted, ContainerDied)
        │
        ▼
4. Push events to `kubelet` main SyncLoop channel
```
If PLEG's iteration takes longer than the threshold (default: 3 minutes), kubelet posts PLEG is not ready.

This happens when:

* CRI Hangs: containerd or the underlying storage driver (e.g., overlay2, NFS) hangs on I/O operations when ListContainers() or ContainerStatus() is called.

* CPU Throttling: kubelet itself is CPU-throttled by system cgroups, preventing the PLEG loop from completing within the health-check deadline.


