# End-to-End AKS Architecture for Java Microservices

## 1. Problem statement

Design an end‑to‑end platform to host a Java‑based microservice on Azure Kubernetes Service (AKS) that is:

- Highly scalable: handle sudden spikes, multi‑region growth, and team‑driven deployments.
- Fault tolerant: survive node, zone, and component failures without user‑visible impact.
- Secured: zero‑trust access, hardened supply chain, least‑privilege identities, and encrypted data in motion and at rest.

Assumptions:

- Stateless Java Spring Boot microservice.
- HTTP/REST + JSON interface.
- Transactional data in Azure SQL Database; optional read‑heavy/event workloads in Cosmos DB, Redis, Event Hubs.
- Enterprise controls: Azure AD, Azure Firewall, Private Link, WAF, Defender for Cloud.

## 2. High-level infrastructure architecture

At a high level, traffic flows from the internet through an Azure edge and hub‑spoke network into a private AKS cluster that talks to private PaaS data stores.

| Layer | Key components | Purpose |
| --- | --- | --- |
| Edge & security | Azure Front Door / Application Gateway WAF, Azure Firewall | Global routing, TLS, WAF, egress control. |
| Network | Hub‑spoke VNets, Azure CNI, network policies | Isolate AKS, control east‑west and north‑south traffic. |
| Compute | AKS cluster, node pools, HPA/Cluster Autoscaler | Run Java pods, scale based on load, isolate system vs workload nodes. |
| Services | Ingress controller, internal services, Config/Secret stores | Routing, service discovery, app configuration. |
| Data | Azure SQL, Cosmos DB, Redis, Storage | Persistence, caching, blob/object storage. |
| Observability & SecOps | Azure Monitor, Log Analytics, Defender for Containers | Metrics, logs, traces, threat detection. |

Narrative topology summary:

- Hub‑spoke VNet: hub contains Azure Firewall; spoke hosts AKS, Application Gateway WAF, and Private Endpoints to data services.
- Private AKS cluster with Azure CNI (e.g., Cilium) for eBPF‑based data plane and network policy.
- Separate node pools: system, ingress/gateway, Java workloads, batch/cron.
- Private endpoints from AKS spoke to Azure SQL, Cosmos DB, Redis, and Storage; no public exposure of data stores.
- GitOps or CI/CD pipelines push images to Azure Container Registry (ACR) and deploy manifests/Helm charts into AKS.

## 3. Application traffic flow

End‑to‑end request path for a client call to the Java microservice:

1. **Client → Edge/WAF**  
   - Client sends HTTPS request to `https://api.company.com/service` terminating at Azure Front Door or Application Gateway WAF with a TLS certificate.  
   - WAF performs OWASP rules, geo/IP filtering, rate limiting, and header validation.

2. **Edge → AKS ingress**  
   - After WAF, traffic is forwarded over HTTPS to an internal Application Gateway (if Front Door used) or directly to the AKS ingress controller (NGINX or AGIC) in the AKS spoke VNet.  
   - TLS is re‑established to the ingress controller; end‑to‑end encryption is preserved.

3. **Ingress → Kubernetes Service**  
   - Ingress routes based on host/path to a Kubernetes Service (ClusterIP) for the Java microservice.  
   - The Service load‑balances across multiple Java pods via the cluster data plane.

4. **Kubernetes → Java Pod**  
   - A Java pod (Spring Boot container) receives the request, executes domain logic, and calls downstream services (cache, DB, other microservices) over mTLS within the cluster.

5. **Java microservice → Data and other services**  
   - Queries relational data via Azure SQL private endpoint and/or reads/writes documents in Cosmos DB.  
   - Uses Redis cache for hot reads, Event Hubs/Service Bus for async flows, and Blob/Storage for large objects.  
   - Outbound internet calls (e.g., external APIs) go through Azure Firewall with FQDN‑based egress rules.

6. **Response path**  
   - Java pod returns response through the Kubernetes Service → ingress controller → WAF → client, preserving correlation IDs and tracing headers.

## 4. Distributed systems design on AKS

### 4.1 Microservice boundaries and deployment

- Domain‑aligned services, e.g., `orders-service`, `payments-service`, `inventory-service`.
- Each service has its own repo, pipeline, and database or schema.
- AKS namespaces per bounded context or per team; separate `platform` namespace for shared tools (ingress, logging, operators).
- Each Java microservice is a container image in ACR, scanned and validated in CI.

### 4.2 Scaling and availability

- Kubernetes Deployments with multiple replicas spread across availability zones where available.
- Horizontal Pod Autoscaler (HPA) based on CPU/memory or custom metrics (p95 latency, queue length).
- Cluster Autoscaler adds/removes nodes in each node pool as demand changes.
- PodDisruptionBudgets preserve minimum replicas during maintenance.
- Rolling updates, canary, or blue‑green deployments to avoid downtime.

### 4.3 Resilience patterns

- Liveness and readiness probes for each Java pod to detect deadlocks and control load‑balancer registration.
- Retry with exponential backoff and circuit breakers (e.g., Resilience4j) for downstream calls.
- Bulkheads: separate node pools and thread pools for latency‑sensitive vs batch workloads.
- For multi‑region, active/active or active/passive AKS clusters with Front Door global routing and failover.

### 4.4 Service discovery and communication

- Internal communication: HTTP/gRPC via Kubernetes Services; DNS names like `service.namespace.svc.cluster.local`.
- Optional API gateway pattern for external clients, aggregating multiple microservice calls and handling auth and response shaping.
- Optional service mesh (e.g., Open Service Mesh, Istio) for mTLS, traffic policies, observability without app code changes.

### 4.5 Configuration, secrets, and identity

- Configuration via ConfigMaps or Azure App Configuration for non‑secret data.
- Secrets stored in Azure Key Vault and mounted/injected using CSI Secret Store driver or workload identity.
- AKS workload identity (OIDC + Azure AD) for pods to access Azure SQL, Storage, Key Vault with short‑lived tokens.
- Azure RBAC + Kubernetes RBAC for least‑privilege cluster and namespace access.

## 5. Security architecture

### 5.1 Network and perimeter security

- Hub‑spoke topology: hub VNet with Azure Firewall; spoke VNet for AKS and PaaS endpoints, peered to hub.
- Private AKS cluster: API server restricted to authorized ranges or private endpoint; worker nodes not directly internet‑exposed.
- Network policies (Cilium/Calico) define allowed pod‑to‑pod and namespace‑to‑namespace traffic; deny‑all by default.
- All egress via Azure Firewall with threat intelligence and FQDN‑based rules.

### 5.2 Supply chain and image security

- Multi‑stage Dockerfiles, slim base images (distroless/Alpine), pinned versions.
- ACR with image scanning and signed images; only trusted registries allowed.
- Admission controllers/Azure Policy enforce: no privileged containers, required resource limits, pod security standards.

### 5.3 AuthN/AuthZ and data protection

- External clients: OAuth2/OIDC (Azure AD or other IdP); tokens validated at API gateway/ingress or in Java code.
- Internal service‑to‑service auth: mTLS or signed JWTs with scoped roles.
- Data at rest encrypted in Azure SQL, Cosmos DB, Storage, and managed disks; optionally CMKs.
- TLS 1.2+ everywhere; certs managed via Key Vault and cert‑manager/AGIC.

## 6. Data layer and consistency

### 6.1 Polyglot persistence

- Azure SQL Database for transactional data (zone‑redundant, auto‑failover groups, read replicas).
- Cosmos DB for document/JSON and globally distributed workloads with tunable consistency.
- Azure Cache for Redis for hot keys and expensive queries (cache‑aside pattern).
- Azure Service Bus for reliable messaging; Event Hubs for streaming workloads.

### 6.2 Ownership and schema isolation

- Prefer database‑per‑service or schema‑per‑service.
- Each microservice owns its data; cross‑service data sharing via APIs or events, not shared writes.

### 6.3 Transactions and events

- Local transactions within a microservice; eventual consistency between services (saga pattern).
- Outbox pattern: write domain events into an outbox table in the same transaction, then publish asynchronously.
- Idempotent consumers and deduplication to handle retries safely.

### 6.4 Backup, DR, and retention

- Automatic backups and point‑in‑time restore for SQL and Cosmos DB; regular restore drills in non‑prod.
- Geo‑redundant storage for blobs and logs; RPO/RTO defined per service and validated via DR tests.
- Data retention policies and purging for logs and PII to control cost and meet compliance.

## 7. DevOps and observability

- CI/CD: pipelines build/test/scan Java images, push to ACR, deploy via Helm/Kustomize or GitOps; policy checks in the pipeline.
- Metrics: Azure Monitor and Container Insights for node/pod metrics; HPA uses custom metrics where needed.
- Logging: centralized logs (app, ingress, audit, firewall) in Log Analytics; KQL dashboards and alerts on SLOs.
- Tracing: OpenTelemetry agents in Java services, with traces in Application Insights for end‑to‑end request visibility.



# CI/CD

- [CI/CD and GitOps with Argo CD](./ci-cd-argocd.md)
