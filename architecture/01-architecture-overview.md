# Architecture Overview

## Table of Contents

- [Executive Summary](#executive-summary)
- [Core Concepts](#core-concepts)
  - [System Scope and Philosophy](#system-scope-and-philosophy)
  - [Ports and Adapters Architecture](#ports-and-adapters-architecture)
  - [Architectural Principles](#architectural-principles)
  - [Component Architecture](#component-architecture)
  - [Service Architecture](#service-architecture)
  - [Technology Stack](#technology-stack)
  - [Canonical Identifier Pattern](#canonical-identifier-pattern)
- [Integration Points](#integration-points)
  - [External Systems](#external-systems)
  - [Platform Operators](#platform-operators)
  - [Application Deployment](#application-deployment)
- [Constraints](#constraints)
  - [Platform Constraints](#platform-constraints)
  - [Scale Limits](#scale-limits)
  - [Technical Constraints](#technical-constraints)
  - [Security Boundaries](#security-boundaries)
- [Future Considerations](#future-considerations)
  - [Planned Enhancements](#planned-enhancements)
  - [Extension Points](#extension-points)

## Executive Summary

Catalyst Forge is an internal developer platform that manages the complete software development lifecycle from commit to deployment. The platform operates on AWS and supports on-premises deployments through a ports and adapters architecture that enables infrastructure portability without compromising on operational capabilities.

Built for engineering teams of 15-20 developers with capacity for growth, Catalyst Forge enforces conventions where consistency matters while providing flexibility where teams need it. The platform abstracts infrastructure complexity through well-defined contracts, allowing development teams to focus on application logic rather than operational details.

The architecture's defining characteristic is its systematic application of the ports and adapters pattern. Infrastructure dependencies are expressed as contracts that can be fulfilled by different implementations based on deployment environment. This design enables deployment on AWS-managed services or self-hosted alternatives through adapter substitution.

## Core Concepts

### System Scope and Philosophy

Catalyst Forge manages the complete SDLC for projects within GitHub repositories. The platform discovers projects through repository conventions, orchestrates build and test execution through configurable pipelines, packages artifacts with complete provenance tracking, and deploys applications through GitOps workflows.

The platform mandates workflow structure while allowing teams to control workflow content. Teams must use Earthly for builds and deploy to Kubernetes, but they control how they configure builds and what runs within each phase. Deployment abstractions through Crossplane are encouraged but raw Kubernetes resources remain available.

**Platform Capabilities:**
- Repository and project discovery
- Dynamic CI pipeline construction and execution
- Artifact build, publication, and tracking
- Release creation with immutable snapshots
- GitOps-based deployment management
- Environment-agnostic application configuration

**Explicit Constraints:**
- AWS and on-premises Kubernetes only (not multi-cloud)
- Earthly required for build execution
- Kubernetes required for application deployment
- GitHub required for source control

### Ports and Adapters Architecture

The platform defines clear contracts (ports) for all infrastructure dependencies, with multiple implementations (adapters) available based on deployment environment. This pattern ensures applications remain portable while leveraging environment-specific optimizations.

#### Contract Catalog

The platform defines eleven core contracts that abstract infrastructure dependencies:

| Contract                        | Purpose                                    | Platform-Provided                   |
| ------------------------------- | ------------------------------------------ | ----------------------------------- |
| Object Storage                  | S3-compatible artifact storage             | Yes                                 |
| Container Registry              | OCI image distribution                     | Yes                                 |
| Secret Management               | Credential storage and synchronization     | Yes                                 |
| DNS Management                  | Hostname registration and record lifecycle | No (external provider required)     |
| Message Queue                   | Work distribution with durable streams     | Yes                                 |
| Cluster Provisioning            | Kubernetes cluster lifecycle               | Partial (environment clusters only) |
| Observability                   | Metrics, logs, and trace collection        | Yes                                 |
| Authentication                  | Identity verification and OIDC federation  | Yes                                 |
| Workload Scaling                | Pod and node autoscaling                   | Yes                                 |
| Database                        | Application database provisioning          | Yes                                 |
| Network (with Service Mesh)     | Ingress and service-to-service policies    | Yes                                 |

Complete contract specifications are defined in Platform Contracts.

#### Deployment Profiles

The platform supports two deployment profiles that differ only in adapter selection:

**Production Profile (AWS):**
Leverages AWS-managed services where appropriate. Object storage uses S3, container registry uses ECR, secret management uses Secrets Manager, DNS management uses Route53. Authentication runs through Keycloak with AWS IAM integration via IRSA. External identity provider required for Keycloak federation.

**On-Premises Profile (Kubernetes):**
Uses self-hosted, open-source implementations. Object storage uses MinIO, container registry uses Harbor, secret management uses Vault or Kubernetes Secrets. DNS management requires external provider (Cloudflare, PowerDNS, RFC2136). Authentication runs through Keycloak with local user database or LDAP integration. External identity provider required for Keycloak federation if SSO desired.

Applications interact exclusively with contract interfaces. Adapter selection occurs at platform deployment time through configuration, requiring no changes to application definitions or pipeline configurations.

### Architectural Principles

#### Convention with Escape Hatches

The platform mandates workflow structure but not workflow contents. Teams must use the platform's pipeline phases and deployment model, but they control the specifics of each phase and can use raw Kubernetes resources when abstractions are insufficient.

#### Progressive Disclosure of Complexity

Projects start simple and evolve with need. A new project integrates with continuous integration through minimal configuration, then progressively adopts artifact publishing, release management, and infrastructure abstractions as requirements grow. The platform supports incremental adoption of capabilities as project requirements evolve.

#### Built for Change

The ports and adapters pattern ensures technology choices can evolve. Core workflows remain stable while underlying implementations can be swapped as technologies mature or requirements change. Migration from AWS to on-premises deployment requires only adapter reconfiguration, not application changes.

#### Immutability

Releases represent immutable snapshots of projects at specific commits. Once created, a release cannot be modified—only superseded by newer releases or redeployed to different environments. This immutability ensures reproducible deployments and enables reliable rollback operations.

#### Environment Agnosticism

Releases contain no environment-specific configuration. They package universal Kubernetes resource definitions that work across all environments, with environment-specific values provided at deployment time through Crossplane EnvironmentConfigs. This separation enables the same release to deploy to development, staging, and production without modification.

#### Sequential Phase Execution with Fail-Fast

Pipeline phases execute sequentially in group order, with only one phase group active at a time. Any failure immediately terminates the pipeline. This model aligns with the principle that continuous integration must maintain a green state.

### Component Architecture

The platform consists of four logical components that work together to provide complete SDLC management:

**Pipeline Component:**
Discovers projects, constructs dynamic CI pipelines, and orchestrates build execution. Manages state persistence, audit trails, and worker coordination.

**Project Component:**
Defines configuration schema and discovery mechanisms for projects within repositories. Projects are identified by `.forge` folders containing CUE configuration files that include deployment specifications.

**Artifacts Component:**
Manages build outputs and their publication to registries. All artifacts are tracked using OCI images for universal provenance and metadata management.

**Release & Deployment Component:**
Creates point-in-time snapshots of projects including their Kubernetes resource definitions. Projects primarily leverage platform-provided Crossplane Composite Resources, though raw Kubernetes resources are also supported. Integrates with Argo CD for GitOps-based deployments.

See System Architecture: Component Architecture and Data Flow Architecture for detailed interactions.

### Service Architecture

While the platform is organized into logical components, these components are realized through three core runtime services:

**Platform API:**
Single unified REST API service for all platform operations. Handles authentication via Keycloak, processes webhook ingestion from GitHub Actions, and manages all domain entities (PipelineRun, Release, Deployment). Deployed as horizontally-scaled Kubernetes Deployment.

**Worker Service:**
Executes all asynchronous job types via NATS JetStream pull consumers. Internal modules handle different job types (discovery, CI execution, artifact handling, release creation, deployment). Each worker maintains persistent Git repository cache. Deployed as multiple Kubernetes StatefulSets with different resource profiles, scaled by KEDA based on NATS consumer lag.

**Dispatcher:**
Lightweight ephemeral pods created by Argo Workflows. Submits jobs to NATS JetStream with reply subjects, awaits direct replies from workers, and returns results to workflow engine.

The messaging layer (NATS JetStream) is intentionally ephemeral. All durable state lives in PostgreSQL for audit trails and object storage for artifacts and logs. Lost messages result in timeouts and retries, not data loss.

See System Architecture: Service Communication Topology for detailed patterns.

### Technology Stack

**Configuration:** CUE for type-safe, composable configuration with built-in validation and Kubernetes resource generation.

**Build Execution:** Earthly for reproducible builds with native monorepo support.

**Deployment:** GitOps with Argo CD for declarative deployments with automatic drift detection.

**Infrastructure Abstraction:** Crossplane for abstracting Kubernetes complexity through Composite Resources. Platform provides curated XRD catalog with smart defaults.

**Orchestration:** Argo Workflows for Kubernetes-native workflow orchestration with built-in UI and retry logic.

**Messaging:** NATS JetStream for cluster-local replicated messaging providing work queues and request-reply patterns.

**Persistence:** PostgreSQL for relational state management and audit trails.

See Implementation Guide: Platform Deployment Architecture for complete infrastructure specifications including required platform operators.

### Canonical Identifier Pattern

All platform entities follow the `repository/project/commit-sha` pattern for global uniqueness. Repository provides organizational scope, project identifies the deliverable unit within that scope, and commit SHA provides immutable version identity. This triplet pattern enables globally unique identifiers without centralized coordination.

**Example Identifiers:**
- Artifact OCI: `registry.example.com/myapp/api-service/backend:sha-a1b2c3d4`
- Release OCI: `registry.example.com/myapp/api-service:a1b2c3d4`
- Namespace: `myapp-api-service`

The pattern appears consistently across artifact naming, release identification, deployment organization, and namespace generation.

## Integration Points

### External Systems

**GitHub:**
Source control and webhook trigger source. GitHub Actions workflows submit pipeline runs via CLI, which authenticates through GitHub OIDC tokens exchanged for Keycloak identity.

**DNS Provider:**
External DNS-compatible provider (Route53, Cloudflare, PowerDNS, RFC2136) required. Platform manages record lifecycle through ExternalDNS operator.

**Identity Provider:**
OIDC or SAML provider required for Keycloak federation. Platform authenticates all users and services through Keycloak, which delegates to external IdP for user authentication.

**Container Registries:**
OCI-compatible registries for artifact and release storage. Platform provides adapters for ECR, Harbor, and Docker Registry.

**Object Storage:**
S3-compatible storage for pipeline artifacts and logs. Platform provides adapters for S3, MinIO, and Ceph RGW.

### Platform Operators

The platform requires several Kubernetes operators for functionality. These operators are part of platform infrastructure, not application concerns.

**Universal Operators (both profiles):**
- External DNS (DNS record synchronization)
- External Secrets Operator (secret synchronization)
- Istio Ambient Mode (ingress via Gateway API and zero-trust service mesh)
- cert-manager (TLS certificate lifecycle)
- Argo CD (GitOps continuous delivery)
- Crossplane (infrastructure abstraction)
- KEDA (workload autoscaling)

**AWS-Specific Operators (production profile):**
- AWS Load Balancer Controller (ELB/ALB/NLB provisioning)
- Karpenter (dynamic node provisioning)
- EBS CSI Driver (persistent volume provisioning)

See Implementation Guide: Production/AWS Profile and On-Premises Profile for complete operator specifications and configuration.

### Application Deployment

Applications deploy through the following flow:

1. Pipeline discovers projects and constructs execution DAG
2. Worker Service executes phases based on project configuration
3. Release creation packages Kubernetes resource definitions as OCI image
4. Platform updates ReleasePointer files in GitOps repository
5. Argo CD Custom Management Plugin extracts resources from Release OCI image
6. Kubernetes and Crossplane apply resources to cluster

Applications interact with platform through:
- CUE configuration defining pipeline phases and deployment resources
- Crossplane XRDs for infrastructure abstraction
- Kubernetes resources for application-specific needs
- Platform API for pipeline status and release management

See Developer Guide: Deployment Configuration and Common Workflows for complete application integration patterns.

## Constraints

### Platform Constraints

- Single AWS region per installation
- Kubernetes required for all deployments
- Earthly required for build execution
- GitHub required for source control
- Monorepo-focused workflows (optimized for single repository with multiple projects)

### Scale Limits

- Recommended maximum 100 projects per repository
- Recommended maximum 1000 concurrent pipelines
- Hard limit 24 hours per pipeline execution
- Maximum 1MB discovery output size
- Maximum 50 parallel workers per Worker Service deployment

### Technical Constraints

- No cross-repository dependencies (projects cannot depend on other repositories)
- Sequential phase execution (phases run sequentially, not in parallel)
- No multi-project releases (each project releases independently)
- Environment policies global (cannot customize per-project)

### Security Boundaries

All authentication flows managed through Keycloak. Worker services access AWS resources via IRSA (IAM Roles for Service Accounts) without long-lived credentials. Secrets accessed through provider pattern, never stored in database or logs. Each pipeline run isolated via unique NATS subjects and namespaced S3 results.

Zero-trust networking via Istio Ambient Mode ensures all cluster traffic encrypted via mTLS. Service-to-service communication requires explicit authorization policies. Default deny model with allow-list policies.

## Future Considerations

### Planned Enhancements

**Multi-Region Support:**
Worker Service deployments in multiple regions for disaster recovery, region-aware artifact publishing, and cross-region release replication.

**Enhanced Discovery:**
Incremental discovery for large monorepos, cached discovery results, and predictive project loading to reduce pipeline initialization time.

**Extended Build Support:**
Alternative execution engines beyond Earthly, GPU-enabled workers for ML workloads, and custom worker resource configurations per project.

**Deployment Enhancements:**
Multi-project release groups for coordinated deployments, progressive deployment strategies (canary, blue-green), and dynamic environment creation for feature branches.

### Extension Points

The ports and adapters architecture provides clear extension points for future capabilities:
- Additional secret providers through provider interface
- Alternative build execution engines through consistent abstraction
- Custom Crossplane compositions for organization-specific patterns
- Additional deployment strategies through Argo CD integration

---

**Related Documents:**
- Platform Contracts - Infrastructure contract specifications and adapter implementations
- System Architecture - Service architecture, component interactions, and data flows
- Implementation Guide - Deployment profiles, bootstrapping, and infrastructure abstractions
- Developer Guide - Configuration patterns and application workflows

**Last Updated:** 2025-10-05
