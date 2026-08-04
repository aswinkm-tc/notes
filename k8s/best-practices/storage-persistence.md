# Storage & Persistence (CSI)
In production Kubernetes, persistent storage is all about CSI driver abstraction, volume binding topology, and local vs. distributed storage performance trade-offs.

## The Container Storage Interface (CSI) Architecture
```
K8s Control Plane                   Worker Node
┌────────────────────┐              ┌──────────────────────────┐
│ API Server         │              │ Kubelet                  │
└─────────┬──────────┘              └────────────┬─────────────┘
          │ Watch PVCs                           │ Watch VolumeAttachment
          ▼                                      ▼
┌────────────────────┐  gRPC (Unix Socket) ┌──────────────────┐
│ CSI Controller     ├────────────────────►│ CSI Node Plugin  │
│ (Provision/Attach) │                     │ (Mount/Unmount)  │
└────────────────────┘                     └──────────────────┘
```

**CSI Provisioner (Control Plane):** Watches PersistentVolumeClaim (PVC) creation and invokes the underlying storage API (e.g., AWS EBS CreateVolume, Ceph rbd create) to allocate storage, creating a PersistentVolume (PV) object.

**CSI Attacher (Control Plane):** Attaches the block storage device to the target node instance (e.g., AWS AttachVolume).

**CSI Node Plugin (Worker DaemonSet):** Formats the attached raw block device (e.g., ext4 or xfs) and bind-mounts the directory into the container's root filesystem space (/var/lib/kubelet/pods/...).

## Topology-Aware Volume Binding

A common failure mode in Multi-AZ clusters occurs when a PVC is created in Zone A, but the pod gets scheduled in Zone B (where AWS EBS cannot attach cross-zone).
```
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: ebs-sc
provisioner: ebs.csi.aws.com
volumeBindingMode: WaitForFirstConsumer # CRITICAL FOR MULTI-AZ
allowedTopologies:
- matchLabelExpressions:
  - key: topology.ebs.csi.aws.com/zone
    values:
    - us-east-1a
    - us-east-1b
```

**volumeBindingMode: Immediate (Default/Legacy):** Volume is created immediately upon PVC creation in a random AZ before the Pod is scheduled. Causes pod unschedulable errors if node placement doesn't match volume AZ.

**volumeBindingMode: WaitForFirstConsumer (Production Standard):** Delays PV allocation until the kube-scheduler selects a valid node for the Pod. Guarantees the volume is created in the exact Availability Zone where the pod lands.

## Storage Patterns: Distributed Block (Rook-Ceph/Longhorn) vs. Local PVs
| Requirement | Distributed Block (Rook-Ceph / Longhorn / Cloud Disk) | Local Persistent Volumes (Local PV) |
|-|-|-|
| How it Works | Storage is networked across worker nodes or attached via Cloud SAN. | Directly uses local NVMe/SSD disks attached physically to the node. |
| High Availability | Network-level replication. | If Node A dies, the volume re-attaches to Node B. Single Point of Failure (SPOF). |If the node dies, the data on that local disk is unreachable. |
| IOPS / Latency | Network latency overhead (2-10ms). | Maximum raw disk performance (< 0.1ms sub-millisecond latency). |
| Best Used For | General stateless/stateful apps (PostgreSQL, Redis single-node, web uploads). | Distributed databases with built-in replication (Kafka, Cassandra, Elasticsearch, ClickHouse). |
