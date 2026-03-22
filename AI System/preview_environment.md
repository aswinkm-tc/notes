# Preview Environments
## Problem Statement
Allow engineers or stakeholders to spin up a fully functional RAG environment in < 3 minutes, using a sample vector dataset, without touching production systems.

## The Strategy: "Shift-Left" Infrastructure
To achieve < 3 minutes, you must follow these three pillars:
*Logical Isolation (Namespaces)*
* Don't create new clusters or VPCs. Deploy the PR into a dedicated Namespace in a "Preview" K8s cluster.
* Benefits:
    * Avoids creating new clusters or VPCs → saves provisioning time.
    * Provides isolated compute, networking, and secrets per preview.
    * Easy cleanup once the preview is done.
* Implementation Insight:
    * Use Resource Quotas & Network Policies per namespace.
    * Shared underlying infrastructure ensures fast startup and low cost.
*Virtual Databases* 
* Don't "restore" a 5GB Vector DB from a snapshot. 
* Use a Thin Clone or a specialized service like Neon (for Postgres) or Weaviate/Pinecone (with filtered namespaces) that allows "instant" branching of data.
* Solution: Use thin clones or branchable data services.
* Postgres/Neon: Instant branching of DB snapshots.
* Vector DBs (Weaviate, Pinecone): Filtered namespaces or temporary clones.
* Benefit:
    * The preview environment sees a full dataset immediately.
    * No long snapshot restores → achieves the sub-3-minute target.
**Pre-Warmed Images**
* Use a Local Container Registry inside the K8s cluster to avoid pulling heavy LLM/App images over the public internet.

# Architecture
```
+--------------------+       +----------------------+
| Preview Namespace  |       | Local Container      |
| (K8s Namespace)    | <---> | Registry (Pre-Warmed)|
+--------------------+       +----------------------+
        |
        v
+--------------------+
| Virtual Databases  |
| (Thin Clone /      |
| Filtered Namespaces)|
+--------------------+
        |
        v
+--------------------+
| RAG Application    |
| LLM + Vector Search|
+--------------------+
```
## Key Principles:
* All preview environments are ephemeral, isolated, and resource-efficient.
* No extra infra beyond what’s already in the preview cluster.
* Users can test RAG functionality with sample vector datasets instantly.

## User Experience
* Developer opens a PR → triggers a preview environment.
* Preview spins up in < 3 minutes:
    * Namespace deployed
    * DB thin clone ready
    * LLM containers running
* Interactive RAG testing starts:
    * Query sample vector dataset
    * See results instantly
* Preview destroyed after PR merge or expiration.

# Pre-Warmed Images with a DaemonSet
1. Why a DaemonSet Makes Sense
**Goal**: Avoid the time penalty of pulling large LLM or RAG application images on-demand.
**DaemonSet Behavior**:
* Runs one pod per node in the cluster.
* The pod’s sole job: docker pull (or crictl pull) of all necessary images.
* Images are now cached in the node’s container runtime.
**Result**: Any new preview pod that schedules on that node can start almost instantly, since the image is already local.

Implementation Pattern
Create a DaemonSet called image-prepuller.
Spec:
* initContainers or containers that pull LLM/RAG images.
> Optionally keep pods running or just exit successfully after pull.
Trigger:
* Cluster startup
* Node addition (DaemonSet automatically runs on new nodes)
Pseudo-Flow:
```
Node boots → K8s schedules DaemonSet → Pod pulls all required images → Pod exi
```

## Additional Optimizations
* Version Pinning: Always pull specific image tags for stability.
* Parallel Pulling: Pull multiple images concurrently to reduce startup time.
* Local Registry Integration: Combine with a local container registry inside the cluster so that pulls are fast and network-independent.
* Optional Health Check: DaemonSet pod can report which images are ready on each node.

