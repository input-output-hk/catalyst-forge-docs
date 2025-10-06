# Glossary of Terms

## Table of Contents

- [A](#a)
- [B](#b)
- [C](#c)
- [D](#d)
- [E](#e)
- [F](#f)
- [G](#g)
- [I](#i)
- [J](#j)
- [K](#k)
- [M](#m)
- [N](#n)
- [O](#o)
- [P](#p)
- [R](#r)
- [S](#s)
- [T](#t)
- [W](#w)
- [X](#x)

---

This glossary provides definitions for key terms and concepts used throughout the Catalyst Forge platform documentation.

---

## A

### Adapter
A concrete implementation of a platform contract (port). Adapters fulfill contract requirements using specific technologies (e.g., S3 adapter for Object Storage Contract, MinIO adapter for Object Storage Contract). See Platform Contracts.

### Alias
Human-readable identifiers for releases, including numeric (auto-incrementing), tag (from Git tags), and branch aliases. See Domain Model: Release.

### Approval
Required authorization for deploying releases to protected environments (typically production). See Domain Model: ReleaseApproval.

### Argo CD
GitOps continuous delivery tool used for deploying releases to Kubernetes clusters. See System Architecture: GitOps Deployment Flow.

### Argo Workflows
Kubernetes-native workflow orchestration engine used for pipeline execution. See System Architecture: Component Architecture.

### Artifact
A build output (container image, binary, or archive) produced during pipeline execution. See Developer Guide: Artifact Configuration.

### Artifact OCI Tracking Image
An OCI image containing metadata, SBOM, and provenance for a built artifact. Format: `[tracking-registry]/[repository]/[project]/[artifact]:sha-[commit_sha]`. See System Architecture: OCI Image Formats.

### Authentication Contract
Platform contract defining identity verification and OIDC federation requirements. See Platform Contracts: Authentication Contract.

---

## B

### Base Image
The foundation container image used to build a derived container image. Tracked in artifact metadata.

### Branch Alias
Release alias derived from branch name, format: `{branch-name}-{release-number}`. Example: `main-42`.

### Bootstrapping
The five-phase process of deploying the platform: Prerequisites, GitOps Foundation, Infrastructure Abstraction Layer, Platform Services, and Environment Provisioning. See Implementation Guide: Bootstrapping Sequence.

---

## C

### Canonical Identifier
The immutable triplet `repository/project/commit-sha` that uniquely identifies releases and artifacts. See Architecture Overview: Canonical Identifier Pattern.

### Cluster Provisioning Contract
Platform contract defining Kubernetes cluster lifecycle management requirements. See Platform Contracts: Cluster Provisioning Contract.

### Composition
Crossplane resource that implements an XRD by creating concrete Kubernetes resources. See Implementation Guide: Infrastructure Abstractions.

### Container Registry Contract
Platform contract defining OCI-compliant image storage requirements. See Platform Contracts: Container Registry Contract.

### Contract
A platform port specification defining required capabilities, interface specifications, and behavioral requirements independent of implementation. See Platform Contracts.

### Control Plane
The platform cluster that hosts platform services and manages environment clusters. See Implementation Guide: Control Plane / Data Plane Separation.

### CronJob XRD
Crossplane composite resource for scheduled task execution. See Implementation Guide: Infrastructure Abstractions.

### Crossplane
Kubernetes extension for infrastructure abstraction through composite resources (XRDs). See Architecture Overview: Technology Stack.

### CUE
Configuration language used for type-safe, composable platform configuration. See Developer Guide: Configuration with CUE.

---

## D

### Data Plane
Environment clusters that execute application workloads. See Implementation Guide: Control Plane / Data Plane Separation.

### Database Contract
Platform contract defining relational database provisioning requirements. See Platform Contracts: Database Contract.

### Deployment
An instance of a release running in a specific environment. See Domain Model: Deployment.

### Deployment Profile
Configuration set defining adapter implementations for all platform contracts (Production/AWS or On-Premises). See Implementation Guide: Deployment Profiles.

### Deployment XRD
Crossplane composite resource for stateless service deployment. See Implementation Guide: Infrastructure Abstractions.

### Dispatcher
Ephemeral pods created by Argo Workflows that submit jobs to NATS JetStream and await replies. See System Architecture: Service Architecture.

### DNS Management Contract
Platform contract defining DNS record lifecycle management requirements. See Platform Contracts: DNS Management Contract.

---

## E

### Earthly
Build tool used for all pipeline step execution. See Developer Guide: Earthly Integration.

### Environment
A Kubernetes cluster running application workloads (dev, staging, production). See Architecture Overview: Core Concepts.

### Environment Cluster
A Kubernetes cluster that executes application workloads, managed by the platform cluster. See Implementation Guide: Environment Clusters.

### EnvironmentConfig
Crossplane resource providing environment-specific configuration values at deployment time. See Implementation Guide: Environment Configuration Model.

### Environment-Agnostic Release
Release containing no environment-specific values, enabling deployment to any environment through EnvironmentConfig. See Architecture Overview: Environment Agnosticism.

### External Secrets Operator
Kubernetes operator that synchronizes secrets from external providers (AWS Secrets Manager, Vault) to Kubernetes Secrets. See Platform Contracts: Secret Management Contract.

---

## F

### Fail-Fast
Pipeline execution model where any phase failure immediately terminates the pipeline. See Developer Guide: Pipeline Workflow.

---

## G

### Git SHA
The commit identifier used as the immutable version component in canonical identifiers. See Architecture Overview: Canonical Identifier Pattern.

### GitOps
Deployment model using Git as single source of truth for declarative infrastructure and applications. See System Architecture: GitOps Deployment Flow.

### GitOps Repository
Git repository managed by the platform containing ReleasePointer resources for all environments. See System Architecture: GitOps Repository Structure.

---

## I

### Immutability
Architectural principle ensuring releases cannot be modified once created, only superseded or redeployed. See Architecture Overview: Immutability.

### IRSA (IAM Roles for Service Accounts)
AWS mechanism for pod-level AWS service authentication without long-lived credentials (AWS profile only). See Platform Contracts: Authentication Contract.

---

## J

### Job XRD
Crossplane composite resource for one-time task execution. See Implementation Guide: Infrastructure Abstractions.

---

## K

### KEDA
Kubernetes Event-Driven Autoscaling, used for pod-level autoscaling based on metrics. See Platform Contracts: Workload Scaling Contract.

### Keycloak
Identity and access management system providing authentication for all platform access. See Architecture Overview: Technology Stack.

---

## M

### Message Queue Contract
Platform contract defining work queue and request-reply messaging requirements. See Platform Contracts: Message Queue Contract.

---

## N

### NATS JetStream
Messaging system providing ephemeral work queues and request-reply patterns for platform job distribution. See Architecture Overview: Technology Stack.

### Network Contract
Platform contract defining service exposure, DNS, and service mesh policy requirements. See Platform Contracts: Network Contract.

---

## O

### Object Storage Contract
Platform contract defining S3-compatible object storage requirements. See Platform Contracts: Object Storage Contract.

### Observability Contract
Platform contract defining metrics, logs, and traces collection requirements. See Platform Contracts: Observability Contract.

### OCI (Open Container Initiative)
Standards for container formats and registries. The platform uses OCI images for both artifacts and releases. See System Architecture: OCI Image Formats.

### On-Premises Profile
Deployment profile using self-hosted, open-source adapter implementations for all infrastructure contracts. See Implementation Guide: On-Premises Profile.

---

## P

### Phase
A sequential execution group in a pipeline, containing one or more tasks that execute in parallel. See Developer Guide: Pipeline Configuration.

### Phase Execution
Entity representing execution of a specific phase within a pipeline run. See Domain Model: PhaseExecution.

### Pipeline Component
Logical component responsible for project discovery, CI orchestration, and build execution. See Architecture Overview: Component Architecture.

### Pipeline Run
Entity representing execution of a complete pipeline for a commit. See Domain Model: PipelineRun.

### Platform API
REST API service handling authentication, entity management, and webhook ingestion. See System Architecture: Service Architecture.

### Platform Cluster
The persistent Kubernetes cluster hosting platform services and managing environment clusters. See Implementation Guide: Platform Cluster.

### Platform-Provided Contract
Infrastructure contract fulfilled by adapters deployed by the platform. See Platform Contracts: Contract Classification.

### Port
See Contract.

### Ports and Adapters Pattern
Architectural pattern separating interface specifications (ports/contracts) from implementations (adapters), enabling infrastructure portability. See Architecture Overview: Ports and Adapters Architecture.

### PostgreSQL
Relational database used for all platform persistent state and audit trails. See Architecture Overview: Technology Stack.

### Producer
Configuration defining how an artifact is created (e.g., Earthly target specification). See Developer Guide: Artifact Configuration.

### Production/AWS Profile
Deployment profile using AWS-managed services where appropriate and self-hosted services for platform-specific functionality. See Implementation Guide: Production/AWS Profile.

### Project
A deliverable unit within a repository, identified by a `.forge` directory. Projects are identified globally by `repository/project-name`. See Developer Guide: Repository and Project Structure.

### Publisher
Configuration defining where artifacts are distributed (Docker registries, GitHub Releases, PyPI, etc.). See Developer Guide: Publisher Configuration.

---

## R

### Reference Resolution
Pattern for accessing values from other XRDs using syntax like `outputs/<xr>/<key>`, `connections/<xr>/<key>`, `secrets/<xr>/<secret>/[key]`. See Implementation Guide: Reference Resolution Pattern.

### Release
Immutable snapshot of a project at a specific commit, including all artifacts and Kubernetes resource definitions. See Domain Model: Release.

### Release Component
Logical component responsible for creating releases and managing deployments. See Architecture Overview: Component Architecture.

### Release Number
Auto-incrementing numeric identifier for releases within a project. See Domain Model: Release.

### Release OCI Image
OCI image containing all Kubernetes resource definitions for a release. Format: `[release-registry]/[repository]/[project]:sha-[commit_sha]`. See System Architecture: OCI Image Formats.

### ReleasePointer
Kubernetes resource in the GitOps repository specifying which release is deployed in an environment. See System Architecture: GitOps Repository Structure.

### Repository
A GitHub repository containing one or more projects. Repository configuration defines platform-wide settings. See Developer Guide: Repository Setup.

---

## S

### Secret Management Contract
Platform contract defining secret storage and synchronization requirements. See Platform Contracts: Secret Management Contract.

### Self-Hosting
Platform deployment model where the platform manages itself through the same mechanisms it provides to applications. See Implementation Guide: Self-Hosting Model.

### StatefulSet
Kubernetes workload type used for Worker Service deployment, providing persistent storage and stable identities. See System Architecture: Service Architecture.

### Stateful XRD
Crossplane composite resource for services with persistent identity and storage. See Implementation Guide: Infrastructure Abstractions.

### Step Execution
Entity representing execution of a specific step within a task. See Domain Model: StepExecution.

---

## T

### Task Execution
Entity representing execution of a project's participation in a phase. See Domain Model: TaskExecution.

### Tracking OCI Image
See Artifact OCI Tracking Image.

---

## W

### Worker Service
StatefulSets executing asynchronous jobs via NATS JetStream pull consumers, maintaining persistent Git repository caches. See System Architecture: Service Architecture.

### Workload Scaling Contract
Platform contract defining pod and node autoscaling requirements. See Platform Contracts: Workload Scaling Contract.

---

## X

### XRD (Composite Resource Definition)
Crossplane abstraction defining infrastructure patterns (Deployment, Stateful, Network, Secrets, Database, etc.). See Implementation Guide: Infrastructure Abstractions.

---

**Last Updated:** 2025-10-05
