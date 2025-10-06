# Platform Contracts

## Table of Contents

- [Executive Summary](#executive-summary)
- [Core Concepts](#core-concepts)
  - [Ports and Adapters Pattern](#ports-and-adapters-pattern)
  - [Contract Types](#contract-types)
  - [Deployment Profiles](#deployment-profiles)
- [Architecture](#architecture)
  - [Contract Catalog](#contract-catalog)
- [Contract Specifications](#contract-specifications)
  - [1. Object Storage Contract](#1-object-storage-contract)
  - [2. Container Registry Contract](#2-container-registry-contract)
  - [3. Secret Management Contract](#3-secret-management-contract)
  - [4. DNS Management Contract](#4-dns-management-contract)
  - [5. Message Queue Contract](#5-message-queue-contract)
  - [6. Cluster Provisioning Contract](#6-cluster-provisioning-contract)
  - [7. Observability Contract](#7-observability-contract)
  - [8. Authentication Contract](#8-authentication-contract)
  - [9. Workload Scaling Contract](#9-workload-scaling-contract)
  - [10. Database Contract](#10-database-contract)
  - [11. Network Contract (with Service Mesh)](#11-network-contract-with-service-mesh)
- [Configuration](#configuration)
  - [Adapter Selection](#adapter-selection)
  - [Contract Verification](#contract-verification)
- [Integration Points](#integration-points)
  - [Application Integration](#application-integration)
  - [Platform Service Integration](#platform-service-integration)
- [Constraints](#constraints)
  - [Hard Requirements](#hard-requirements)
  - [Adapter Constraints](#adapter-constraints)
- [Future Considerations](#future-considerations)
  - [Additional Contracts](#additional-contracts)
  - [Adapter Implementations](#adapter-implementations)

## Executive Summary

This document defines the platform's 11 infrastructure contracts using the ports and adapters architectural pattern. Each contract specifies required capabilities, interface specifications, and behavioral requirements independent of any specific implementation.

The platform provides multiple adapter implementations for each contract, enabling deployment across different infrastructure environments (AWS production, on-premises) without changing application code or platform architecture. Applications interact with stable port interfaces while the platform swaps underlying adapters based on deployment profile.

The 11 contracts cover: Object Storage, Container Registry, Secret Management, DNS Management, Message Queue, Cluster Provisioning, Observability, Authentication, Workload Scaling, Database, and Network (with Service Mesh).

## Core Concepts

### Ports and Adapters Pattern

The platform separates **what** infrastructure is needed (ports) from **how** it's provided (adapters):

- **Port**: A contract defining required capabilities and interface specifications
- **Adapter**: A concrete implementation of a port contract
- **Application**: Interacts only with ports, never directly with adapters

This separation enables infrastructure portability: applications remain unchanged while adapters swap based on deployment environment.

### Contract Types

Contracts are classified by provisioning responsibility:

**Platform-Provided**: The platform deploys and manages the adapter implementation
- Examples: Object Storage (S3/MinIO), Message Queue (NATS), Authentication (Keycloak)
- Platform handles deployment, scaling, upgrades
- Multiple adapter options available per deployment profile

**External Required**: External systems or services that must be provided
- Examples: DNS Provider, Identity Provider (for Keycloak federation)
- Platform integrates via standard protocols
- Environment-specific configuration required

### Deployment Profiles

Adapter implementations are selected based on deployment profile:

**Production/AWS Profile**:
- Managed AWS services where available
- Self-hosted alternatives for platform-specific needs
- IRSA for authentication to AWS services

**On-Premises Profile**:
- Kubernetes-native implementations
- Self-hosted open-source alternatives
- ServiceAccount-based authentication

## Architecture

### Contract Catalog

The platform defines 11 infrastructure contracts:

| Contract                        | Type                  | AWS Adapter                       | On-Premises Adapter                  |
| ------------------------------- | --------------------- | --------------------------------- | ------------------------------------ |
| Object Storage                  | Platform-Provided     | Amazon S3                         | MinIO / S3-compatible                |
| Container Registry              | Platform-Provided     | Amazon ECR                        | Harbor / OCI-compliant registry      |
| Secret Management               | Platform-Provided     | AWS Secrets Manager               | HashiCorp Vault / Kubernetes Secrets |
| DNS Management                  | External Required     | Route53                           | Any ExternalDNS-compatible provider  |
| Message Queue                   | Platform-Provided     | NATS JetStream                    | NATS JetStream                       |
| Cluster Provisioning            | Platform-Provided     | EKS + Crossplane                  | Crossplane with provider             |
| Observability                   | Platform-Provided     | Grafana Cloud                     | In-cluster Prometheus/Loki/Grafana   |
| Authentication                  | Platform-Provided     | Keycloak                          | Keycloak                             |
| Workload Scaling                | Platform-Provided     | KEDA + Karpenter                  | KEDA + cluster autoscaler            |
| **Database**                    | **Platform-Provided** | **Amazon RDS**                    | **CloudNativePG**                    |
| **Network (with Service Mesh)** | **Platform-Provided** | **Envoy Gateway + Istio Ambient** | **Envoy Gateway + Istio Ambient**    |

---

## Contract Specifications

### 1. Object Storage Contract

**Purpose**: Store and retrieve large unstructured data (logs, artifacts, staging files)

**Type**: Platform-Provided

**Required Capabilities**:
- Object storage and retrieval with globally unique keys
- Bucket/prefix-based organization
- Lifecycle policies for automatic cleanup
- Presigned URLs for temporary authenticated access
- Versioning support (optional)

**Interface Specification**:
- S3-compatible API (putObject, getObject, deleteObject, listObjects)
- Authentication via credentials or IAM roles
- HTTPS transport with TLS 1.2+
- Bucket naming and key patterns per S3 conventions

**Behavioral Requirements**:
- Strong consistency for read-after-write
- Durable storage (11 nines or equivalent)
- Scalable throughput (minimum 100 MB/s per client)
- Support for objects up to 5TB

**Adapter Implementations**:

**Amazon S3 (AWS Profile)**:
- Managed object storage service
- IRSA authentication for platform services
- Lifecycle policies for 90-day log retention, 1-day artifact staging cleanup
- Versioning enabled for audit trails
- Server-side encryption (SSE-S3)

**MinIO / S3-Compatible Storage (On-Premises Profile)**:
- Self-hosted S3-compatible object storage
- Options: MinIO, Ceph RGW, local disk arrays, network-attached storage
- ServiceAccount-based authentication or access keys
- Lifecycle management through MinIO policies or equivalent
- Configurable storage backend (local disks, network storage, distributed systems)

---

### 2. Container Registry Contract

**Purpose**: Store and distribute OCI-compliant container images and artifacts

**Type**: Platform-Provided

**Required Capabilities**:
- Push and pull OCI images with digest-based addressing
- Image scanning for vulnerabilities (recommended)
- Immutable tags (recommended)
- Registry authentication and authorization
- Multi-architecture image support

**Interface Specification**:
- OCI Distribution Specification v1.0+
- Standard Docker registry API v2
- Authentication via Docker config or registry credentials
- HTTPS with TLS 1.2+

**Behavioral Requirements**:
- Content-addressable storage by SHA256 digest
- Support for image layers up to 10GB
- Tag immutability to prevent accidental overwrites
- Garbage collection for unreferenced layers

**Adapter Implementations**:

**Amazon ECR (AWS Profile)**:
- Managed container registry
- IRSA authentication for platform workers
- Automatic encryption at rest
- Image scanning with AWS Inspector
- Lifecycle policies for image retention
- CloudWatch audit logging integration

**Harbor / OCI-Compliant Registries (On-Premises Profile)**:
- Self-hosted container registry implementing OCI Distribution Spec
- Options: Harbor, Quay, GitLab Container Registry, Docker Registry
- Robot accounts or ServiceAccount authentication
- Content trust and signing support
- Vulnerability scanning with Trivy or Clair
- Replication policies for high availability
- Multiple storage backends (local, S3-compatible, filesystem)

---

### 3. Secret Management Contract

**Purpose**: Secure storage, rotation, and distribution of sensitive configuration

**Type**: Platform-Provided

**Required Capabilities**:
- Encrypted secret storage with access control
- Hierarchical organization (prefixes/paths)
- Secret rotation support
- Audit logging of access
- Integration with Kubernetes via External Secrets Operator or equivalent

**Interface Specification**:
- Secret retrieval by path/key
- Environment-based prefixing (`{env}/`, `global/`)
- JSON or key-value storage formats
- Secret versioning (optional)

**Behavioral Requirements**:
- Encryption at rest (AES-256 or equivalent)
- Fine-grained access control per path prefix
- Secrets materialized as Kubernetes Secrets
- No secrets in platform configuration files
- Resolution only at point of use (no caching in clear text)

**Adapter Implementations**:

**AWS Secrets Manager (AWS Profile)**:
- Managed secret storage service
- IRSA-based authentication with path prefix scoping
- Automatic rotation for supported secret types
- CloudWatch audit logging
- Hierarchical paths: `{env}/{purpose}/*` and `global/*`
- External Secrets Operator synchronization to Kubernetes Secrets

**HashiCorp Vault (On-Premises Profile)**:
- Self-hosted secret management with dynamic secrets
- Kubernetes authentication with role-based policies
- Versioning and lease management
- Path-based organization matching AWS structure
- External Secrets Operator or Vault Agent injection

**Kubernetes Native Secrets (On-Premises Profile)**:
- Direct secret management using Kubernetes native resources
- RBAC controls access by namespace and secret name
- Suitable for deployments not requiring external secret stores
- Sealed Secrets or SOPS for GitOps encryption
- No automatic rotation (manual process)

---

### 4. DNS Management Contract

**Purpose**: Automatic DNS record lifecycle management for services

**Type**: External Required

**Required Capabilities**:
- DNS record creation, update, and deletion
- Integration with Kubernetes via External DNS operator
- Support for A, AAAA, CNAME, and TXT records
- Zone delegation for environment isolation

**Interface Specification**:
- ExternalDNS-compatible provider API
- Authentication per provider requirements
- DNS zone format: `{environment}.{region}.{domain}`
- TTL configuration per record type

**Behavioral Requirements**:
- Record creation within 60 seconds of service deployment
- Automatic cleanup on service deletion
- No manual DNS record management required
- Environment-isolated DNS zones

**Provider Examples**:

**Route53 (AWS Profile)**:
- AWS managed DNS service
- IRSA or access key authentication
- Delegated zones per environment
- CloudWatch logging for changes

**Any ExternalDNS-Compatible Provider (On-Premises Profile)**:
- Options: Cloudflare, Google Cloud DNS, PowerDNS, CoreDNS, RFC2136
- Provider-specific authentication (API keys, TSIG, etc.)
- Must support ExternalDNS integration
- Zone delegation recommended for environment isolation

---

### 5. Message Queue Contract

**Purpose**: Asynchronous job distribution with work queue semantics

**Type**: Platform-Provided

**Required Capabilities**:
- Message publishing with reply subjects (request-reply pattern)
- Durable message storage with guaranteed delivery
- Consumer groups with load balancing
- Message acknowledgment and redelivery
- Stream-based partitioning for different job types

**Interface Specification**:
- NATS JetStream API (publish, subscribe, acknowledge)
- Subject-based routing: `pipeline.{type}.*` (discovery, tasks, artifacts, releases)
- Durable consumers with pull-based message retrieval
- TLS authentication with per-service credentials

**Behavioral Requirements**:
- At-least-once delivery guarantee
- Message persistence across broker restarts
- Scalable throughput (thousands of messages/second)
- Consumer lag monitoring for autoscaling
- Redelivery on timeout or explicit NACK
- Request-reply timeout configurable per job type

**Adapter Implementation**:

**NATS JetStream (Both Profiles)**:
- Same implementation for AWS and on-premises
- 3-node cluster for high availability
- Persistent storage (EBS volumes on AWS, local/network storage on-premises)
- Streams: pipeline-discovery, pipeline-tasks, pipeline-artifacts, pipeline-releases
- TLS with mutual authentication
- Scales horizontally with cluster size

---

### 6. Cluster Provisioning Contract

**Purpose**: Declarative Kubernetes resource provisioning and lifecycle management

**Type**: Platform-Provided

**Required Capabilities**:
- Composite resource definitions (XRDs) for common patterns
- Environment-aware value resolution via EnvironmentConfigs
- Output publishing for resource dependencies
- Composition functions for transformation logic
- Multi-adapter support per deployment environment

**Interface Specification**:
- Crossplane Composite Resources API
- XRD catalog: Deployment, Stateful, Job, CronJob, Network, Secrets, ConfigMaps, Storage
- Environment configuration via cluster-scoped EnvironmentConfig CRDs
- Reference resolution patterns: `outputs/`, `connections/`, `secrets/`, `configs/`

**Behavioral Requirements**:
- XR reconciliation within 30 seconds
- EnvironmentConfig changes applied on next reconcile
- Composition selection via labels or environment configuration
- Status reporting through XR conditions
- Output freshness based on reconcile interval

**Adapter Implementations**:

**Crossplane with AWS Provider (AWS Profile)**:
- AWS provider for managed resources (RDS, ElastiCache, etc.)
- Compositions using AWS-specific managed resources
- IRSA for Crossplane authentication to AWS
- EKS-optimized compositions

**Crossplane with Helm/Kubernetes Providers (On-Premises Profile)**:
- Helm provider for chart-based deployments
- Kubernetes provider for native resources
- Compositions using Bitnami charts or custom Helm charts
- ServiceAccount-based authentication

---

### 7. Observability Contract

**Purpose**: Unified metrics, logs, and traces collection across all components

**Type**: Platform-Provided

**Required Capabilities**:
- Prometheus-compatible metrics collection
- Log aggregation with metadata enrichment
- Dashboard and alerting interfaces
- Service discovery for automatic scraping
- Long-term retention and querying

**Interface Specification**:
- Grafana Alloy agent for data collection (component-based configuration)
- Prometheus metrics endpoints (`/metrics`)
- Structured logging (JSON) to stdout/stderr
- Standard labels: namespace, pod, container, service

**Behavioral Requirements**:
- Metrics scraped every 15 seconds
- Logs collected in near real-time (< 5 second delay)
- Metadata enrichment (Kubernetes labels, annotations)
- Configurable retention periods
- Alerting on platform health metrics

**Adapter Implementations**:

**Grafana Cloud (AWS Profile)**:
- Managed observability platform
- Grafana Alloy ships metrics via `prometheus.remote_write` to cloud endpoints
- Grafana Alloy ships logs via `loki.write` to cloud endpoints
- Authentication via API keys
- Automatic retention management (default: metrics 13 months, logs 30 days)
- Pre-built dashboards for platform components

**In-Cluster Stack (On-Premises Profile)**:
- Self-hosted Prometheus, Loki, and Grafana
- Grafana Alloy configured with local service endpoints
- Prometheus: local retention (typically 15-30 days), configurable storage capacity
- Loki: S3-compatible backend for log storage beyond active query window
- Grafana: unified visualization with datasources for local Prometheus and Loki
- Platform-specific dashboards for pipeline execution, resource utilization, health

---

### 8. Authentication Contract

**Purpose**: Unified identity and access management for all platform interactions

**Type**: Platform-Provided

**Required Capabilities**:
- OIDC/OAuth2 authentication flows
- Identity federation with external providers
- Service account token generation for machine-to-machine auth
- Role-based access control (RBAC)
- Token validation and refresh

**Interface Specification**:
- OIDC discovery endpoint (`/.well-known/openid-configuration`)
- OAuth2 authorization code flow
- Token introspection endpoint
- Service account creation and credential management
- RBAC policy definition

**Behavioral Requirements**:
- GitHub OIDC token exchange for CI/CD authentication
- CLI authentication via OAuth2 device flow or browser redirect
- Service-to-service authentication via service account tokens
- Token expiration and refresh (default: 60 minutes access token, 30 days refresh token)
- Audit logging of authentication events

**Adapter Implementation**:

**Keycloak (Both Profiles)**:
- Same implementation for AWS and on-premises
- Centralized authentication for all platform access
- GitHub OIDC provider federation
- Internal service accounts for worker-to-API communication
- RBAC managed through Keycloak roles and groups
- PostgreSQL backend for user/session storage
- High availability via multiple replicas

**IRSA Exception (AWS Profile Only)**:
- Workers use IRSA for AWS service access (S3, Secrets Manager, ECR). IRSA operates at infrastructure level and does not replace Keycloak. Workers authenticate to Platform API via Keycloak service accounts.

---

### 9. Workload Scaling Contract

**Purpose**: Automatic horizontal scaling of workloads based on demand

**Type**: Platform-Provided

**Required Capabilities**:
- Pod-level autoscaling based on external metrics
- Node-level autoscaling based on pending pod requirements
- Scale-to-zero support for idle workloads
- Metric source integration (NATS consumer lag, custom metrics)
- Rapid scaling response (< 1 minute for pods, < 2 minutes for nodes)

**Interface Specification**:
- KEDA ScaledObject CRD for pod scaling configuration
- Metric scalers: NATS JetStream consumer lag, Prometheus, custom
- Karpenter NodePool or equivalent for node provisioning
- Kubernetes HPA integration for seamless scaling

**Behavioral Requirements**:
- Pod scaling: KEDA monitors metrics, scales Deployment/StatefulSet
- Node scaling: Autoscaler provisions nodes when pods are pending
- Scale-down: Remove underutilized nodes after grace period
- Platform workers scale on NATS consumer lag (queue depth)
- Application workloads can define custom scaling triggers

**Adapter Implementations**:

**KEDA + Karpenter (AWS Profile)**:
- KEDA for pod-level scaling based on NATS JetStream consumer lag
- Karpenter for dynamic node provisioning
- Automatic instance type selection for cost optimization
- Spot instance support for non-production workloads
- Sub-minute node provisioning
- Automatic deprovisioning of empty nodes

**KEDA + Cluster Autoscaler (On-Premises Profile)**:
- KEDA for pod-level scaling (same as AWS)
- Kubernetes Cluster Autoscaler for node management
- Integration with infrastructure provider (OpenStack, VMware, bare metal)
- Node pools with defined instance types
- Slower provisioning (depends on infrastructure provider)

---

### 10. Database Contract

**Purpose**: Application database provisioning with automated lifecycle management

**Type**: Platform-Provided

**Required Capabilities**:
- Relational database instance provisioning (PostgreSQL, MySQL)
- Automated backups and point-in-time recovery
- Connection pooling and credential management
- High availability with automated failover
- Storage scaling without downtime

**Interface Specification**:
- Crossplane Database XRD for declarative provisioning
- Standard SQL connection endpoints
- Credentials delivered via Kubernetes Secrets (External Secrets integration)
- Connection string format: `postgres://host:port/database`

**Behavioral Requirements**:
- Database provisioning within 10 minutes (managed) or 5 minutes (operator-based)
- Automatic credential rotation support
- Daily automated backups with configurable retention
- Connections via connection pooling (PgBouncer or equivalent)
- Monitoring and alerting integration

**Adapter Implementations**:

**Amazon RDS (AWS Profile)**:
- Managed PostgreSQL and MySQL databases
- IRSA authentication for administrative operations
- Multi-AZ deployment for high availability
- Automated backups to S3 with configurable retention (default: 7 days)
- CloudWatch metrics and logging
- Storage autoscaling up to configured maximum
- Credential rotation via AWS Secrets Manager integration
- Connection pooling via RDS Proxy (optional)

**CloudNativePG (On-Premises Profile)**:
- Kubernetes-native PostgreSQL operator
- Declarative cluster management via CRDs
- Streaming replication for high availability
- Automated backup to S3-compatible storage (MinIO, local)
- Point-in-time recovery support
- Connection pooling via PgBouncer sidecar
- Monitoring integration with Prometheus
- Storage expansion via PVC resizing
- Self-healing with automatic failover

**Note**: Platform services currently use a shared RDS PostgreSQL instance per environment. This contract enables applications to provision dedicated database instances.

---

### 11. Network Contract (with Service Mesh)

**Purpose**: Service exposure, DNS management, and service mesh policy configuration

**Type**: Platform-Provided

**Required Capabilities**:
- HTTP/HTTPS service exposure with TLS termination
- Automatic DNS record management
- Service-to-service authorization policies
- Traffic management (retries, timeouts, circuit breaking)
- Observability configuration (tracing, metrics)

**Interface Specification**:
- Crossplane Network XRD for service exposure and DNS
- Service mesh policy fields for authorization and traffic management
- Gateway API resources for ingress configuration
- Istio AuthorizationPolicy for cross-service communication

**Behavioral Requirements**:
- DNS record creation within 60 seconds
- Automatic TLS certificate provisioning via cert-manager
- Service mesh policies applied within 30 seconds
- Default-deny network policies with explicit allow rules
- Encrypted service-to-service communication (mTLS)

**Adapter Implementations**:

**Envoy Gateway + Istio Ambient Mode (Both Profiles)**:
- Same implementation for AWS and on-premises
- **Envoy Gateway**: North-south traffic (external → cluster)
  - Implements Kubernetes Gateway API
  - TLS termination with cert-manager integration
  - Rate limiting and request authentication
- **Istio Ambient Mode**: East-west traffic (service → service)
  - Sidecar-less service mesh (ztunnel for L4, waypoint for L7)
  - Automatic mTLS between services
  - Service-to-service authorization via AuthorizationPolicy
  - Traffic management (retries, timeouts, circuit breaking)
  - Default namespace isolation
- **External DNS**: Automatic DNS record management
  - Route53 (AWS) or any ExternalDNS-compatible provider (on-premises)
  - Synchronizes DNS from Gateway and Service resources

**Network XRD Configuration**:
Applications configure both ingress and service mesh policies through a single Network XRD:
- `subdomain`: DNS subdomain for external access
- `port`: Container port to expose
- `authorizationPolicies`: List of allowed source services/namespaces
- `trafficPolicy`: Retry, timeout, circuit breaker configuration

Istio Ambient Mode deploys platform-wide. Applications configure policies through the Network XRD while the service mesh provides automatic encryption and traffic management.

---

## Configuration

### Adapter Selection

Adapter selection is environment-specific and configured at platform deployment time.

For Crossplane-managed resources, adapter selection is controlled via EnvironmentConfig resources that specify deployment profile and composition selectors. The platform uses these configurations to choose appropriate compositions based on infrastructure provider labels.

For platform services, adapter selection is configured through platform deployment values that specify which adapter implementation to use per contract (e.g., S3 vs MinIO for object storage, AWS Secrets Manager vs Vault for secret management).

### Contract Verification

Each adapter implementation must satisfy contract requirements:

1. **Capability Verification**: Automated tests validate required capabilities
2. **Interface Compliance**: Contract tests ensure API compatibility
3. **Behavioral Validation**: Integration tests verify behavioral requirements
4. **Performance Testing**: Load tests confirm scalability requirements

## Integration Points

### Application Integration

Applications interact with contracts through platform abstractions:

**XRD Usage** (Cluster Provisioning Contract):
```yaml
apiVersion: forge.io/v1
kind: Deployment
metadata:
  name: myapp
spec:
  containers:
    - name: web
      image: @artifact("web.uri")
      env:
        SECRET_VALUE:
          source: secrets/myapp-secrets/database/password  # Secret Management Contract
```

**Direct Integration** (Platform Services):
- Object Storage: Workers use S3-compatible SDK
- Message Queue: Workers use NATS client libraries
- Authentication: All services validate Keycloak tokens

### Platform Service Integration

Platform services interact with contracts via:

**Object Storage**: S3-compatible SDK (AWS SDK or MinIO client)
**Container Registry**: OCI distribution client
**Secret Management**: External Secrets Operator or direct API
**DNS Management**: External DNS operator
**Message Queue**: NATS JetStream client
**Observability**: Grafana Alloy configuration

## Constraints

### Hard Requirements

**All Contracts**:
- Must support Kubernetes integration
- Must provide health check/readiness endpoints
- Must support TLS/encryption in transit

**Platform-Provided Contracts**:
- Must deploy via GitOps (Argo CD)
- Must support high availability (multi-replica)
- Must integrate with platform observability

**External Required Contracts**:
- Must provide stable API for integration
- Must support authentication mechanisms
- Must meet platform SLAs (uptime, latency)

### Adapter Constraints

**AWS Profile**:
- Single region deployment (multi-region not supported)
- IRSA available for EKS clusters only
- Managed service quotas apply

**On-Premises Profile**:
- Kubernetes 1.28+ required
- Sufficient cluster resources for self-hosted services
- Network policies for service isolation
- Persistent volume provisioner required

## Future Considerations

### Additional Contracts

**Cache Contract**:
- Distributed caching layer (Redis, Memcached)
- Redis-compatible API
- Automatic failover and scaling
- AWS adapter: ElastiCache
- On-premises adapter: Redis Operator or Bitnami Redis Helm chart

### Adapter Implementations

**Multi-Cloud Adapters**:
- GCP equivalents (GCS, GCR, Secret Manager, Cloud DNS)
- Azure equivalents (Blob Storage, ACR, Key Vault, Azure DNS)

**Hybrid Adapters**:
- Mix cloud and on-premises per contract
- Disaster recovery across providers

---

**Related Documents:**
- Architecture Overview - Ports and adapters pattern and contract catalog
- Implementation Guide - Deployment profiles and adapter selection
- System Architecture - Service architecture and infrastructure integration

**Last Updated**: 2025-10-05
