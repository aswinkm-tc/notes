# Pod Networking Lifecycle
```
Host Network Namespace]                          [Pod Network Namespace]
  +-------------------------------------+          +----------------------+
  |                                     |  veth    |                      |
  |  eth0 / Encap Engine  <--->  vethX <===========> eth0                 |
  |                                     |  pair    | (10.244.1.5)         |
  +-------------------------------------+          +----------------------+
```

1. Namespace Isolation: CRI creates a new, isolated Linux Network Namespace (/var/run/netns/...) for the Pod.
2. Veth Pair Provisioning: The CNI plugin creates a virtual ethernet pair (veth pair)—think of it as a virtual patch cable with two ends.
3. Plugging the Cable:
* One end (eth0) is moved into the Pod's network namespace and assigned an IP address, netmask, and default route pointing to the host end.
* The other end (e.g., veth1234abcd) stays in the host network namespace.
* Host Attachment: The CNI attaches the host-end veth to either a Linux bridge (cni0), an eBPF program, or the host routing table, depending on the CNI implementation.
* Route Registration: The CNI binary updates the local node routing table or BGP agent so traffic destined for the Pod's IP knows which interface to traverse.

## CNI Binary vs. CNI Daemon Set
| Component	| What it is | Where it lives | Responsibility |
|-|-|-|-|
| CNI Plugin Binary	| Standardized executable (/opt/cni/bin/) | Host disk (/opt/cni/bin/) | Invoked synchronously by CRI (via CNI spec inputs via STDIN) on Pod creation/deletion (ADD/DEL). Executed once, then exits. |
| CNI Config File | JSON configuration file | Host disk (/etc/cni/net.d/) | Read by CRI to determine which CNI plugin binary to invoke and with what parameters. |
| CNI Daemon (e.g., cilium-agent, calico-node) | Long-running process (DaemonSet) | Runs in the cluster as a Pod | Manages IP Address Management (IPAM) pools, updates BGP routes/eBPF maps/iptables rules across nodes, syncs cluster state asynchronously. |

> The CRI calls the CNI Binary on disk, which talks to the local CNI Daemon socket to reserve an IP and establish network routes.

## VXLAN vs BGP
### VXLAN (Overlay Mode)

VXLAN wraps original Layer 2 Pod frames inside a standard Layer 4 UDP packet (UDP port 4789).

```
+---------------------------------------------------------------------------------------+
| Outer MAC | Outer IP (Node A -> Node B) | Outer UDP (4789) | VXLAN Header | Inner L2/L3 Frame |
+---------------------------------------------------------------------------------------+
|<------------------------- Outer VXLAN Encap -------------------> | <-- Original Pod Payload --> |
```

1. Egress (Node A): Pod A sends a packet to Pod B IP. The kernel realizes Pod B is remote. It sends the frame to the VTEP (VXLAN Tunnel Endpoint, e.g., vxlan.calico or cilium_vxlan).
2. Encapsulation: The VTEP wraps the entire inner frame inside an Outer IP/UDP packet.
* Outer Source IP: Node A IP
* Outer Dest IP: Node B IP
3. Transit: Physical network routers only see traffic between Node A and Node B. They know nothing about Pod IPs.
4. Ingress (Node B): Node B's VTEP decapsulates the UDP packet, strips the outer headers, inspects the inner destination IP (Pod B), and routes it across the veth pair into Pod B's network namespace.

### BGP / Direct Routing (Underlay Mode)
There is no encapsulation or extra headers.
1. Routing: Each node runs a BGP daemon (like calico-node or bird) that peers with other nodes or Top-of-Rack (ToR) switches. Nodes advertise their local Pod CIDR blocks (Node A advertises 10.244.1.0/24, Node B advertises 10.244.2.0/24).
2. Egress (Node A): When Pod A sends a packet to Pod B, Node A's standard Linux routing table says: "For 10.244.2.0/24, send to Next-Hop: Node B IP".
3. Transit: The physical switch forwards the raw IP packet directly to Node B.
4. Performance: Zero encapsulation/decapsulation CPU overhead, lowest latency, but requires underlying network/ToR switch support for BGP peering.

### MTU Misconfiguration
Standard Ethernet MTU is 1500 bytes.

VXLAN adds a 50-byte overhead (Outer Ethernet + Outer IP + Outer UDP + VXLAN Header).

If the Pod network interface is left at 1500 bytes instead of being adjusted to 1450 bytes:
* The Trap: Pod A sends a large packet (e.g., 1500 bytes).
* The Encapsulation: Node A adds 50 bytes of VXLAN encapsulation -> total packet size becomes 1550 bytes.
* The Failure Modes:
* * If DF (Don't Fragment) bit is set (common in TCP/TLS): The underlying physical network drops the 1550-byte packet and sends back an ICMP Need to Fragment error. If ICMP is blocked by a firewall, the connection hangs indefinitely (known as a Blackhole Connection).
* * If Fragmentation is allowed: The host network breaks the packet into multiple IP fragments. This severely degrades performance, increases CPU overhead on decapsulation, and often leads to subtle packet drop bugs.


## Kube-Proxy (iptables) vs. eBPF (Cilium)
### Traditional kube-proxy (iptables) Pipeline

When a Pod makes a request to a Service ClusterIP (10.96.0.10:80):
```
[ Pod A ] 
    │ (Traffic to ClusterIP)
    ▼
[ host netfilter / iptables chains ] ───► Sequential O(N) evaluation of 100,000 rules
    │                                     (KUBE-SERVICES -> KUBE-SVC-xxx -> KUBE-SEP-xxx)
    │ (DNAT: Rewrites ClusterIP -> Pod B Real IP)
    ▼
[ Routing Table / CNI ] ──────────────► Forward to Node B
```

The Problem:
* Rule Traversal: At 10,000 Services, netfilter evaluates tens of thousands of rules per packet O(N).
* Rule Sync Latency: When an endpoint changes (a Pod dies or starts), kube-proxy must rewrite the entire iptables rule dump. At scale, updating iptables can lock the kernel and take tens of seconds.

### eBPF Architecture (Cilium kube-proxy-replacement)

eBPF attaches programs directly to kernel hooks (e.g., tc at the network driver layer, or socket hooks at the socket layer).
```
[ Pod A Socket ] 
    │ (sys_connect)
    ▼
[ eBPF cgroup/socket Hook ] ───► BPF Hash Map Lookup O(1) ───► In-kernel socket translation
    │                                                           (Rewrites dest before packet is formed!)
    ▼
[ Fast Path / XDP ] ──────────► Bypasses host TCP/IP stack & netfilter entirely
```
* O(1) Lookups: Service VIP mappings are stored in eBPF Hash Maps. Instead of looping through thousands of rules, eBPF does a direct key-value hash lookup in O(1) constant time.
* Socket-Layer Load Balancing: With eBPF sockops hooks, Cilium can intercept the connect() syscall right at the application socket layer. It rewrites the destination IP before the packet even enters the host TCP/IP network stack, bypassing netfilter entirely!
* Zero-Downtime Atomic Map Updates: When an endpoint changes, Cilium updates a single entry in the eBPF map instantly, without re-evaluating or lock-dumping kernel rules.
