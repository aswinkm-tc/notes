# Flow Schema
FlowSchema defines the schema of a group of flows. Note that a flow is made up of a set of inbound API requests with similar attributes and is identified by a pair of strings: the name of the FlowSchema and a "flow distinguisher".

> apiVersion: flowcontrol.apiserver.k8s.io/v1

FlowSchema is the entry point for API Priority and Fairness (APF)—the mechanism that prevents the kube-apiserver from crashing when flooded with HTTP requests.

When an HTTP request (like a kubectl get pods or a kubelet status update) hits the kube-apiserver, it passes through three stages:
```
[ Incoming HTTP Request ]
          │
          ▼
   1. FlowSchema  ────────► Matches request attributes (Who? What? Where?)
          │                 and assigns it to a Priority Level.
          ▼
2. PriorityLevel  ────────► Controls concurrency (How many requests can run at once?)
   Configuration            and provides queueing buffer if full.
          │
          ▼
   3. APIServer  ─────────► Executes the request against etcd or cache.
```

A FlowSchema defines matching rules to classify incoming HTTP requests. Every request is evaluated against all active FlowSchemas in order of their matchingPrecedence (lower numbers = higher priority evaluation).

When a FlowSchema matches a request, it does two things:

1. Assigns a Priority Level: Tells the API server which pool of workers/concurrency seats will process this request (e.g., workload-high, system, exempt, leader-election).

2. Assigns a Flow Distinguisher: Assigns a "hash key" (usually based on the requesting User or the target Namespace). This ensures that if ten users are in the same FlowSchema queue, a single noisy user cannot hog the entire queue—APF uses fair queuing (shuffle sharding) across flow distinguishers.

## Structure
```yaml
apiVersion: flowcontrol.apiserver.k8s.io/v1
kind: FlowSchema
metadata:
  name: platform-operators-schema
spec:
  # Lower number = evaluated earlier
  matchingPrecedence: 500 
  
  # Which Priority Level Configuration manages the execution seats/queues for this schema?
  priorityLevelConfiguration:
    name: workload-high

  # How do we group flows inside this schema to ensure fairness? (By Namespace or By User)
  distinguisherMethod:
    type: ByUser # ByUser and ByNamespace are possible

  # Rules to match incoming HTTP calls
  rules:
  - subjects:
    - kind: ServiceAccount
      serviceAccount:
        name: my-custom-operator
        namespace: platform-system
    resourceRules:
    - apiGroups: ["*"]
      resources: ["*"]
      verbs: ["get", "list", "watch", "patch"]
```

## Defaults
Kubernetes comes pre-configured with several built-in FlowSchemas.

| FlowSchema Name | Priority Level | What it Matches | Why it Exists |
|-|-|-|-|
| exempt | exempt | system:masters group (admin requests). | Never throttled. Ensures cluster admins can always log in and fix a broken cluster. |
| system-nodes | system | Requests from system:nodes (Kubelets updating leases/status). | Protects node heartbeats so nodes aren't mistakenly marked NotReady. |
| kube-controller-manager | workload-high | Built-in controllers (deployment-controller, pv-controller, etc.). | Gives core control loops priority over user applications. |
| service-accounts | workload-low | In-cluster pod requests (e.g., microservices talking to the API). | Prevents app pods from overwhelming the control plane. |
| catch-all | catch-all | Anything that didn't match an earlier schema. | Fallback safety net with low concurrency and small queues. |

## Gotchas
* FlowSchema Precedence Matters: If you write a custom FlowSchema with matchingPrecedence: 9000, a default schema with matchingPrecedence: 1000 might catch your traffic first. Custom schemas should usually have a lower precedence number (e.g., 400-800) to override defaults.
* Shuffle Sharding & Fairness: The distinguisherMethod (e.g., ByUser) uses shuffle sharding under the hood. If 1,000 pods use the same ServiceAccount, APF assigns them across multiple virtual queues. Even if several queues fill up and drop requests, other users/pods assigned to different queue shards continue operating without dropped packets.
* HTTP Webhook Interceptors Bypass APF Queues: If a request matches a FlowSchema, gets an execution seat, and then hangs waiting for a slow Validating/Mutating Webhook, that request holds its APF concurrency seat for the entire duration of the webhook call. This is why a slow webhook can exhaust an entire Priority Level's seats even if APF is working correctly!

# priorityLevelConfiguration
While a FlowSchema acts as the traffic classifier (matching requests and deciding who gets placed into which queue), a PriorityLevelConfiguration (PLC) acts as the resource allocator and queue engine inside Kubernetes API Priority and Fairness (APF).

The PLC determines how many concurrent requests from a given FlowSchema are allowed to execute at the same time and what happens when that capacity is exceeded.

When a FlowSchema matches an incoming request, it hands that request off to its assigned PriorityLevelConfiguration. The PLC manages a dedicated pool of execution seats and a set of fair queues.

```
Incoming Request (e.g., Pod Creation)
         │
         ▼
[ FlowSchema: platform-operators ]
         │
         ▼
[ PriorityLevelConfiguration: workload-high ]
         │
         ├──► Seat Available? ──► [ EXECUTE IN ETCD / API SERVER ]
         │
         └──► Seats Full?     ──► [ PLACE IN SHUFFLE-SHARDED QUEUE ]
                                          │
                                          ├──► Timeout/Queue Full? ──► Return 429 Too Many Requests
                                          └──► Seat Opens?       ──► Execute Request
```
Every PLC divides its capacity into two categories:
* Nominal Concurrency Shares: The number of parallel requests this priority level is guaranteed to execute at any single moment.
* Queuing Mechanics: If all nominal seats are occupied, incoming requests are buffered in virtual queues instead of being immediately rejected with an HTTP 429 Too Many Requests.

## Structure
```yaml
apiVersion: flowcontrol.apiserver.k8s.io/v1
kind: PriorityLevelConfiguration
metadata:
  name: workload-high
spec:
  type: Limited  # Can be 'Limited' or 'Exempt'
  limited:
    # 1. Seat Allocation
    nominalConcurrencyShares: 40  # Weight/percentage of total API Server concurrency
    lendablePercent: 20           # Percentage of unused seats this level can lend to others
    borrowingLimitPercent: 10     # Max percentage of extra seats this level can borrow from others

    # 2. Queue Configuration (When seats are full)
    queuing:
      type: Queue
      queues: 64                  # Number of virtual queues (for shuffle sharding)
      handshakeSeats: 1           # Seats reserved for low-latency dispatching
      queueLengthLimit: 50        # Max items per queue before dropping requests (HTTP 429)
```

## Concurrency Shares, Lending, and Borrowing
* Instead of setting hard limits per priority level, APF calculates dynamic concurrency seats based on the API Server’s total limits (--max-requests-inflight and --max-mutating-requests-inflight).

* Nominal Concurrency Shares: A relative weight. If PriorityLevel A has 40 shares and PriorityLevel B has 20 shares, Level A gets twice as many concurrency seats as Level B.

* Lending and Borrowing (Fairness): If PriorityLevel A is completely idle, it can lend up to lendablePercent of its execution seats to PriorityLevel B. If Level A suddenly experiences a burst of traffic, it reclaims its seats, and Level B’s borrowed requests finish while new Level B requests are pushed back to its queues.

## Shuffle Sharding (Fair Queuing)

What happens when 1,000 pods managed by 10 different engineering teams all flood PriorityLevel: workload-high at once? Should Team A's bug crush Team B's deployment?

To prevent this, PLCs use Shuffle Sharding:
* The PLC creates multiple virtual queues (e.g., queues: 64).
* When a request arrives, APF hashes the FlowSchema's Flow Distinguisher (e.g., the requesting Namespace or User) to assign the request to a small subset (a shard) of those 64 queues.
* Even if Team A’s microservice completely floods its assigned queues, Team B's requests hash to a different set of queues, allowing Team B’s traffic to pass through unaffected.

## Limited vs. Exempt Types

A PriorityLevelConfiguration can have one of two types:

* Limited: Managed by seats, queues, lending, and borrowing. Subject to queue timeouts and HTTP 429 throttling.
* Exempt: Bypasses APF concurrency limits completely. Requests are dispatched immediately without queuing. Reserved exclusively for critical administrative traffic (e.g., the system:masters group via the exempt FlowSchema) to ensure cluster admins can always access a failing cluster.

## Default PriorityLevels
Kubernetes ships with default PLCs tuned for cluster stability:
| Priority Level | Type |Shares / Purpose |
|-|-|-|
| exempt | Exempt | Unlimited seats. Used for system:masters super-admin traffic.| 
|node-high / system | Limited | High nominal shares. Used for kubelet leases, node status, and critical cluster heartbeats. |
| leader-election | Limited | High priority, low latency. Dedicated to controller leader election leases to prevent split-brain states. |
| workload-high | Limited | Core platform controllers (deployment-controller, statefulset-controller). |
| workload-low | Limited | In-cluster application workloads and custom user Service Accounts. |
| catch-all | Limited | Lowest priority. Fallback for any unclassified API traffic. |
