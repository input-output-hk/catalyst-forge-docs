# Glossary of Terms

This glossary provides definitions for key terms and concepts used throughout the Catalyst Forge platform documentation.

---

## A

### Alias
Human-readable identifiers for releases, including numeric (auto-incrementing), tag (from Git tags), and branch aliases. See [Release & Deployment: Release Identification & Aliasing](05-release-deployment.md#release-identification--aliasing).

### API Server
REST API component responsible for pipeline management, state persistence, and authentication. Part of the Pipeline Component. See [Execution & Orchestration: Pipeline Architecture](03-execution-orchestration.md#pipeline-architecture).

### Approval
Required authorization for deploying releases to protected environments (typically production). See [Domain Model & API Reference: ReleaseApproval](06-domain-model-api-reference.md#releaseapproval).

### Argo CD
GitOps continuous delivery tool used for deploying releases to Kubernetes clusters. See [Core Architecture: GitOps Repository Structure](01-core-architecture.md#gitops-repository-structure).

### Argo Workflows
Kubernetes-native workflow orchestration engine used for pipeline execution. See [Execution & Orchestration: Argo Workflows Engine](03-execution-orchestration.md#argo-workflows-engine).

### Artifact
A build output (container image, binary, or archive) produced during pipeline execution. See [Build & Distribution: Artifact Types & Definitions](04-build-distribution.md#artifact-types--definitions).

### Artifact OCI Tracking Image
An OCI image containing metadata, SBOM, and provenance for a built artifact. Format: `[tracking-registry]/[repository]/[project]/[artifact]:sha-[commit_sha]`. See [Core Architecture: OCI Image Formats](01-core-architecture.md#artifact-oci-tracking-images).

### Artifacts Component
Platform component responsible for building and publishing artifacts. See [Build & Distribution](04-build-distribution.md).

### Authentication Boundary
Point where authentication is required for system access. Platform supports GitHub OIDC, CLI (via Keycloak), worker pools (IRSA), and internal services (Keycloak service accounts). See [Core Architecture: Authentication & Authorization Model](01-core-architecture.md#authentication--authorization-model).

### AWS Secrets Manager
Secret storage service used as the initial secret provider for the platform. See [Core Architecture: Secret Management Patterns](01-core-architecture.md#secret-providers).

---

## B

### Base Image
The foundation container image used to build a derived container image. Tracked in artifact metadata.

### Branch Alias
Release alias derived from branch name, format: `{branch-name}-{release-number}`. Example: `main-42`.

---

## C

### Canonical Identifier
The authoritative unique identifier for a release: `{repository}/{project}/{commit-sha}`. See [Release & Deployment: Canonical Identity](05-release-deployment.md#canonical-identity).

### Canonical Triplet
The pattern `repository/project/commit-sha` used throughout the platform for global uniqueness. See [Core Architecture: Key Terms](01-core-architecture.md#key-terms).

### CI Phase
A named stage in the pipeline execution (e.g., test, build, release). Phases execute sequentially by group. See [Configuration & Discovery: Phase Definitions](02-configuration-discovery.md#repository-configuration-schema).

### CMP (Custom Management Plugin)
Argo CD plugin that extracts Kubernetes resources from Release OCI images. See [Core Architecture: GitOps Repository Structure](01-core-architecture.md#gitops-repository-structure).

### Commit SHA
The Git commit hash that uniquely identifies source code state. Used as canonical identifier for releases.

### Component
A major subsystem of the platform: Pipeline, Project, Artifacts, or Release & Deployment. See [Core Architecture: Component Architecture & Dependencies](01-core-architecture.md#component-architecture--dependencies).

### Composite Resource (XR)
A Crossplane abstraction that simplifies Kubernetes resource provisioning. Projects primarily use XRs for deployment. See [Core Architecture: Technology Stack](01-core-architecture.md#core-technologies).

### Configuration & Discovery
Platform component defining CUE schemas and project discovery mechanisms. See [Configuration & Discovery](02-configuration-discovery.md).

### Cosign
Tool used for signing OCI images to provide cryptographic attestation. Used for artifact tracking images.

### Crossplane
CNCF tool for resource composition and infrastructure provisioning. Provides XRs for abstracting Kubernetes complexity. See [Core Architecture: Technology Stack](01-core-architecture.md#core-technologies).

### CUE
Configuration language used for all platform configuration. Provides type-safety and validation. See [Configuration & Discovery: CUE Configuration System](02-configuration-discovery.md#cue-configuration-system).

---

## D

### DAG (Directed Acyclic Graph)
The execution graph constructed from phase and step configurations. Steps within a phase run in parallel unless prioritized.

### Deployment
The application of a release to a specific environment. Tracked as a database entity. See [Domain Model & API Reference: Deployment](06-domain-model-api-reference.md#deployment).

### Discovery
The process of traversing repository filesystem to find projects with `.forge` directories. See [Configuration & Discovery: Discovery Mechanisms & Rules](02-configuration-discovery.md#discovery-mechanisms--rules).

### Discovery Output
JSON structure containing repository config, discovered projects, and phase groups. Used to drive pipeline execution. See [Configuration & Discovery: Discovery Output](02-configuration-discovery.md#discovery-output).

### Discovery Worker Pool
Hot worker pool optimized for repository discovery with cached Git repositories. See [Execution & Orchestration: Hot Discovery Pool](03-execution-orchestration.md#hot-discovery-pool).

### Dispatcher
Lightweight pod that submits jobs to SQS and polls DynamoDB for completion. Part of Argo Workflows execution. See [Execution & Orchestration: Dispatcher Templates](03-execution-orchestration.md#dispatcher-templates).

### DLQ (Dead Letter Queue)
SQS queue for failed jobs after maximum retry attempts. Monitored via CloudWatch alarms.

### Domain Entity
A persistent business object in the platform (e.g., PipelineRun, Release, Deployment). See [Domain Model & API Reference: Domain Entity Catalog](06-domain-model-api-reference.md#domain-entity-catalog).

### DynamoDB
AWS NoSQL database used for real-time job status tracking with automatic TTL cleanup. See [Execution & Orchestration: DynamoDB Job Tracking](03-execution-orchestration.md#dynamodb-job-tracking).

---

## E

### Earthly
Container-based build tool used as the default artifact producer. Provides reproducible builds with caching. See [Build & Distribution: Earthly Producer](04-build-distribution.md#earthly-producer).

### Environment
A deployment target (dev, staging, production). Each has specific promotion policies. See [Release & Deployment: Environment Promotion](05-release-deployment.md#environment-promotion).

### Environment Agnostic
Design principle where releases contain no environment-specific configuration. EnvironmentConfigs provide values during deployment. See [Core Architecture: Environment Agnosticism](01-core-architecture.md#environment-agnosticism).

### EnvironmentConfig
Crossplane resource that provides environment-specific values to Composite Resources during deployment.

### Execution & Orchestration
Platform component managing pipeline execution via Argo Workflows and worker pools. See [Execution & Orchestration](03-execution-orchestration.md).

---

## F

### Fail-Fast
Execution principle where any failure immediately terminates the pipeline. See [Core Architecture: Sequential Phase Execution](01-core-architecture.md#sequential-phase-execution-with-fail-fast).

### Forge Version
Semantic version specified in `repo.cue` that applies to configuration schemas. Example: `v1.0.0`. See [Configuration & Discovery: Schema Versioning](02-configuration-discovery.md#schema-versioning--evolution).

---

## G

### Git Worktree
Lightweight checkout of a specific commit used by discovery workers for concurrent operations. See [Execution & Orchestration: Repository Caching](03-execution-orchestration.md#worker-execution-patterns).

### GitHub OIDC
OpenID Connect tokens from GitHub Actions, exchanged for Keycloak identity for API authentication. See [Core Architecture: Authentication Boundaries](01-core-architecture.md#authentication-boundaries).

### GitOps
Declarative deployment approach using Git as source of truth. Platform uses pointer files in GitOps repository. See [Core Architecture: GitOps Repository Structure](01-core-architecture.md#gitops-repository-structure).

### GitOps Repository
Central Git repository containing pointer files for all environments and projects. Structure: `environments/{env}/{repo}/{project}/release.yaml`.

### Grafana Cloud
Unified observability platform for metrics, logs, and dashboards. See [Execution & Orchestration: Monitoring & Observability](03-execution-orchestration.md#monitoring--observability).

---

## H

### Hot Worker Pool
Persistent pods with cached repositories and warm connections to eliminate cold start penalties. See [Execution & Orchestration: Worker Pool Architecture](03-execution-orchestration.md#worker-pool-architecture).

---

## I

### Idempotency
Property where repeated operations with same inputs produce same outputs. Release creation and artifact publishing are idempotent. See [Build & Distribution: Idempotent Operations](04-build-distribution.md#idempotent-operations).

### Immutable
Cannot be modified after creation. Releases are immutable snapshots. See [Core Architecture: Immutability](01-core-architecture.md#immutability).

### Integration Contract
Defined interface for component-to-component communication. See [Integration Contracts](07-integration-contracts.md).

### IRSA (IAM Roles for Service Accounts)
Kubernetes service accounts with AWS IAM roles for secure AWS service access without long-lived credentials. See [Core Architecture: Worker Pool Security](01-core-architecture.md#worker-pool-security).

---

## K

### Keycloak
Identity and access management system handling all platform authentication. All auth flows (GitHub OIDC, CLI, internal services) go through Keycloak. See [Core Architecture: Authentication & Authorization Model](01-core-architecture.md#authentication--authorization-model).

### KRD (Kubernetes Resource Definition)
YAML manifest defining Kubernetes resources. Generated from CUE during release creation. See [Core Architecture: Key Terms](01-core-architecture.md#key-terms).

---

## L

### Loki
Log aggregation system that streams logs to Grafana Cloud. See [Execution & Orchestration: Logging](03-execution-orchestration.md#logging).

---

## M

### Metadata Layer
Layer in OCI images containing structured metadata. Both artifact tracking and release images have metadata layers. See [Core Architecture: OCI Image Formats](01-core-architecture.md#oci-image-formats).

### Monorepo Strategy
Tagging strategy where tags follow `{project}/v*` pattern for individual project releases. See [Configuration & Discovery: Tagging Strategy](02-configuration-discovery.md#repository-configuration-schema).

---

## N

### Numeric Alias
Auto-incrementing release identifier per project. Example: Release 42. See [Release & Deployment: Alias System](05-release-deployment.md#alias-system).

---

## O

### OCI (Open Container Initiative)
Standard for container images and registries. Platform uses OCI format for both artifact tracking and release packaging. See [Core Architecture: OCI Image Formats](01-core-architecture.md#oci-image-formats).

### OCI Registry
Storage for OCI images. Platform uses separate registries for artifact tracking and releases. See [Integration Contracts: OCI Registries](07-integration-contracts.md#oci-registries).

---

## P

### Phase
Named stage in pipeline execution (e.g., test, build). Phases are grouped and execute sequentially by group. See [Configuration & Discovery: Phase Definitions](02-configuration-discovery.md#repository-configuration-schema).

### Phase Group
Numeric grouping that determines execution order. All phases in same group run in parallel. See [Execution & Orchestration: Phase & Step Execution](03-execution-orchestration.md#phase--step-execution).

### PhaseExecution
Domain entity tracking execution of a single phase within a pipeline run. See [Domain Model & API Reference: PhaseExecution](06-domain-model-api-reference.md#phaseexecution).

### Pipeline Component
Platform component orchestrating all other components and managing execution. See [Core Architecture: Component Architecture](01-core-architecture.md#pipeline-component).

### Pipeline Run
Single execution of the CI/CD pipeline for a commit. See [Domain Model & API Reference: PipelineRun](06-domain-model-api-reference.md#pipelinerun).

### Pointer File
ReleasePointer resource in GitOps repository that references a release by commit SHA. See [Core Architecture: GitOps Repository Structure](01-core-architecture.md#gitops-repository-structure).

### Ports and Adapters
Architectural pattern enabling technology swapping without changing core workflows. Used for secret providers. See [Core Architecture: Built for Change](01-core-architecture.md#built-for-change-not-forever).

### PostgreSQL
Relational database for persistent storage of audit trails and queryable state. See [Domain Model & API Reference: Database Technology](06-domain-model-api-reference.md#database-technology).

### Producer
Component that builds artifacts from source code. Initial implementation: Earthly. See [Build & Distribution: Producer Contract](04-build-distribution.md#producer-contract--implementations).

### Progressive Disclosure
Design principle supporting both quick reference and deep dives. See [Core Architecture: Progressive Disclosure of Complexity](01-core-architecture.md#progressive-disclosure-of-complexity).

### Project
A deliverable unit marked by a `.forge` folder. Unique within repository, not globally. See [Core Architecture: Key Terms](01-core-architecture.md#key-terms).

### Project Component
Platform component defining configuration schema and discovery mechanisms. See [Core Architecture: Component Architecture](01-core-architecture.md#project-component).

### Prometheus
Metrics collection system that scrapes platform components and pushes to Grafana Cloud. See [Execution & Orchestration: Metrics Collection](03-execution-orchestration.md#metrics-collection).

### Provenance
Traceable record of artifact origin, build process, and materials. Stored in artifact tracking OCI images. See [Build & Distribution: Universal Provenance](04-build-distribution.md#universal-provenance).

### Publisher
Component that distributes built artifacts to registries (Docker, PyPI, GitHub, etc.). See [Build & Distribution: Publisher Contract](04-build-distribution.md#publisher-contract--registry-types).

---

## R

### RBAC (Role-Based Access Control)
Authorization model using roles to control permissions. Managed in Keycloak for all platform access. See [Core Architecture: Authorization Model](01-core-architecture.md#authorization-model).

### Release
Immutable snapshot of a project at a specific commit, including KRDs and artifact references. See [Domain Model & API Reference: Release](06-domain-model-api-reference.md#release).

### Release & Deployment Component
Platform component managing release creation and environment progression. See [Release & Deployment](05-release-deployment.md).

### Release OCI Image
OCI image containing release metadata and rendered Kubernetes resources. Format: `[release-registry]/[repository]/[project]:[commit-sha]`. See [Core Architecture: OCI Image Formats](01-core-architecture.md#release-oci-images).

### ReleasePointer
Kubernetes custom resource in GitOps repository pointing to a release by commit SHA. See [Core Architecture: GitOps Repository Structure](01-core-architecture.md#gitops-repository-structure).

### Release Trigger
Condition that initiates release creation (branch push, tag creation, manual). See [Release & Deployment: Trigger Evaluation](05-release-deployment.md#trigger-evaluation).

### Repository
A GitHub repository containing one or more projects. Part of the canonical triplet. See [Core Architecture: Key Terms](01-core-architecture.md#key-terms).

### Resources Layer
Layer in Release OCI images containing rendered Kubernetes YAML as tar archive. See [Core Architecture: OCI Image Formats](01-core-architecture.md#release-oci-images).

### Rollback
Forward deployment of a previous release to an environment. Implemented as redeployment, not true rollback. See [Release & Deployment: Rollback Operations](05-release-deployment.md#rollback-operations).

---

## S

### S3 (Simple Storage Service)
AWS object storage for discovery outputs, logs, and artifact staging. See [Execution & Orchestration: S3 Storage](03-execution-orchestration.md#s3-storage).

### SBOM (Software Bill of Materials)
List of software components in an artifact. Generated automatically using Syft. See [Build & Distribution: Artifact Tracking OCI Format](04-build-distribution.md#artifact-tracking-oci-format).

### Secret Provider
Pluggable backend for secret resolution. Initial release: AWS Secrets Manager. See [Core Architecture: Secret Providers](01-core-architecture.md#secret-providers).

### Secret Reference
Configuration pointer to a secret, never containing the actual value. Uses ports and adapters pattern. See [Core Architecture: Secret Management Patterns](01-core-architecture.md#secret-management-patterns).

### Sequential Phase Execution
Execution model where phases run one group at a time, with fail-fast on errors. See [Core Architecture: Sequential Phase Execution](01-core-architecture.md#sequential-phase-execution-with-fail-fast).

### Service Account (Keycloak)
Machine identity for internal service-to-service authentication managed by Keycloak. See [Core Architecture: Internal Services](01-core-architecture.md#authentication-boundaries).

### Single Source of Truth
Principle where every concept has exactly one authoritative definition. See [Core Architecture: Single Source of Truth](01-core-architecture.md#single-source-of-truth).

### SQS (Simple Queue Service)
AWS message queue for distributing jobs to worker pools. See [Execution & Orchestration: SQS Configuration](03-execution-orchestration.md#sqs-configuration).

### Staging Directory
Temporary filesystem location where producers output artifacts before publishing. Cleaned up after release.

### Step
Individual task within a phase (e.g., running a specific Earthly target). See [Configuration & Discovery: CI Configuration](02-configuration-discovery.md#project-configuration-schema).

### StepExecution
Domain entity tracking execution of a single step within a task. See [Domain Model & API Reference: StepExecution](06-domain-model-api-reference.md#stepexecution).

### Syft
Tool for generating SBOMs from artifacts. Used automatically by platform.

---

## T

### Tag Alias
Release alias derived from Git tag. Format depends on repository tagging strategy. See [Release & Deployment: Alias System](05-release-deployment.md#alias-system).

### Tag-All Strategy
Tagging strategy where any version tag triggers releases for all projects. See [Configuration & Discovery: Tagging Strategy](02-configuration-discovery.md#repository-configuration-schema).

### Task Pool
Hot worker pool executing Earthly targets with pre-established connections. See [Execution & Orchestration: Hot Task Pool](03-execution-orchestration.md#hot-task-pool).

### TaskExecution
Domain entity tracking execution of all steps for a project within a phase. See [Domain Model & API Reference: TaskExecution](06-domain-model-api-reference.md#taskexecution).

### Tracking Image
See **Artifact OCI Tracking Image**.

### TTL (Time To Live)
Automatic expiration for DynamoDB records (7 days). Enables automatic cleanup of job status.

---

## W

### Worker Pool
Fleet of persistent pods that execute platform work. Two types: Discovery and Task. See [Execution & Orchestration: Worker Pool Architecture](03-execution-orchestration.md#worker-pool-architecture).

### Workflow
Argo Workflows execution instance representing a pipeline run. See [Execution & Orchestration: Workflow Structure](03-execution-orchestration.md#workflow-structure).

---

## X

### XR (Composite Resource)
Crossplane Composite Resource - platform abstraction for Kubernetes resources. See [Core Architecture: Key Terms](01-core-architecture.md#key-terms).

### XRD (Composite Resource Definition)
Schema defining a Crossplane Composite Resource type. Platform provides XRDs for common patterns.

---

## Acronyms Quick Reference

| Acronym | Full Term | Definition |
|---------|-----------|------------|
| API | Application Programming Interface | REST interface for platform services |
| AWS | Amazon Web Services | Cloud provider for platform infrastructure |
| CD | Continuous Delivery | Automated deployment process |
| CI | Continuous Integration | Automated build and test process |
| CMP | Custom Management Plugin | Argo CD plugin for Release OCI extraction |
| CNCF | Cloud Native Computing Foundation | Open source foundation (Crossplane, Argo) |
| CRD | Custom Resource Definition | Kubernetes extension mechanism |
| DAG | Directed Acyclic Graph | Execution dependency graph |
| DLQ | Dead Letter Queue | Failed message queue |
| IAM | Identity and Access Management | AWS authentication service |
| IRSA | IAM Roles for Service Accounts | Kubernetes-AWS integration |
| KRD | Kubernetes Resource Definition | YAML manifest |
| OCI | Open Container Initiative | Container image standard |
| OIDC | OpenID Connect | Authentication protocol |
| RBAC | Role-Based Access Control | Authorization model |
| S3 | Simple Storage Service | AWS object storage |
| SBOM | Software Bill of Materials | Component inventory |
| SCM | Source Control Management | Git/GitHub |
| SQS | Simple Queue Service | AWS message queue |
| SSO | Single Sign-On | Unified authentication |
| TTL | Time To Live | Expiration period |
| URI | Uniform Resource Identifier | Resource address |
| XR | Composite Resource | Crossplane abstraction |
| XRD | Composite Resource Definition | Crossplane schema |

---

**Last Updated**: 2025-10-02
