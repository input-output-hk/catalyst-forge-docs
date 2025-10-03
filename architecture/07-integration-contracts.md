# Integration Contracts Document

## Executive Summary

This document defines all integration points and contracts between components in the Catalyst Forge platform. It serves as the authoritative reference for component interactions, shared infrastructure usage, and error propagation patterns.

## Core Concepts

### Integration Philosophy

#### Clear Boundaries

Components interact through well-defined contracts that specify inputs, outputs, and behavior. Internal implementation details remain encapsulated, allowing components to evolve independently as long as they maintain their external contracts.

#### Dependency Direction

The platform follows a layered dependency structure:
- Pipeline Component orchestrates all other components
- Project Component provides configuration to all other components
- Artifacts Component operates independently during release phase
- Release Component coordinates Artifacts and Project components
- No circular dependencies exist between components

#### Contract Versioning

All API contracts include version identifiers to enable evolution without breaking existing integrations. Version changes follow semantic versioning principles: major versions for breaking changes, minor versions for backward-compatible additions.

## Component Communication Patterns

### API-First Architecture

All component interactions flow through REST APIs with authentication and audit logging. This provides:
- **Security boundary**: All requests authenticated and authorized
- **Persistence layer**: API interactions recorded for audit trail
- **Version control**: API versioning enables contract evolution
- **Observability**: Centralized logging and metrics for all interactions

### Authentication Patterns

For complete authentication specifications, see [Core Architecture: Authentication & Authorization Model](01-core-architecture.md#authentication--authorization-model).

**External API Calls** (GitHub Actions → Pipeline API):
- GitHub OIDC tokens exchanged for Keycloak identity
- Repository membership validated through Keycloak
- Time-limited tokens (default 60 minutes)

**CLI Operations** (Developers → Platform APIs):
- Authentication via Keycloak (SSO, OAuth, etc.)
- Role-based access control managed in Keycloak
- Operations scoped by user permissions

**Internal Service Communication** (Component → Component):
- Keycloak service account tokens for machine-to-machine auth
- Service-specific RBAC policies in Keycloak
- Short-lived tokens with minimal scope

**Worker Pools** (Workers → AWS Services):
- IAM Roles via IRSA (no long-lived credentials)
- Least-privilege policies per worker type
- Kubernetes service account authentication

### Synchronous vs Asynchronous Patterns

**Synchronous Interactions**:
- API calls that require immediate response (e.g., release metadata lookup)
- Configuration reads during discovery
- Status queries for user-facing operations

**Asynchronous Interactions**:
- Pipeline execution (fire-and-forget with status updates)
- Worker pool job processing via SQS
- GitOps deployments triggered by pointer file updates
- GitHub status updates (non-blocking)

### Data Flow Patterns

**Configuration Flow** (Read-Only):
```
Project Component (CUE definitions)
    → Pipeline Component (discovery reads)
    → Artifacts Component (artifact configs)
    → Release Component (deployment resources)
```

**Execution Flow** (Write Path):
```
GitHub Actions
    → Pipeline API (create run)
    → Argo Workflows (orchestration)
    → Worker Pools (execution via SQS)
    → AWS Services (S3, DynamoDB status tracking)
    → Pipeline API (status updates)
    → GitHub (commit status)
```

**Artifact Flow** (Release Phase):
```
Release Orchestrator
    → Artifacts Component (build & publish)
    → Producers (Earthly creates artifacts)
    → Publishers (distribute to registries)
    → OCI Tracking Image (provenance)
    → Release Component (artifact references)
```

**Deployment Flow** (GitOps):
```
Release Component
    → GitOps Repository (pointer file update)
    → Argo CD (detects change)
    → Custom Management Plugin (fetch Release OCI)
    → Kubernetes (apply resources)
    → Crossplane (reconcile XRs)
```

## Architecture

### Inter-Component Contracts

#### Pipeline → Project

**Contract**: Discovery and Configuration Reading

**Interaction**: Pipeline Component reads project configurations during discovery phase

**Data Flow**:
```
Input (Pipeline provides):
- Repository path (local checkout)
- Commit SHA
- Git context (branch, tags, ref)

Output (Project provides):
- Repository configuration (phases, publishers, forge version)
- List of discovered projects
- Project configurations (phases, artifacts, releases, deployments)
- Phase groups for sequential execution
- Release-triggered projects
```

**Integration Point**: File system reads of `.forge` directories and CUE parsing

**Error Handling**: Configuration errors fail discovery immediately, terminating pipeline

**Reference**: [Configuration & Discovery: Discovery Mechanisms](02-configuration-discovery.md#discovery-mechanisms)

#### Pipeline → Artifacts

**Contract**: Artifact Building During Release Phase

**Interaction**: Pipeline orchestrates artifact creation when release is triggered

**Data Flow**:
```
Input (Pipeline provides to Release Orchestrator):
- Project configuration
- Repository checkout path
- Git context
- Staging directory

Pipeline triggers Release Orchestrator which coordinates with Artifacts:

Input (Release Orchestrator provides to Artifacts):
- Artifact configurations from project
- Repository-level publisher definitions
- Git context (commit SHA, tags, branch)
- Staging paths for outputs

Output (Artifacts returns):
- Built artifact metadata (per-artifact type structure)
- Publisher results (URIs and metadata per publisher)
- OCI tracking image references
- Build metadata (SBOMs, attestations)
```

**Integration Point**: Release Component invokes Artifacts Component APIs during release phase

**Error Handling**:
- Producer failures terminate release immediately
- Publisher failures mark release as partial
- Retries rebuild all artifacts (no caching across retries)

**Reference**: [Build & Distribution: Producer/Publisher Interaction Model](04-build-distribution.md#producerpublisher-interaction-model)

#### Pipeline → Release

**Contract**: Release Creation and Deployment

**Interaction**: Pipeline triggers release creation when conditions are met

**Data Flow**:
```
Input (Pipeline provides):
- Release trigger context
- Project configuration
- Repository checkout
- Git metadata
- Artifact build results (from Artifacts Component)

Output (Release returns):
- Release identifier (canonical triplet)
- Release OCI image URI
- Generated release aliases (numeric, tag, branch)
- Deployment status (if auto-deploy enabled)
```

**Integration Point**: Pipeline calls Release API during release phase workflow step

**Error Handling**:
- Release creation failures terminate pipeline as failed
- Deployment failures are logged but don't fail release creation
- Idempotent - duplicate requests return existing release

**Reference**: [Release & Deployment: Release Creation Workflow](05-release-deployment.md#release-creation-workflow)

#### Artifacts → Project

**Contract**: Artifact Configuration Schema

**Interaction**: Artifacts Component reads configuration defined by Project Component

**Data Flow**:
```
Input (Artifacts reads from Project config):
- Artifact definitions (name, type)
- Producer configuration (e.g., Earthly targets)
- Publisher references (names from repository config)
- Publisher overrides (project-specific settings)

Output (Project config provides):
- Type-safe artifact specifications
- Validated producer configurations
- Publisher credential references
```

**Integration Point**: CUE schema validation and parsing

**Error Handling**: Configuration errors detected during discovery, preventing release phase execution

**Reference**: [Configuration & Discovery: Project Configuration Schema](02-configuration-discovery.md#project-configuration-schema)

#### Release → Artifacts

**Contract**: Artifact Reference Resolution

**Interaction**: Release Component resolves artifact references in deployment resources

**Data Flow**:
```
Input (Release receives from Artifacts):
- Artifact output schemas (producer fields + publisher fields)
- OCI tracking image URIs
- Published artifact URIs
- Artifact metadata (digests, sizes, etc.)

Output (Release uses in rendering):
- Resolved @artifact() references in CUE
- Rendered Kubernetes resources with artifact URIs
- Metadata layer references to tracking images
```

**Integration Point**: CUE attribute resolution during resource rendering

**Error Handling**: Missing artifact references cause release creation to fail

**Reference**: [Release & Deployment: Artifact Coordination](05-release-deployment.md#artifact-coordination)

#### Release → Project

**Contract**: Deployment Resource Rendering

**Interaction**: Release Component renders Kubernetes resources from Project CUE definitions

**Data Flow**:
```
Input (Release reads from Project config):
- Deployment resource definitions (CUE)
- Artifact references (@artifact attributes)
- Custom deployment configurations

Output (Release generates):
- Rendered Kubernetes YAML manifests
- Resolved artifact references
- Environment-agnostic resource definitions
```

**Integration Point**: CUE evaluation and YAML rendering

**Error Handling**: Rendering errors fail release creation with detailed CUE evaluation errors

**Reference**: [Release & Deployment: Resource Rendering](05-release-deployment.md#resource-rendering)

#### Release → GitOps

**Contract**: Pointer File Management

**Interaction**: Release Component updates pointer files to trigger deployments

**Data Flow**:
```
Input (Release provides):
- Release identifier (commit SHA)
- Release aliases (numeric, tag, branch)
- Environment target
- Deployment metadata (deployer, approver, timestamps)

Output (GitOps repository contains):
- ReleasePointer resources in environment directories
- Git commits with deployment audit trail
- History of all deployments per project per environment
```

**Integration Point**: Git commits to GitOps repository

**Error Handling**: Git push failures retry with exponential backoff; persistent failures mark deployment as failed

**Reference**: [Core Architecture: GitOps Repository Structure](01-core-architecture.md#gitops-repository-structure)

#### Argo CD → Release

**Contract**: OCI Image Extraction

**Interaction**: Argo CD Custom Management Plugin fetches and extracts Release OCI images

**Data Flow**:
```
Input (Argo CD provides to plugin):
- ReleasePointer resource from Git
- Release commit SHA from pointer spec

Output (Plugin provides to Argo CD):
- Kubernetes YAML manifests (extracted from resources layer)
- Ready-to-apply resource definitions
```

**Integration Point**: Argo CD Custom Management Plugin

**Error Handling**: OCI fetch failures surface in Argo CD UI; plugin retries with backoff

**Reference**: [Core Architecture: GitOps Repository Structure](01-core-architecture.md#gitops-repository-structure)

### Shared Infrastructure Usage

#### AWS Services

##### SQS (Simple Queue Service)

**Purpose**: Job distribution to worker pools with reliable delivery

**Consumers**:
- Pipeline Component (creates queues, submits discovery and task jobs)
- Hot Worker Pools (consume jobs with long polling)

**Queue Configuration**:
```yaml
discovery-queue:
  name: pipeline-discovery
  visibilityTimeout: 300  # 5 minutes
  messageRetention: 3600  # 1 hour
  maxReceiveCount: 3      # DLQ after 3 failures

task-queue:
  name: pipeline-tasks
  visibilityTimeout: 1800  # 30 minutes
  messageRetention: 7200   # 2 hours
  maxReceiveCount: 3
```

**Message Format**:
```json
{
  "job_id": "string",
  "run_id": "string",
  "type": "discovery|task|artifact",
  "payload": {
    "repository": "string",
    "commit_sha": "string",
    "project": "string",
    "phase": "string",
    "steps": ["string"]
  },
  "result_path": "s3://bucket/path/to/result.json"
}
```

**Error Handling**: Failed jobs move to DLQ after max receive count; CloudWatch alarms monitor DLQ depth

**Authentication**: Workers use IRSA to access queues

**Reference**: [Execution & Orchestration: SQS Configuration](03-execution-orchestration.md#sqs-configuration)

##### DynamoDB

**Purpose**: Real-time job status tracking with automatic cleanup

**Consumers**:
- Pipeline Component (creates table, queries job status)
- Worker Pools (update job status)
- Dispatcher Pods (poll for job completion)

**Schema**:
```
Table: pipeline-jobs
Primary Key: job_id (String)

Attributes:
- job_id: string
- run_id: string
- type: enum (discovery, task, artifact)
- status: enum (pending, running, completed, failed)
- result_location: string (S3 URI)
- created_at: timestamp
- updated_at: timestamp
- ttl: timestamp (auto-cleanup after 7 days)
```

**Access Patterns**:
- Workers: Write status updates
- Dispatchers: Poll by job_id for completion
- API: Query by run_id for aggregated status

**Consistency Model**: Eventual consistency acceptable (status updates tolerate slight delay)

**Authentication**: IRSA for all access

**Reference**: [Execution & Orchestration: DynamoDB Job Tracking](03-execution-orchestration.md#dynamodb-job-tracking)

##### S3 (Simple Storage Service)

**Purpose**: Storage for discovery outputs, logs, and build artifacts

**Consumers**:
- Pipeline Component (stores discovery results, logs)
- Worker Pools (write outputs, stream logs)
- Artifacts Component (stages artifacts before publishing)
- All components (read logs via presigned URLs)

**Bucket Structure**:
```
pipeline-results/
  {run_id}/
    discovery.json
    phases/
      {phase}/
        {project}/
          {step}.json

pipeline-logs/
  {run_id}/
    {phase}/
      {project}/
        {step}.log

artifact-staging/
  {release_id}/
    {project}/
      {artifact}/
        [files]
```

**Lifecycle Policies**:
- Results: 7 days
- Logs: 90 days
- Staging: 1 day (cleanup after release)

**Authentication**: IRSA for all access

**Reference**: [Execution & Orchestration: S3 Storage](03-execution-orchestration.md#s3-storage)

##### Secrets Manager

**Purpose**: Secure secret storage with rotation and audit logging

**Consumers**:
- Artifacts Component (retrieves publisher credentials)
- Worker Pools (accesses secrets during execution)

**Secret Organization**:
```
ci/
  github-ssh-key
  earthly-token

publishers/
  docker/ecr-credentials
  pypi/token
  github/release-token
```

**Access Pattern**:
- Secrets never resolved until point of use
- Components receive only secret references
- Resolution happens in worker pods with IRSA
- Secrets exist in memory only during active use

**Authentication**: IRSA policies scope access by prefix

**Reference**: [Core Architecture: Secret Management Patterns](01-core-architecture.md#secret-management-patterns)

#### Kubernetes Resources

##### Argo Workflows

**Purpose**: Workflow orchestration for pipeline execution

**Consumers**:
- Pipeline Component (submits workflows, queries status)

**Resource Types**:
- Workflow CRD instances (one per pipeline run)
- Workflow Templates (reusable step definitions)
- Cron Workflows (not currently used)

**Namespace**: `catalyst-forge-pipelines`

**Service Account**: `pipeline-workflow-runner` (with minimal workflow execution permissions)

**Integration**: Pipeline API submits workflows via Argo Server REST API

**Reference**: [Execution & Orchestration: Argo Workflows Engine](03-execution-orchestration.md#argo-workflows-engine)

##### Persistent Volume Claims

**Purpose**: Shared cache storage for worker pools

**Consumers**:
- Discovery Worker Pools (Git repository cache)
- Task Worker Pools (Earthly build cache)

**Configuration**:
```yaml
discovery-cache:
  storageClass: gp3
  size: 100Gi
  accessMode: ReadWriteMany
  mountPath: /cache/repos

task-cache:
  storageClass: gp3
  size: 500Gi
  accessMode: ReadWriteMany
  mountPath: /cache/earthly
```

**Management**: Kubernetes handles provisioning and attachment

**Reference**: [Execution & Orchestration: Worker Pool Architecture](03-execution-orchestration.md#worker-pool-architecture)

##### ConfigMaps and Secrets

**Purpose**: Component configuration and Kubernetes-native secrets

**Consumers**: All platform components for configuration

**Usage**:
- Component configuration (non-sensitive)
- TLS certificates for internal services
- Argo CD plugin configuration
- NOT used for user secrets (AWS Secrets Manager preferred)

**Reference**: Component-specific deployment configurations

#### OCI Registries

##### Artifact Tracking Registry

**Purpose**: Storage for artifact OCI tracking images

**Consumers**:
- Artifacts Component (pushes tracking images)
- Release Component (references tracking images in metadata)
- Security tooling (scans SBOMs and attestations)

**Format**: `[tracking-registry]/[repository]/[project]/[artifact]:sha-[commit_sha]`

**Authentication**: IRSA for workers, read-only for security tools

**Reference**: [Core Architecture: OCI Image Formats](01-core-architecture.md#artifact-oci-tracking-images)

##### Release Registry

**Purpose**: Storage for Release OCI images

**Consumers**:
- Release Component (pushes release images)
- Argo CD Custom Management Plugin (pulls release images)

**Format**: `[release-registry]/[repository]/[project]:[commit-sha]`

**Authentication**: IRSA for platform, Argo CD service account for reads

**Reference**: [Core Architecture: OCI Image Formats](01-core-architecture.md#release-oci-images)

#### PostgreSQL Database

**Purpose**: Persistent storage for audit trails and queryable state

**Consumers**: All platform components via API layer

**Schema Ownership**:
- Pipeline Component: PipelineRun, PhaseExecution, TaskExecution, StepExecution
- Release Component: Release, ReleaseAlias, ReleaseTrigger, Deployment, ReleaseApproval
- Artifacts Component: ReleaseArtifact
- Shared: Repository, Project (managed by Project Component)

**Access Pattern**: All database access through API layer (no direct component access)

**Reference**: [Domain Model & API Reference](06-domain-model-api-reference.md)

#### Observability Stack

##### Grafana Cloud

**Purpose**: Unified metrics, logs, and dashboards

**Data Sources**:
- Prometheus metrics (scraped from platform components)
- Loki logs (streamed via promtail)
- CloudWatch metrics (polled via integration)

**Consumers**: All platform components export metrics and logs

**Integration**:
- Metrics: Prometheus `/metrics` endpoints scraped every 15s
- Logs: Promtail daemonset tails container logs
- CloudWatch: Grafana Cloud polls AWS metrics every 60s

**Reference**: [Execution & Orchestration: Monitoring & Observability](03-execution-orchestration.md#monitoring--observability)

### Error Propagation Patterns

#### Fail-Fast Principles

The platform enforces fail-fast behavior at all levels:

**Phase Execution**: Any step failure terminates the entire phase immediately. No parallel steps continue execution after first failure.

**Pipeline Execution**: Any phase failure terminates the entire pipeline. Subsequent phases never execute.

**Release Creation**: Any artifact build failure terminates release creation. Partial releases are not persisted.

**Publisher Failures**: Publisher failures mark the release as partial but don't prevent release creation. This allows retries of failed publishers without rebuilding artifacts.

#### Error Context Propagation

All errors include rich context for debugging:

```go
type PlatformError struct {
    Code        string                 // Error code (e.g., "PRODUCER_FAILED")
    Message     string                 // Human-readable message
    Component   string                 // Component that generated error
    Context     map[string]interface{} // Additional context
    Timestamp   time.Time              // When error occurred
    TraceID     string                 // Distributed trace identifier
    Cause       error                  // Underlying error (if any)
}
```

**Error Chain**: Errors propagate with full context:
```
Worker: Earthly target failed
  → Artifacts: Producer execution failed
    → Release: Artifact building failed
      → Pipeline: Release phase failed
        → GitHub: Commit status set to "failure"
```

#### Component-Specific Error Handling

##### Pipeline Component

**Discovery Errors**:
- Configuration parsing errors → Terminate pipeline with detailed CUE error
- Missing `.forge` directories → Successful pipeline with zero projects
- Invalid forge version → Terminate pipeline with version mismatch error

**Execution Errors**:
- Worker pool unavailable → Retry with exponential backoff (max 3 attempts)
- Step timeout → Terminate phase immediately
- Network failures → Retry transient errors, fail persistent issues

**Status Update Errors**:
- GitHub API failures → Log error but continue execution (non-blocking)
- Database write failures → Retry with backoff, mark pipeline as errored if persistent

**Reference**: [Execution & Orchestration: Error Handling](03-execution-orchestration.md#error-handling)

##### Artifacts Component

**Producer Errors**:
- Build failures → Include full build log in error context
- Invalid configuration → Fail during validation before execution
- Timeout → Configurable per producer type, defaults to 30 minutes

**Publisher Errors**:
- Authentication failures → Fail immediately with credential resolution details
- Network failures → Retry with exponential backoff (max 5 attempts)
- Registry errors → Include registry response in error context
- Idempotency check failures → Log and continue if artifact already exists

**Tracking Image Errors**:
- SBOM generation failures → Log warning but proceed (non-blocking)
- Signing failures → Fail release creation (cryptographic integrity required)
- Push failures → Retry with backoff, fail release after max attempts

**Reference**: [Build & Distribution: Error Handling & Retry Logic](04-build-distribution.md#error-handling--retry-logic)

##### Release Component

**Resource Rendering Errors**:
- CUE evaluation failures → Include full CUE error with line numbers
- Missing artifact references → Fail with clear message about missing artifacts
- Invalid Kubernetes resources → Fail with validation errors

**OCI Packaging Errors**:
- Layer creation failures → Fail with filesystem or serialization details
- Manifest generation failures → Fail with schema validation errors
- Registry push failures → Retry with backoff, fail after max attempts

**Deployment Errors**:
- Policy validation failures → Fail with policy requirements and actual values
- GitOps push failures → Retry with exponential backoff
- Argo CD sync failures → Visible in Argo CD UI, don't fail deployment record

**Reference**: [Release & Deployment: Error Handling](05-release-deployment.md#error-handling)

#### Retry Strategies

##### Transient vs Persistent Failures

**Transient** (automatically retried):
- Network timeouts
- Service temporarily unavailable (5xx)
- Rate limiting (429)
- SQS message visibility timeout

**Persistent** (fail immediately):
- Authentication failures (401, 403)
- Invalid configuration (400)
- Resource not found (404)
- Business logic errors

##### Backoff Algorithm

All retries use exponential backoff with jitter:
```
delay = min(max_delay, base_delay * 2^attempt) + random(0, jitter)

Default values:
- base_delay: 1s
- max_delay: 60s
- jitter: 1s
- max_attempts: 5
```

##### Idempotency Requirements

All retryable operations must be idempotent:
- Pipeline creation: Same commit SHA returns existing run
- Release creation: Same project/commit returns existing release
- Artifact publishing: Check for existing artifacts before pushing
- Deployment: Same release to same environment updates pointer file

#### Error Recovery Mechanisms

##### Pipeline Failures

**Automatic**: Workers retry transient failures within job visibility timeout

**Manual**: Trigger new pipeline run for same commit SHA

**State**: Failed runs remain in database as audit trail

##### Release Failures

**Automatic**: Artifact publisher retries within release attempt

**Manual**: Explicit retry command rebuilds artifacts and attempts all publishers

**State**: Failed releases marked with error details and retry history

##### Deployment Failures

**Automatic**: GitOps push retries automatically

**Manual**: Retry deployment with same release identifier

**Rollback**: Deploy previous release identifier to same environment

### Event Flow Diagrams

#### Complete Pipeline Flow

```
┌─────────────────┐
│  GitHub Actions │
│  (Push/PR)      │
└────────┬────────┘
         │
         │ POST /v1/runs
         │ (OIDC token)
         ▼
┌────────────────────┐
│  Pipeline API      │◄─────────────────────────┐
│  (Keycloak auth)   │                          │
└─────────┬──────────┘                          │
          │                                     │
          │ Submit Workflow                     │
          ▼                                     │
┌────────────────────┐                          │
│  Argo Workflows    │                          │
│  (Orchestration)   │                          │
└─────────┬──────────┘                          │
          │                                     │
          │ Discovery Dispatcher                │
          ▼                                     │
┌────────────────────┐                          │
│  SQS               │                          │
│  (Job Queue)       │                          │
└─────────┬──────────┘                          │
          │                                     │
          │ Poll                                │
          ▼                                     │
┌────────────────────┐                          │
│  Discovery Worker  │                          │
│  (Git + CUE)       │                          │
└─────────┬──────────┘                          │
          │                                     │
          │ Write result                        │
          ▼                                     │
┌────────────────────┐                          │
│  S3 + DynamoDB     │                          │
│  (Results)         │                          │
└─────────┬──────────┘                          │
          │                                     │
          │ Poll completion ────────────────────┘
          ▼
    [Dispatcher retrieves result]
          │
          │ Phase Loop (sequential)
          ▼
┌────────────────────┐
│  Task Dispatchers  │
│  (Parallel)        │
└─────────┬──────────┘
          │
          │ Submit to SQS
          ▼
┌────────────────────┐
│  Task Workers      │
│  (Earthly)         │
└─────────┬──────────┘
          │
          │ Update status
          ▼
┌────────────────────┐
│  DynamoDB + API    │◄─────────┐
└─────────┬──────────┘          │
          │                     │
          │ Status updates      │
          ▼                     │
┌────────────────────┐          │
│  GitHub            │          │
│  (Commit Status)   │          │
└────────────────────┘          │
          │                     │
          │ If release trigger  │
          ▼                     │
┌────────────────────┐          │
│  Release Phase     │          │
└─────────┬──────────┘          │
          │                     │
          │ [See Release Flow]  │
          └─────────────────────┘
```

#### Artifact Build and Publish Flow

```
┌────────────────────┐
│  Release           │
│  Orchestrator      │
└─────────┬──────────┘
          │
          │ For each artifact
          ▼
┌────────────────────┐
│  Artifacts         │
│  Component         │
└─────────┬──────────┘
          │
          │ Create staging
          ▼
┌────────────────────┐
│  Producer          │
│  (Earthly)         │
└─────────┬──────────┘
          │
          │ Execute build
          ▼
    [Artifact created]
          │
          │ Container → Docker daemon
          │ Binary/Archive → Staging FS
          ▼
┌────────────────────┐
│  SBOM Generator    │
│  (Syft)            │
└─────────┬──────────┘
          │
          │ Generate SBOM
          ▼
┌────────────────────┐
│  Publishers        │
│  (Parallel)        │
└─────────┬──────────┘
          │
          ├──────────────┬──────────────┬──────────────┐
          ▼              ▼              ▼              ▼
    Docker Pub     PyPI Pub      GitHub Pub    S3 Pub
          │              │              │              │
          │ Check exist  │ Check exist  │ Check exist  │ Check exist
          ▼              ▼              ▼              ▼
    [Push if new]  [Push if new]  [Upload]      [Upload]
          │              │              │              │
          └──────────────┴──────────────┴──────────────┘
                         │
                         │ All publishers complete
                         ▼
              ┌────────────────────┐
              │  OCI Builder       │
              │  (Tracking Image)  │
              └─────────┬──────────┘
                        │
                        │ Sign with Cosign
                        ▼
              ┌────────────────────┐
              │  Tracking Registry │
              │  (OCI Image)       │
              └─────────┬──────────┘
                        │
                        │ Return metadata
                        ▼
              ┌────────────────────┐
              │  Release Component │
              └────────────────────┘
```

#### Release Creation and Deployment Flow

```
┌────────────────────┐
│  Pipeline          │
│  (Release Phase)   │
└─────────┬──────────┘
          │
          │ Trigger release
          ▼
┌────────────────────┐
│  Release           │
│  Orchestrator      │
└─────────┬──────────┘
          │
          │ Build artifacts
          ▼
┌────────────────────┐
│  Artifacts         │
│  Component         │
└─────────┬──────────┘
          │
          │ Return artifact metadata
          ▼
┌────────────────────┐
│  Resource Renderer │
│  (CUE)             │
└─────────┬──────────┘
          │
          │ Resolve @artifact() refs
          │ Render Kubernetes YAML
          ▼
┌────────────────────┐
│  OCI Builder       │
│  (Release Image)   │
└─────────┬──────────┘
          │
          │ Package metadata + resources
          ▼
┌────────────────────┐
│  Release Registry  │
│  (OCI Image)       │
└─────────┬──────────┘
          │
          │ Push successful
          ▼
┌────────────────────┐
│  Database          │
│  (Release record)  │
└─────────┬──────────┘
          │
          │ If auto-deploy or manual
          ▼
┌────────────────────┐
│  Deployment        │
│  Manager           │
└─────────┬──────────┘
          │
          │ Check policies
          │ Check approvals
          ▼
┌────────────────────┐
│  GitOps Repo       │
│  (Pointer file)    │
└─────────┬──────────┘
          │
          │ Git commit & push
          ▼
    [Pointer updated]
          │
          │ Argo CD polls
          ▼
┌────────────────────┐
│  Argo CD           │
│  (Detect change)   │
└─────────┬──────────┘
          │
          │ Call CMP
          ▼
┌────────────────────┐
│  Custom Mgmt       │
│  Plugin            │
└─────────┬──────────┘
          │
          │ Fetch Release OCI
          ▼
┌────────────────────┐
│  Release Registry  │
└─────────┬──────────┘
          │
          │ Extract resources layer
          ▼
┌────────────────────┐
│  Argo CD           │
│  (Apply resources) │
└─────────┬──────────┘
          │
          ├──────────────┬──────────────┐
          ▼              ▼              ▼
    Standard K8s   Crossplane XR   ConfigMaps
          │              │              │
          └──────────────┴──────────────┘
                         │
                         ▼
              ┌────────────────────┐
              │  Kubernetes        │
              │  (Reconciliation)  │
              └────────────────────┘
```

#### Error Propagation Flow

```
┌────────────────────┐
│  Worker            │
│  (Step failure)    │
└─────────┬──────────┘
          │
          │ Write error to DynamoDB
          │ Exit with failure code
          ▼
┌────────────────────┐
│  Dispatcher        │
│  (Poll status)     │
└─────────┬──────────┘
          │
          │ Detect failure
          │ Propagate to Argo
          ▼
┌────────────────────┐
│  Argo Workflows    │
│  (Mark failed)     │
└─────────┬──────────┘
          │
          │ Terminate parallel tasks
          │ Skip remaining phases
          ▼
┌────────────────────┐
│  Pipeline API      │
│  (Webhook)         │
└─────────┬──────────┘
          │
          │ Update database
          │ Fetch error details
          ▼
┌────────────────────┐
│  GitHub API        │
│  (Update status)   │
└─────────┬──────────┘
          │
          │ Set commit status: failure
          │ Include error message
          ▼
┌────────────────────┐
│  Developer         │
│  (Notification)    │
└────────────────────┘
          │
          │ View logs
          ▼
┌────────────────────┐
│  Argo UI           │
│  or S3 presigned   │
└────────────────────┘
```

## Configuration

All configuration for integration contracts is specified in component-specific documents:

- [Configuration & Discovery](02-configuration-discovery.md) - Project and repository configuration
- [Execution & Orchestration](03-execution-orchestration.md) - Pipeline and worker configuration
- [Build & Distribution](04-build-distribution.md) - Artifact and publisher configuration
- [Release & Deployment](05-release-deployment.md) - Release and deployment configuration

## Operations

Operational aspects of integration are covered in component-specific documents:

- [Execution & Orchestration: Performance & Scaling](03-execution-orchestration.md#performance--scaling)
- [Execution & Orchestration: Monitoring & Observability](03-execution-orchestration.md#monitoring--observability)
- [Build & Distribution: Error Handling & Retry Logic](04-build-distribution.md#error-handling--retry-logic)
- [Release & Deployment: Error Handling](05-release-deployment.md#error-handling)

## Integration Points

For shared infrastructure patterns, see [Core Architecture](01-core-architecture.md)

For domain entities involved in integration, see [Domain Model & API Reference](06-domain-model-api-reference.md)

## Reference

### Integration Summary Table

| Source Component | Target Component | Contract Type | Protocol | Authentication |
|-----------------|------------------|---------------|----------|----------------|
| GitHub Actions | Pipeline API | Trigger Pipeline | REST | GitHub OIDC → Keycloak |
| Pipeline | Project | Read Config | File System | N/A |
| Pipeline | Argo Workflows | Submit Workflow | REST | K8s Service Account |
| Argo Dispatcher | SQS | Submit Job | AWS SDK | IRSA |
| Worker Pool | SQS | Consume Job | AWS SDK | IRSA |
| Worker Pool | DynamoDB | Update Status | AWS SDK | IRSA |
| Worker Pool | S3 | Write Results | AWS SDK | IRSA |
| Release | Artifacts | Build Artifacts | In-Process | N/A |
| Release | Project | Read Config | File System | N/A |
| Release | GitOps Repo | Update Pointer | Git | SSH Key |
| Argo CD | Release Registry | Fetch OCI | OCI Pull | Service Account |
| All Components | PostgreSQL | State Persistence | SQL (via API) | Keycloak Token |
| All Components | Grafana Cloud | Metrics/Logs | Prometheus/Loki | API Key |

### Contract Versioning

All API contracts follow semantic versioning:
- **Major version**: Breaking changes (e.g., `/v1/` → `/v2/`)
- **Minor version**: Backward-compatible additions (handled via optional fields)
- **Patch version**: Bug fixes (no API changes)

Current versions:
- Pipeline API: `v1`
- Release API: `v1`
- Artifacts API: `v1` (internal)
- Project Schema: `v1` (CUE `forgeVersion` field)

## Examples

Practical examples of integration patterns are provided in component-specific documents:

- [Execution & Orchestration: GitHub Integration](03-execution-orchestration.md#github-integration)
- [Build & Distribution: Producer Contract](04-build-distribution.md#producer-contract--implementations)
- [Release & Deployment: GitOps Integration](05-release-deployment.md#gitops-integration)

## Constraints & Limitations

### Integration Constraints

- **No Direct Database Access**: Components must use API layer for all database operations
- **No Cross-Component File Access**: Each component has isolated filesystem except during explicit handoffs
- **No Shared Memory**: Components communicate only through defined contracts
- **Sequential Phase Execution**: No parallel execution across phase groups

### Performance Considerations

- **API Rate Limits**: GitHub API limited to 5000 req/hour for authenticated requests
- **SQS Throughput**: 3000 messages/second per queue (standard queue)
- **DynamoDB Throughput**: On-demand billing mode, auto-scales
- **S3 Request Rate**: 3500 PUT/COPY/POST/DELETE and 5500 GET/HEAD per second per prefix

### Security Boundaries

- **Network Isolation**: Worker pools cannot directly access external APIs
- **Secret Isolation**: Secrets never traverse component boundaries as plain text
- **RBAC Enforcement**: All API calls authenticated and authorized via Keycloak
- **Audit Requirements**: All state-changing operations logged to database

## Future Enhancements

### Planned Integration Improvements

**Event Bus**: Introduce event-driven architecture for non-critical notifications
- Reduces tight coupling between components
- Enables async notifications without blocking operations
- Supports future integration with external systems

**Enhanced Observability**: Distributed tracing across all component interactions
- OpenTelemetry instrumentation
- End-to-end trace visibility from GitHub push to deployment
- Performance bottleneck identification

**Contract Testing**: Automated validation of inter-component contracts
- Consumer-driven contract tests
- API compatibility verification in CI
- Breaking change detection

**Service Mesh**: Introduce Istio or Linkerd for advanced traffic management
- Mutual TLS between components
- Fine-grained retry and timeout policies
- Circuit breaking for fault tolerance

---

**Component Owner**: Platform Architecture
**Related Documents**: All component specifications
**Last Updated**: 2025-10-02
