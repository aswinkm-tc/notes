# Security & Zero Trust Pipeline
Security Mindset: "Defense in Depth." Assume your container has a remote code execution (RCE) vulnerability. Security controls must prevent container escape to the node host, lateral movement across namespaces, and exfiltration of cloud resources.

## Identity & Privilege Escalation (IRSA vs. Workload Identity)
Never store long-lived cloud credentials (AWS Access Keys, GCP Service Account JSON keys) inside Kubernetes secrets.
```
      Pod Service Account Token (JWT)
                      │
                      ▼
   kube-apiserver OIDC Provider Endpoint
                      │
                      ▼
Cloud IAM (AWS STS / GCP IAM / Azure Entra)
                      │
                      ▼
      Short-Lived Ephemeral Cloud Token
```

### The Mechanism:
1. The Kubelet projects a signed service account JWT (boundServiceAccountToken) into the pod volume.
2. The application SDK presents this JWT to the Cloud IAM STS (Security Token Service).
3. Cloud IAM verifies the JWT against the cluster's public OIDC discovery endpoint (/.well-known/openid-configuration).
4. STS exchanges the JWT for a short-lived (e.g., 1 hour) cloud IAM role token.

**Implementations:** EKS Pod Identity / IRSA (AWS), Workload Identity (GCP), Workload ID (Azure).

## Pod Security Standards (PSS) & Admission Controllers
Legacy PodSecurityPolicy (PSP) was deprecated and removed. Modern security governance relies on a two-tiered model:

### Built-in Pod Security Admission (PSA): Enforces Pod Security Standards via namespace labels:
**privileged:** Unrestricted execution.

**baseline:** Prevents known privilege escalations (default host path blocks, host network blocks).

**restricted:** Hardened runtime. Forces runAsNonRoot: true, drops ALL capabilities, requires readOnlyRootFilesystem: true.

### Policy-as-Code Engine (Kyverno / OPA Gatekeeper):

Uses Validating & Mutating Webhooks.

**Mutating Webhooks:** Automatically inject sidecars, default resource requests, or security contexts.

**Validating Webhooks:** Block manifests at kubectl apply time if they contain prohibited image registries, missing team tags, or :latest container tags.

### Securing Network East-West Traffic (Zero-Trust NetworkPolicies)
By default, all pods in a Kubernetes cluster can talk to all other pods across all namespaces.
```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-all
  namespace: production
spec:
  podSelector: {} # Applies to ALL pods in namespace
  policyTypes:
  - Ingress
  - Egress
```

**Best Practice:** Apply a Default Deny All Ingress/Egress policy in every production namespace, then explicitly whitelist required communication paths:

Whitelist ingress from Ingress Controllers / Gateway Proxies.

Whitelist egress to CoreDNS (kube-system port 53 UDP/TCP) and specific backend dependencies.

