# Leases
In Kubernetes, coordination.k8s.io is an API group dedicated to system coordination. Its primary resource is the Lease, which represents mutual exclusion or temporary ownership of a shared resource.

In distributed systems, multiple components often run concurrently. Leases allow these components to coordinate activity, ensure high availability, and prevent split-brain scenarios or redundant work.

## Structure
A Lease object in coordination.k8s.io/v1 contains a spec that tracks who holds the lock and for how long:  
```yaml
apiVersion: coordination.k8s.io/v1
kind: Lease
metadata:
  name: my-component-lock
  namespace: default
spec:
  holderIdentity: "controller-pod-abc123" # Who currently holds the lease
  leaseDurationSeconds: 15              # How long the lease is valid
  acquireTime: "2026-06-06T10:00:00Z"   # When it was acquired
  renewTime: "2026-06-06T10:05:20Z"     # When it was last refreshed
  leaseTransitions: 2                   # Number of times leadership changed
```

**holderIdentity:** Identifies the current owner of the lease (e.g., a specific pod name or hostname).
**leaseDurationSeconds:** The duration that non-holding candidates must wait before they can force-acquire an expired lease.
**renewTime:** The timestamp of when the current holder last updated (renewed) the lease. If this gets too old, another instance can step in and take over.
**acquireTime:** The timestamp of when the current holder initially acquired the lease.
**leaseTransitions:** A counter tracking how many times the lease has changed hands.
