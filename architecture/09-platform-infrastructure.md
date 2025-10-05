# Platform Infrastructure Specification

## Executive Summary

This document defines the infrastructure architecture for the Catalyst Forge platform. The platform operates on AWS with EKS as the Kubernetes foundation, implementing a hub-and-spoke network topology with centralized egress control. All infrastructure follows Infrastructure as Code principles, with environments deployed as identical clones differentiated only by sizing, permissions, and runtime configuration.

The platform explicitly adopts an AWS-first strategy, leveraging managed services where appropriate while maintaining clear architectural boundaries for potential future evolution.

## Core Concepts

### Infrastructure Philosophy

**Immutable Infrastructure** - All infrastructure is declaratively defined and version-controlled. Changes are applied through controlled deployments, not manual modifications.

**Environment Uniformity** - Environments are architecturally identical, avoiding "works in dev, fails in prod" infrastructure discrepancies.

**Network Isolation with Controlled Connectivity** - Complete environment isolation at network boundaries with explicit, auditable connectivity through Transit Gateway.

### AWS-First Architecture

The platform is designed for AWS and leverages AWS-native services for critical functionality. This is an explicit architectural decision, not a limitation. The AWS-first approach provides:
- Managed service reliability (EKS, RDS, Secrets Manager)
- Native security integration (IRSA, IAM)
- Operational simplicity through service integration

While the application layer remains cloud-agnostic through Kubernetes abstractions, the infrastructure layer explicitly depends on AWS services.

## Infrastructure Architecture

### Foundational AWS Dependencies

#### Tier 1: Critical Platform Requirements
These services are non-negotiable platform dependencies:

**EKS (Elastic Kubernetes Service)**
- Provides managed Kubernetes control plane
- Eliminates control plane operational overhead
- Integrates with AWS security and networking

**IAM/IRSA (IAM Roles for Service Accounts)**
- Pod-level AWS authentication without static credentials
- Fine-grained permission boundaries per workload
- Temporary credential rotation handled by AWS

**VPC/Transit Gateway**
- Network isolation and controlled routing
- Hub-and-spoke topology for centralized control
- Enables cost-optimized egress architecture

**Secrets Manager**
- Centralized secret storage with encryption at rest
- Automatic rotation capabilities
- Hierarchical organization with prefix-based access

**RDS PostgreSQL**
- Managed relational database per environment
- Automated backups and multi-AZ capability
- Reduces operational overhead for data persistence

**S3 (Simple Storage Service)**
- Pipeline artifacts and logs storage
- Persistent data for platform services
- Cost-effective object storage

#### Tier 2: Platform Operators
Required Kubernetes operators that enable platform functionality:

**AWS Load Balancer Controller**
- Provisions ELB/ALB/NLB from Kubernetes resources
- Manages target groups and health checks
- Enables service exposure without manual configuration

**External DNS**
- Synchronizes DNS records from Kubernetes resources
- Manages Route53 zones per environment
- Provides automatic DNS lifecycle management

**External Secrets Operator**
- Synchronizes Secrets Manager to Kubernetes Secrets
- Supports hierarchical secret organization
- Enables secure secret distribution without static embedding

**CSI Drivers (EBS, Snapshot Controller)**
- Dynamic persistent volume provisioning
- Snapshot capabilities for backup/restore
- Multiple storage classes for different performance needs

**cert-manager**
- Automated TLS certificate lifecycle management
- Let's Encrypt integration
- Eliminates manual certificate renewal

**Envoy Gateway**
- Implements Kubernetes Gateway API
- Advanced traffic management capabilities
- Foundation for Network XRD abstraction
- Handles north-south ingress traffic

**Argo CD**
- GitOps continuous delivery
- Declarative application deployment
- Provides deployment audit trail

**Karpenter**
- Dynamic node provisioning based on workload requirements
- Automatic instance type selection for cost optimization
- Spot instance management for non-production workloads
- Rapid scaling without pre-configured node groups
- Requires IRSA with EC2 instance lifecycle permissions

**KEDA (Kubernetes Event-Driven Autoscaling)**
- Scales workloads based on external metrics
- NATS JetStream consumer lag monitoring for worker pools
- Enables zero-to-n scaling for idle workloads
- Integrates with HPA for seamless scaling operations
- Critical for platform worker service elasticity

**Istio (Ambient Mode)**
- Sidecar-less service mesh for zero-trust networking
- End-to-end encryption for all cluster traffic
- Layer 4 and Layer 7 policy enforcement
- Service-to-service authorization policies
- Transparent to application workloads (no sidecar injection)

### Network Architecture Model

#### Hub-and-Spoke Topology
All VPCs connect through a central Transit Gateway, providing:
- Controlled inter-environment communication
- Centralized routing policies
- Simplified network management
- Single point for network monitoring

#### Centralized Egress
A dedicated egress VPC manages all outbound internet traffic:
- Cost optimization (single set of NAT gateways)
- Consistent egress IP addresses for allowlisting
- Centralized point for egress monitoring/filtering
- Simplified security posture management

#### DNS Hierarchy
Delegated zone structure: `<environment>.region.projectcatalyst.io`
- Environment isolation at DNS level
- Independent DNS management per environment
- Automated record management via External DNS

### Node Management Architecture

#### Dynamic Node Provisioning
Karpenter replaces traditional managed node groups with dynamic provisioning:
- **Workload-Driven Scaling**: Nodes provisioned based on actual pod requirements
- **Instance Type Optimization**: Automatic selection of most cost-effective instance types
- **Rapid Response**: Sub-minute node provisioning vs multi-minute with managed node groups
- **Automatic Deprovisioning**: Removes underutilized nodes to minimize costs

#### Spot Instance Strategy
Non-production environments leverage spot instances through Karpenter:
- **Cost Reduction**: Up to 90% savings for development and test workloads
- **Interruption Handling**: Automatic node replacement on spot termination
- **Mixed Instance Pools**: Combines spot and on-demand for baseline capacity
- **Production Isolation**: Production environments use only on-demand instances

#### Node Configuration Patterns
Karpenter provisioners define node characteristics per workload class:
- **General Purpose**: Balanced CPU/memory for typical applications
- **Compute Optimized**: High CPU for processing-intensive workloads
- **Memory Optimized**: Large memory for caches and databases
- **GPU Enabled**: For machine learning and specialized workloads (future)

### Workload Autoscaling Architecture

#### Two-Tier Scaling Model
The platform implements comprehensive autoscaling at both pod and node levels:

**Pod-Level Scaling (KEDA)**
- Scales StatefulSets and Deployments based on external metrics
- Platform workers scale on NATS JetStream consumer lag
- Application workloads can scale on custom metrics (future)
- Supports scale-to-zero for idle services

**Node-Level Scaling (Karpenter)**
- Provisions nodes in response to pending pod requirements
- Automatically adjusts cluster capacity as KEDA scales workloads
- Ensures resources are available for scaled pods
- Deprovisions empty nodes to minimize costs

#### Platform Worker Scaling
Worker services in the Platform Cluster demonstrate the scaling model:
- KEDA monitors NATS JetStream queue depth per worker type
- Scales worker StatefulSets from minimum to maximum replicas
- Karpenter provisions additional nodes when workers can't be scheduled
- Scale-down occurs when queues empty and nodes become underutilized

This two-tier approach ensures responsive scaling while maintaining cost efficiency.

### Zero-Trust Networking Architecture

#### End-to-End Encryption
Istio Ambient Mode provides comprehensive encryption without application changes:
- **North-South Traffic**: TLS from client through Envoy Gateway to services
- **East-West Traffic**: Automatic mTLS between all services within cluster
- **Transparent Encryption**: No application code changes or sidecar configuration
- **Certificate Management**: Automatic certificate rotation via Istio's built-in CA

#### Service-to-Service Authorization
Fine-grained access control between services:
- **Default Deny**: No inter-service communication unless explicitly allowed
- **Identity-Based Policies**: Services identified by their Kubernetes service accounts
- **Namespace Boundaries**: Automatic isolation between project namespaces
- **Progressive Enforcement**: Policies can start in monitoring mode before enforcement

#### Ambient Mode Architecture
Sidecar-less design reduces operational complexity:
- **ztunnel (Zero-Trust Tunnel)**: Per-node proxy for L4 processing
- **Waypoint Proxies**: Optional L7 processing for advanced features
- **Reduced Resource Overhead**: No per-pod sidecars consuming memory/CPU
- **Simplified Operations**: No sidecar injection or pod restarts required

#### Policy Model
Authorization policies follow allow-list pattern:
- Projects isolated by default in their `<repository>-<project>` namespaces (see Infrastructure Abstractions for namespace strategy)
- Cross-project communication requires explicit authorization policies
- Platform services have broader access for operational needs
- Ingress traffic controlled at Gateway level
- Egress traffic controlled via network policies and Istio authorization
- Policies can be managed through platform abstractions (future XRDs)

### Environment Architecture Model

#### Standard Environment Composition
Every environment contains identical infrastructure components:
- Dedicated VPC with public/private/database subnets
- EKS cluster with Karpenter for node management
- Full platform operator deployment (including Istio Ambient Mode)
- RDS PostgreSQL instance
- Connection to Transit Gateway
- Environment-specific Route53 zone

#### Platform Cluster
A dedicated "shared services" cluster hosts both the IDP platform and central management tools:
- Platform API Server
- Worker Services (StatefulSets)
  - Scaled dynamically by KEDA based on NATS JetStream queue depth
  - Separate scaling policies for discovery, CI, artifact, and release workers
- Argo Workflows engine
- NATS messaging infrastructure
- Central Argo CD instance
- From here, the platform manages all environment clusters remotely

#### Environment Differentiation
While architecturally identical, environments differ only in:

**Node Provisioning Strategy**
- Development: Aggressive spot instance usage, minimal baseline capacity
- Production: On-demand instances only, higher baseline capacity, multi-AZ distribution
- All environments: Karpenter-managed dynamic scaling based on workload demands

**Access Control**
- Development: Broader developer access for debugging
- Production: Restricted to platform operations and deployments

**Configuration Tuning**
- Via EnvironmentConfig (see Infrastructure Abstractions)
- Resource limits, replica counts, feature flags
- Applied at deployment time through Crossplane

**Database Configuration**
- Development: Single-AZ, minimal resources
- Production: Multi-AZ, enhanced monitoring, longer backup retention

### Data Persistence Architecture

#### Database Strategy
Single RDS PostgreSQL instance per environment with database-per-project model:
- Cost-efficient resource sharing
- Logical isolation at database level
- Centralized backup and maintenance
- Simplified credential management

Database credentials follow hierarchical structure:
- Root credentials: `<environment>/db/root_account`
- Project databases: Created on-demand
- Credentials synchronized via External Secrets

#### Storage Classes

The platform defines abstract storage classes that map to appropriate underlying storage providers based on the deployment environment. This abstraction enables applications to request storage characteristics without coupling to specific infrastructure implementations.

**Abstract Storage Classes:**

**`fast`:** High-performance storage for latency-sensitive workloads requiring consistent IOPS. Suitable for transactional databases, key-value stores, and applications with strict performance requirements.

**`standard`:** Balanced performance and cost for general-purpose workloads. Appropriate for most application persistent volumes, caches, and medium-performance databases.

**`bulk`:** Cost-optimized storage for large data volumes with infrequent access patterns. Designed for archives, backups, batch processing data, and log storage.

**Environment-Specific Mappings:**

**Production (AWS/EKS):**
- `fast` → io2 (provisioned IOPS SSD with high durability)
- `standard` → gp3 (general purpose SSD with baseline performance)
- `bulk` → sc1 (cold HDD for throughput-optimized workloads)

**On Premises:**
- `fast` → Longhorn/OpenEBS high-performance tier, NVMe local storage, or Ceph with SSD pools
- `standard` → Longhorn/OpenEBS standard tier or Ceph with HDD pools
- `bulk` → Network-attached storage or Ceph with erasure coding for capacity optimization

Applications reference abstract classes in their Storage XRD specifications. The platform's EnvironmentConfig system resolves these to concrete StorageClass names at deployment time, ensuring applications remain portable across environments.

### Observability Architecture

The platform standardizes on Grafana Alloy Operator for metrics collection and log aggregation across all environments. Alloy provides a unified agent that handles both metrics scraping and log collection, with configurable output destinations based on the deployment profile.

**Grafana Alloy Components:**

The Alloy agent operates as a DaemonSet on cluster nodes, collecting metrics and logs through a component-based architecture:

**`prometheus.scrape`:** Discovers and scrapes Prometheus metrics endpoints from platform services and applications. Supports service discovery via Kubernetes annotations and label selectors.

**`loki.source.kubernetes`:** Collects container logs from all pods. Enriches log entries with Kubernetes metadata including namespace, pod name, container name, and labels.

**`prometheus.remote_write`:** Forwards collected metrics to configured Prometheus-compatible endpoints. Supports buffering, retry logic, and compression.

**`loki.write`:** Ships collected logs to configured Loki endpoints. Handles batching, compression, and automatic retry on failures.

**Output Configuration by Environment:**

**Production (AWS/EKS) - Grafana Cloud:**
```yaml
prometheus.remote_write "cloud" {
  endpoint {
    url = "https://prometheus-prod-us-central-0.grafana.net/api/prom/push"
    basic_auth {
      username = "<prometheus-user-id>"
      password = env("GRAFANA_CLOUD_API_KEY")
    }
  }
}

loki.write "cloud" {
  endpoint {
    url = "https://logs-prod-us-central-0.grafana.net/loki/api/v1/push"
    basic_auth {
      username = "<loki-user-id>"
      password = env("GRAFANA_CLOUD_API_KEY")
    }
  }
}
```

**On Premises - In-Cluster Prometheus and Loki:**
```yaml
prometheus.remote_write "local" {
  endpoint {
    url = "http://prometheus.observability.svc.cluster.local:9090/api/v1/write"
  }
}

loki.write "local" {
  endpoint {
    url = "http://loki.observability.svc.cluster.local:3100/loki/api/v1/push"
  }
}
```

The Alloy configuration remains structurally identical across environments, with only endpoint URLs and authentication differing. This uniformity ensures consistent observability capabilities whether operating in cloud-managed services or self-hosted infrastructure.

**In-Cluster Stack Components (On Premises):**

When operating with in-cluster observability, the platform deploys:

**Prometheus:** Time-series metrics database with local retention (typically 15-30 days). Configured with appropriate storage capacity based on cluster size and metric volume.

**Loki:** Log aggregation system with configurable retention. Utilizes object storage backend (MinIO, S3-compatible) for long-term log retention beyond active query window.

**Grafana:** Unified dashboarding and visualization. Pre-configured with datasources for local Prometheus and Loki instances, including platform-specific dashboards for pipeline execution, resource utilization, and application health.

## Architectural Boundaries

### Platform vs Application Responsibilities

**Platform Provides:**
- Kubernetes clusters with standard operators
- Network connectivity and DNS
- Secret management integration
- Persistent storage capabilities
- Database instances (shared)
- Automatic TLS certificate provisioning and renewal
- Ingress/egress capabilities
- Zero-trust networking infrastructure (Istio Ambient)
- Default namespace isolation policies

**Applications Decide:**
- Resource requirements (via XRDs)
- Secret paths and organization
- Database usage (dedicated database within shared instance)
- DNS subdomain selection (via Network XRD)
- Service-to-service communication requirements (allowed connections)

### Infrastructure as Code Boundaries

**Platform Team Manages:**
- Core infrastructure (VPCs, EKS, RDS, Transit Gateway)
- Platform operators and configurations
- Cross-cutting concerns (networking, security groups)
- Environment provisioning

**Development Teams Define:**
- Application resources (via XRDs in CUE)
- Deployment configurations
- Application-specific secrets

## Security Architecture

### Defense in Depth
Multiple security layers provide comprehensive protection:

1. **Network isolation** via VPCs and security groups
2. **No direct internet access** for workloads (egress VPC only)
3. **Zero-trust networking** via Istio Ambient Mode:
   - End-to-end encryption for all traffic (mTLS)
   - Service-to-service authorization policies
   - Default-deny communication model
4. **IRSA** for AWS service authentication
5. **Secrets Manager** for credential storage
6. **RBAC** for Kubernetes access control
7. **TLS everywhere** via cert-manager and Istio

### Secret Management Hierarchy
Structured paths in AWS Secrets Manager:
- Environment-specific: `<environment>/<purpose>/*`
- Global secrets: `global/*`
- Platform secrets: `shared-services/*`

External Secrets Operator enforces prefix-based access ensuring environments cannot access each other's secrets.

## Architectural Decisions

### Single Region Design
Initial deployment in single AWS region (eu-central-1):
- Simplifies network architecture
- Reduces operational complexity
- Meets current latency requirements
- Multi-region possible in future but not initial requirement

### Uniform Infrastructure Principle
All environments run identical platform components:
- Consistent behavior across environments
- Simplified troubleshooting
- Single infrastructure template with parameters
- Reduces cognitive load for operators
- Zero-trust security without per-pod overhead (Ambient Mode)

### Cost Optimization Strategy
Several architectural decisions optimize costs:
- Centralized NAT gateways in egress VPC
- Shared RDS instances with logical database separation
- Karpenter-driven node optimization:
  - Automatic right-sizing of instances based on workload
  - Aggressive spot instance usage in non-production
  - Rapid deprovisioning of unused capacity
  - Instance type diversity to maximize spot availability
- KEDA and Karpenter synergy:
  - KEDA scales pods based on actual demand (including scale-to-zero)
  - Karpenter provisions/deprovisions nodes to match pod requirements
  - Eliminates over-provisioning at both pod and node levels
- Istio Ambient Mode instead of sidecar-based mesh:
  - No per-pod resource overhead
  - Fewer nodes required for same workload capacity

## Integration Points

### Platform Service Deployment
Platform services deploy to the shared services (platform) cluster:
- Provides network access to all environments via Transit Gateway
- Can assume roles in workload accounts if needed
- Maintains separation from workload environments

### Workload Deployment
Applications deploy to environment clusters:
- Access environment-specific RDS database
- Resolve secrets from environment prefix
- Ingress through environment-specific DNS zone (via Envoy Gateway)
- East-west traffic automatically encrypted (via Istio Ambient)
- Service communication controlled by authorization policies
- No direct access to platform cluster

### Cross-Environment Communication
Transit Gateway enables controlled communication:
- Platform services can reach all environments
- Environments isolated by default
- Explicit routes required for environment-to-environment communication

## Constraints and Limitations

### Hard Requirements
- AWS account with appropriate service quotas
- Single region deployment initially
- EKS version compatibility with platform operators
- RDS PostgreSQL 14+ for application databases
- Linux kernel 5.7+ for Istio Ambient Mode eBPF features
- EKS nodes with eBPF support enabled

### Architectural Constraints
- No multi-cloud support (AWS-specific)
- No on-premises connectivity initially
- Single RDS instance per environment (no per-project instances)
- Centralized egress (no direct internet from workload VPCs)

### Scaling Considerations
- Transit Gateway attachment limits (5000 per gateway)
- RDS connection limits based on instance size
- NAT Gateway bandwidth limitations
- EKS API rate limits for Karpenter node operations
- EC2 instance limits per region/account

## Future Considerations

### Potential Enhancements
- Multi-region deployment for disaster recovery
- PrivateLink endpoints for additional AWS services
- Database read replicas for scaling
- Enhanced network segmentation with NACLs

### Extension Points
- Additional environment types (ephemeral, sandbox)
- Alternative database engines (MongoDB Atlas, ElasticSearch)
- Advanced observability stack (distributed tracing via Istio)
- Policy-as-code for authorization rules

---

**Component Owner**: Platform Core
**Related Documents**: [Core Architecture](01-core-architecture.md), [Execution & Orchestration](03-execution-orchestration.md), [Infrastructure Abstractions](08-infrastructure-abstractions.md)
**Last Updated**: 2025-01-02