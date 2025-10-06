# Implementation Guide

## Table of Contents

- [Executive Summary](#executive-summary)
- [Core Concepts](#core-concepts)
  - [Deployment Profiles](#deployment-profiles)
  - [Self-Hosting Model](#self-hosting-model)
  - [Control Plane / Data Plane Separation](#control-plane--data-plane-separation)
- [Platform Deployment Architecture](#platform-deployment-architecture)
  - [Cluster Topology](#cluster-topology)
  - [Bootstrapping Sequence](#bootstrapping-sequence)
  - [Cluster Lifecycle Management](#cluster-lifecycle-management)
- [Production/AWS Profile](#productionaws-profile)
  - [Overview](#overview)
  - [Adapter Implementations](#adapter-implementations)
  - [External Requirements](#external-requirements)
  - [Network Architecture](#network-architecture)
- [On-Premises Profile](#on-premises-profile)
  - [Overview](#overview-1)
  - [Adapter Implementations](#adapter-implementations-1)
  - [External Requirements](#external-requirements-1)
  - [Network Architecture](#network-architecture-1)
- [Infrastructure Abstractions](#infrastructure-abstractions)
  - [Crossplane XRD Catalog](#crossplane-xrd-catalog)
  - [Environment Configuration Model](#environment-configuration-model)
  - [Reference Resolution Pattern](#reference-resolution-pattern)
- [Constraints](#constraints)
  - [Platform Deployment Constraints](#platform-deployment-constraints)
  - [Adapter Selection Constraints](#adapter-selection-constraints)
  - [Infrastructure Abstraction Constraints](#infrastructure-abstraction-constraints)
- [Future Considerations](#future-considerations)
  - [Potential Enhancements](#potential-enhancements)
  - [Extension Points](#extension-points)

## Executive Summary

This document defines how the Catalyst Forge platform is implemented across different infrastructure environments through deployment profiles and adapter selection. The platform implements a self-hosting model with control plane/data plane separation, where a central platform cluster manages multiple environment clusters through GitOps.

The guide covers three main areas: platform deployment architecture (how the platform deploys itself), implementation profiles (AWS vs on-premises adapter selections), and infrastructure abstractions (Crossplane XRDs for application deployment).

## Core Concepts

### Deployment Profiles

The platform supports two deployment profiles that differ only in adapter selection for platform contracts:

**Production/AWS Profile:** Leverages AWS-managed services where they provide operational advantages (S3, ECR, RDS, Secrets Manager) while deploying self-hosted components for platform-specific functionality (NATS, Keycloak).

**On-Premises Profile:** Uses self-hosted, open-source implementations for all infrastructure dependencies (MinIO, Harbor, CloudNativePG, Vault) to enable deployment on any Kubernetes distribution.

Applications interact exclusively with platform contracts. Adapter selection occurs at platform deployment time through configuration, requiring no changes to application code or deployment definitions.

### Self-Hosting Model

The platform deploys and manages itself through the same mechanisms it provides to applications. After initial bootstrap, all platform components are managed as GitOps applications in Argo CD, enabling the platform to evolve through standard deployment processes.

Benefits:
- Consistent deployment patterns between platform and applications
- Declarative infrastructure management with full audit history
- Ability to version and rollback platform changes
- Reduced operational complexity through unified tooling

### Control Plane / Data Plane Separation

The platform topology separates control plane (management) from data plane (workload execution) across distinct Kubernetes clusters:

**Platform Cluster (Control Plane):**
- Hosts platform services (API, Worker Service, Dispatcher)
- Manages GitOps orchestration through central Argo CD
- Provisions and manages environment clusters through Crossplane
- Coordinates release and deployment workflows
- Operates as persistent infrastructure

**Environment Clusters (Data Planes):**
- Execute application workloads in isolated environments
- Managed remotely from platform cluster via Argo CD
- Provisioned on-demand through Crossplane XRDs
- Operate as ephemeral infrastructure (can be destroyed and recreated)
- Each environment contains identical platform operators for consistency

## Platform Deployment Architecture

### Cluster Topology

```
┌────────────────────────────────────────────────────────────┐
│                    Platform Cluster                        │
│                  (Control Plane - Persistent)              │
│                                                            │
│  Platform Services:                                        │
│    • API Server                                            │
│    • Worker Service (StatefulSets)                         │
│    • Argo Workflows                                        │
│    • NATS JetStream                                        │
│                                                            │
│  Management Tools:                                         │
│    • Argo CD (central instance)                            │
│    • Crossplane (cluster provisioning)                     │
│    • Platform operators                                    │
│                                                            │
│  Data Stores:                                              │
│    • PostgreSQL (platform state)                           │
│    • Shared secret backend                                 │
└──────────────────┬─────────────────────────────────────────┘
                   │
                   │ Manages via GitOps + Crossplane
                   │
       ┌───────────┴────────────┬────────────────┐
       │                        │                │
       ▼                        ▼                ▼
┌─────────────┐         ┌─────────────┐   ┌─────────────┐
│  Dev Env    │         │  Staging    │   │ Production  │
│  Cluster    │         │   Cluster   │   │   Cluster   │
│ (Ephemeral) │         │ (Ephemeral) │   │ (Ephemeral) │
│             │         │             │   │             │
│ Applications│         │ Applications│   │ Applications│
│ + Operators │         │ + Operators │   │ + Operators │
└─────────────┘         └─────────────┘   └─────────────┘
```

### Bootstrapping Sequence

The platform follows a five-phase bootstrapping approach:

**Phase 1: Prerequisites**

Manual infrastructure provisioning establishes the bare minimum to run Argo CD:

Production/AWS:
- Provision EKS cluster for platform cluster
- Configure VPC with private/public subnets
- Establish Route53 hosted zone for DNS delegation
- Configure AWS credentials for Crossplane (IRSA setup)

On-Premises:
- Provision Kubernetes cluster (any distribution)
- Configure DNS provider credentials
- Establish TLS certificate authority (internal CA or Let's Encrypt)
- Configure storage class for persistent volumes

**Phase 2: GitOps Foundation**

Deploy Argo CD as the platform's deployment engine:
- Install Argo CD via Helm chart or manifests
- Configure Argo CD to manage itself (app-of-apps pattern)
- Connect Argo CD to platform GitOps repository
- Configure credentials for GitOps repository access
- Enable ApplicationSets for multi-cluster management

**Phase 3: Infrastructure Abstraction Layer**

Deploy Crossplane to enable infrastructure provisioning:
- Install Crossplane via Argo CD
- Deploy providers (provider-aws, provider-helm, provider-kubernetes)
- Configure provider authentication (IRSA for AWS, ServiceAccounts for Kubernetes)
- Install platform XRD catalog
- Configure EnvironmentConfigs for platform cluster

Crossplane then provisions platform infrastructure:

Production/AWS:
- RDS PostgreSQL instance for platform state
- S3 buckets for artifact storage
- AWS Secrets Manager secret paths
- IRSA roles for platform services

On-Premises:
- CloudNativePG PostgreSQL cluster
- MinIO deployment for object storage
- Vault deployment for secret management

**Phase 4: Platform Services**

Deploy core platform components via Crossplane XRDs (reconciled by Argo CD):
- NATS JetStream cluster (Stateful XRD)
- PostgreSQL schema migrations (Job XRD)
- Platform API server (Deployment XRD)
- Worker Service StatefulSets (Stateful XRD)
- Argo Workflows engine (Deployment XRD)
- Service mesh configuration (Istio Ambient Mode via platform operators)
- Observability stack (Grafana Alloy via platform operators)

Argo CD reconciles XRD manifests from GitOps repository, Crossplane compositions create the underlying Kubernetes resources.

**Phase 5: Environment Provisioning**

Create environment clusters through Crossplane:
- Deploy environment cluster XRDs (EKS or generic Kubernetes)
- Provision VPCs/networks for each environment
- Deploy platform operators to each environment cluster via ApplicationSets
- Configure EnvironmentConfigs per environment
- Establish DNS delegation for environment domains

### Cluster Lifecycle Management

**Platform Cluster:**
- Provisioning: Manual bootstrap via infrastructure-as-code
- Lifecycle: Persistent, long-lived infrastructure
- Updates: In-place via GitOps (Argo CD manages itself)
- Disaster Recovery: Backup and restore procedures, cluster recreation from GitOps repository

**Environment Clusters:**
- Provisioning: Automated via Crossplane XRDs from platform cluster
- Lifecycle: Ephemeral, can be destroyed and recreated without data loss
- Updates: Managed remotely via Argo CD ApplicationSets
- Deletion: Declarative removal through Crossplane, applications redeploy on recreation

## Production/AWS Profile

### Overview

The Production/AWS profile leverages AWS managed services where they provide operational advantages while deploying self-hosted components for platform-specific functionality. This profile optimizes for reliability, scalability, and reduced operational overhead.

**Target Environments:**
- Production workloads on AWS
- Staging environments on AWS
- Development environments on AWS (with cost-optimized configurations)

**Infrastructure Foundation:**
- Amazon EKS for Kubernetes clusters
- VPC with Transit Gateway for network topology
- IRSA for pod-level AWS service authentication
- Multi-AZ deployment for high availability

### Adapter Implementations

See Platform Contracts for complete contract specifications. This section defines which adapters fulfill each contract in the AWS profile.

**Object Storage: Amazon S3**
- Native AWS S3 buckets with lifecycle policies
- Authentication via IRSA
- Bucket naming: `catalyst-<environment>-<purpose>`
- Server-side encryption enabled

**Container Registry: Amazon ECR**
- Private ECR repositories per project
- Image scanning enabled
- Authentication via IRSA
- Repository naming: `<repository>/<project>/<artifact-type>`

**Secret Management: AWS Secrets Manager**
- Hierarchical path structure for secrets
- Automatic rotation for supported secret types
- External Secrets Operator synchronizes to Kubernetes Secrets
- Path structure: `<environment>/<purpose>/*` or `global/*`

**DNS Management: Route53**
- Delegated zone per environment: `<environment>.<region>.projectcatalyst.io`
- ExternalDNS operator manages record lifecycle
- Automatic cleanup on resource deletion

**Message Queue: NATS JetStream**
- Self-hosted NATS cluster within platform cluster
- 3-replica cluster for high availability (production)
- EBS-backed persistent volumes for stream storage
- TLS encryption for client connections

**Cluster Provisioning: Amazon EKS + Crossplane**
- Crossplane provider-aws for infrastructure management
- EKS clusters with Karpenter for dynamic node provisioning
- VPC creation with public/private/database subnets
- Transit Gateway attachments for network connectivity

**Observability: Grafana Cloud**
- Grafana Alloy Operator for metrics and log collection
- Remote write to Grafana Cloud endpoints
- Centralized dashboards and alerting

**Authentication: Keycloak**
- Self-hosted Keycloak deployment on platform cluster
- RDS PostgreSQL backend for Keycloak state
- Federation to external identity providers (OIDC/SAML)
- GitHub OIDC integration for pipeline authentication

**Workload Scaling: KEDA + Karpenter**
- KEDA for pod-level autoscaling based on metrics
- Karpenter for node-level autoscaling
- Integration with NATS JetStream metrics for worker scaling

**Database: Amazon RDS**
- Managed PostgreSQL instances via Crossplane XRD
- Multi-AZ deployment for production
- Automated backups and point-in-time recovery
- Connection credentials via External Secrets

**Network (with Service Mesh): Istio Ambient Mode**
- Unified ingress and service mesh via Istio
- Implements Kubernetes Gateway API for ingress
- Automatic mTLS for all cluster traffic
- Service-to-service authorization policies

### External Requirements

**DNS Provider:** Route53 (AWS-managed)

**Identity Provider:** External OIDC or SAML provider for Keycloak federation

**Load Balancer:** AWS Load Balancer Controller provisions ELBs/ALBs/NLBs

**Storage:** EBS CSI Driver provisions persistent volumes

### Network Architecture

**Hub-and-Spoke Topology:**
- Central Transit Gateway connects all VPCs
- Centralized egress VPC for cost optimization
- Environment clusters in dedicated VPCs
- DNS delegation per environment

**Security:**
- Network isolation via VPCs and security groups
- No direct internet access for workloads
- Zero-trust networking via Istio Ambient Mode
- TLS everywhere via cert-manager and Istio

## On-Premises Profile

### Overview

The On-Premises profile uses self-hosted, open-source implementations for all infrastructure dependencies to enable deployment on any Kubernetes distribution without cloud provider dependencies.

**Target Environments:**
- On-premises data centers
- Private cloud environments
- Development environments on local Kubernetes

**Infrastructure Foundation:**
- Any Kubernetes distribution (k3s, RKE2, vanilla Kubernetes)
- Standard Kubernetes networking (CNI plugin of choice)
- ServiceAccount-based authentication for all components

### Adapter Implementations

**Object Storage: MinIO**
- Self-hosted S3-compatible object storage
- Distributed mode for high availability
- Authentication via access keys
- Bucket policies for access control

**Container Registry: Harbor**
- Self-hosted OCI-compliant registry
- Vulnerability scanning with Trivy
- Replication for disaster recovery
- Authentication via Harbor users or OIDC

**Secret Management: HashiCorp Vault or Kubernetes Secrets**
- Vault for production deployments (encryption, rotation, audit)
- Kubernetes Secrets for development environments
- External Secrets Operator synchronizes to Kubernetes Secrets
- Hierarchical path structure similar to AWS

**DNS Management: External Provider**
- ExternalDNS-compatible provider required (Cloudflare, PowerDNS, RFC2136)
- Manual configuration of DNS credentials
- Automatic record lifecycle management

**Message Queue: NATS JetStream**
- Identical to AWS profile
- Self-hosted NATS cluster
- Persistent volumes via local storage or distributed storage

**Cluster Provisioning: Crossplane + Provider**
- Crossplane provider-helm for cluster bootstrapping
- Crossplane provider-kubernetes for namespace and resource provisioning
- Manual cluster creation, automated configuration

**Observability: In-Cluster Prometheus/Loki/Grafana**
- Grafana Alloy Operator for metrics and log collection
- In-cluster Prometheus for metrics storage
- In-cluster Loki for log aggregation
- Grafana for unified dashboards

**Authentication: Keycloak**
- Identical to AWS profile
- CloudNativePG PostgreSQL backend for Keycloak state
- Federation to external identity providers or local user database

**Workload Scaling: KEDA + Cluster Autoscaler**
- KEDA for pod-level autoscaling
- Cluster Autoscaler for node-level scaling
- Integration with NATS JetStream metrics

**Database: CloudNativePG**
- Operator-managed PostgreSQL clusters
- Automated backups to object storage
- High availability via streaming replication
- Connection credentials via Kubernetes Secrets

**Network (with Service Mesh): Istio Ambient Mode**
- Identical to AWS profile
- Unified ingress and service mesh via Istio
- Implements Kubernetes Gateway API for ingress

### External Requirements

**DNS Provider:** Cloudflare, PowerDNS, RFC2136, or other ExternalDNS-compatible provider

**Identity Provider:** External OIDC or SAML provider for SSO (optional, can use Keycloak local users)

**Load Balancer:** MetalLB, hardware load balancer, or cloud provider load balancer

**Storage:** Longhorn, OpenEBS, Ceph, or vendor CSI driver for persistent volumes

### Network Architecture

**Kubernetes-Native Networking:**
- Network policies for isolation
- Ingress through shared load balancer with virtual hosts
- DNS through external provider
- Service mesh (Istio Ambient) for zero-trust networking

## Infrastructure Abstractions

### Crossplane XRD Catalog

The platform provides Crossplane Composite Resources (XRDs) that abstract common infrastructure patterns. Applications declare XRDs in CUE configuration, and the platform renders them to Kubernetes resources during release creation.

| XRD            | Purpose                           | Key Features                                  |
| -------------- | --------------------------------- | --------------------------------------------- |
| **Deployment** | Stateless services                | Horizontal scaling, rolling updates, health checks |
| **Stateful**   | Services with persistent identity | Ordered deployment, stable storage            |
| **Job**        | One-time tasks                    | Completion tracking, retry policies           |
| **CronJob**    | Scheduled tasks                   | Cron syntax, concurrency control              |
| **Network**    | Service exposure & DNS            | HTTP routing, DNS records, TLS termination    |
| **Secrets**    | External secrets sync             | Secret provider integration, automatic rotation |
| **ConfigMaps** | Application configuration         | Key-value pairs, file-based configs           |
| **Storage**    | Persistent volumes                | Volume provisioning, backup scheduling        |
| **Database**   | Application database provisioning | RDS (AWS) or CloudNativePG (on-premises)      |

All XRDs support output publishing through `status.outputs.*` for public values and connection-details Secrets for sensitive values. This enables XRDs to act as producers and consumers in a dependency graph.

See Developer Guide: Crossplane XRDs and Deployment Configuration for complete usage patterns and configuration examples.

### Environment Configuration Model

The platform uses Crossplane EnvironmentConfigs to provide environment-specific configuration at deployment time, enabling environment-agnostic releases.

**Two-Tier Configuration:**

**Cluster-Wide Configuration:** Stored in `<env>/env.yml` in GitOps repository
- Environment identification (name, domain, region)
- Platform conventions (secret prefixes, storage classes)
- Resource defaults per workload type
- Environment-wide feature flags

**Per-Project Configuration:** Stored in `<env>/<repo>-<project>/env.yml` in GitOps repository
- Workload-specific tuning (replicas, resources)
- Schedule modifications for CronJobs
- Feature toggles per project
- Temporary overrides

**Resolution Precedence:**
1. Per-project override (specific overrides for named XRD instances)
2. XRD spec value (developer-specified configuration)
3. Cluster-wide default (environment-level defaults)
4. Composition default (hardcoded fallbacks)

This precedence ensures predictable behavior: developers specify intent, environment configs apply context-specific defaults, and compositions provide reasonable fallbacks.

### Reference Resolution Pattern

The platform provides a universal reference pattern for accessing values from other XRDs:

**Reference Types:**

`outputs/<xr>/<key>` - Literal value inlined from any XR's `status.outputs.*`

`connections/<xr>/<key>` - Reference to any XR's connection-details Secret (secretKeyRef)

`secrets/<xr>/<secret>/[key]` - Reference to secret from Secrets XRD (secretKeyRef)

`configs/<xr>/<config>/[key]` - Reference to ConfigMap from ConfigMaps XRD (configMapKeyRef)

**Security Model:**
- `outputs/` returns literal values (treated as public, safe to log)
- All other reference types return Kubernetes references (secretKeyRef/configMapKeyRef)
- Sensitive values never inlined, always referenced

**Cross-Namespace References:**
- `<repository>-<project>::outputs/<xr>/<key>` for cross-project references
- `platform::outputs/<xr>/<key>` for platform-managed resources
- Only `outputs/` supports cross-namespace references
- `secrets/` and `configs/` limited to same namespace

See Developer Guide: Reference Resolution for complete syntax and examples.

## Constraints

### Platform Deployment Constraints

- Platform cluster must be manually bootstrapped (cannot self-bootstrap)
- Single platform cluster per installation (multi-region requires separate installations)
- Environment clusters ephemeral, but platform cluster persistent
- Profile selection fixed at cluster creation (cannot change AWS → on-premises)

### Adapter Selection Constraints

- Adapters selected per cluster, not per application
- All applications in environment use same adapters
- Migration between profiles requires platform redeployment
- Custom adapters require platform modifications

### Infrastructure Abstraction Constraints

- XRDs defined at platform level (developers cannot create custom XRDs)
- EnvironmentConfig changes require GitOps repository updates
- Reference resolution limited to same-namespace except `outputs/`
- Composition functions defined in platform, not per-project

## Future Considerations

### Potential Enhancements

**Hybrid Profiles:**
- Mix AWS-managed and self-hosted adapters within single profile
- Per-contract adapter selection for flexibility

**Additional Profiles:**
- GCP profile with Google Cloud managed services
- Azure profile with Azure managed services
- Development profile with minimal resource requirements

**Enhanced Abstractions:**
- Additional XRDs for specialized workloads (ML training, GPU workloads)
- Policy-as-code for compliance requirements
- Cost optimization XRDs with budget controls

### Extension Points

- Platform Contracts enable new adapter implementations
- Crossplane composition functions support custom logic
- EnvironmentConfig extensible for new configuration patterns
- GitOps repository structure supports additional environments

---

**Related Documents:**
- Platform Contracts - Infrastructure contract specifications and adapter implementations
- System Architecture - Service architecture and component interactions
- Developer Guide - XRD configuration patterns and deployment workflows
- Architecture Overview - Ports and adapters pattern and architectural principles

**Last Updated:** 2025-10-05
