# Ingestion & Agentic Pipeline
**Goal**: < 30s latency for data freshness; reliable multi-step agent tasks.
## Ingestion
Use Kafka/SQS with Tenant-Based Partitioning. This prevents one large customer from delaying the data of another.
## Execution

Use Durable Execution (Temporal/Step Functions).
* Heartbeats: Workers must "check-in" while doing long LLM tasks. If a user cancels, the heartbeat check kills the task immediately to save API costs.
* Async UX: Return a 202 Accepted immediately. Use Server-Sent Events (SSE) to stream the "Agent's Monologue" (e.g., "Searching Slack...") back to the UI.

## Model Gateway (10,000 RPS)
**Goal**: High-throughput, privacy-first LLM orchestration.
* Zero-Day Retention (ZDR): Use a Stateless Proxy (Go/Rust) that never writes to disk. Inject ZDR headers for providers and scrub PII in memory using a DLP Interceptor (e.g., Presidio).
* Adaptive Hedging: Don't just set a timeout. If the Primary API (OpenAI) takes > P50 latency, fire a parallel request to the Backup API (Anthropic). Take the winner.
* Circuit Breakers: If a provider has a > 5% error rate or extreme latency, "trip" the circuit and fail-fast to save system resources.
## Semantic Cache & "Cold Start"
Goal: Reduce LLM costs and prevent system collapse during traffic spikes.
* Semantic Search: Use Redis Vector Similarity Search (VSS). Cache by the meaning of the query, not the exact string.
* The Thundering Herd: Use Atomic Locking (SETNX) and Request Collapsing.
    * If 1,000 requests hit an empty cache, Request #1 grabs the lock.
    * Requests 2-1,000 wait via Redis Pub/Sub.
* Stale-While-Revalidate: When data is updated (e.g., a Slack message edit), mark the cache as "Stale." Serve the stale data to 99% of users while 1% refreshes the cache in the background.
## Preview Environment Engine
Goal: 150 engineers, each getting a full stack in < 3 minutes.
* Logical Branching: Do not spin up new DB instances. Use Kubernetes Namespaces and a Shared Preview Cluster.
* Vertex AI Branching: Use Restrict Labels (Metadata Filtering).
    * Read Path: Filter = (pr_id OR golden).
    * Write/Delete Path: Filter = (pr_id).
* Egress Gateway: Place a Transparent Proxy between the PR namespace and GCP. It automatically identifies the namespace and injects the pr_id filter into the JSON body so devs can't "bypass" isolation.
* Scale-to-Zero: Implement a Janitor Controller that deletes namespaces on PR merge/close and sleeps environments with no traffic to save cloud OpEx.

## Interview Tips:
1. Trade-offs are King: Whenever you propose a solution, mention one drawback (e.g., "The Egress Gateway adds ~5ms of latency, but it’s worth it for the security and central policy enforcement.").
2. State is the Enemy: Always try to make your API and Gateway layers stateless and push state into specialized distributed stores (Redis, Temporal, etc.).
3. Observability: Mention that you'd implement Distributed Tracing (OpenTelemetry) across all these layers so you can see exactly where a 2-minute agent task is hanging.

Concept	Type	Purpose	Key Question for You
SOC 2	Operational	Proving your processes are secure.	"How do we track who accessed this GPU node?"
GDPR	Legal/Civic	Protecting user rights and privacy.	"Where is this data physically stored?"
AES-256	Technical	Securing the bits on the disk.	"Is our storage encrypted by default?"

# SOC 2 (System and Organization Controls)
SOC 2 is a voluntary compliance standard for service organizations, developed by the AICPA. It specifies how organizations should manage customer data based on five “Trust Services Criteria”: Security, Availability, Processing Integrity, Confidentiality, and Privacy.
* The "Staff" Reality: You don't just "get" SOC 2; you maintain it through evidence.
* Infrastructure Impact:
    * Audit Logging: Every access to a production database must be logged (e.g., GCP Cloud Logging).
    * Change Management: You cannot push code to production without a peer review and a green CI/CD pipeline.
    * Infrastructure as Code (IaC): SOC 2 auditors love Terraform because it provides a clear history of who changed what in the infrastructure.


# GDPR (General Data Protection Regulation)
GDPR is a legal framework that sets guidelines for the collection and processing of personal information from individuals who live in the European Union (EU). Since Sana is based in Stockholm, this is their "home turf" regulation.
* The "Staff" Reality: GDPR is about Data Sovereignty and the Right to be Forgotten.
* Infrastructure Impact:
    * Data Residency: You may need to design clusters that keep German customer data strictly within the GCP europe-west3 (Frankfurt) region.
    * The "Right to Erasure": If a user deletes their account, your infrastructure must ensure their data is wiped from primary DBs, caches, and backups within a specific timeframe.
    * PII Masking: Designing systems so that developers or AI researchers cannot see PII (Personally Identifiable Information) in logs or staging environments.

AES-256 (Advanced Encryption Standard)
AES-256 is a symmetric block cipher chosen by the U.S. government to protect classified information. The "256" refers to the key length—it is currently considered mathematically "unbreakable" by brute force.
* Infrastructure Impact:
    * Encryption at Rest: You’ll likely use GCP’s Cloud KMS (Key Management Service) to manage the keys that encrypt your Postgres disks and S3/GCS buckets.
    * Envelope Encryption: A common Staff-level design pattern where you encrypt the data with a "Data Encryption Key" (DEK) and then encrypt that key with a "Key Encryption Key" (KEK).
    * Performance: Modern CPUs (Intel/AMD) have hardware acceleration for AES (AES-NI), so the performance overhead is negligible, making it a no-brainer for every layer of the stack.


