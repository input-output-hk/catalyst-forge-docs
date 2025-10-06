# System Architecture

## Table of Contents

- [Executive Summary](#executive-summary)
- [Core Concepts](#core-concepts)
  - [Architectural Model](#architectural-model)
  - [Dual Persistence Model](#dual-persistence-model)
  - [Ephemeral Messaging Layer](#ephemeral-messaging-layer)
- [Architecture](#architecture)
  - [Service Architecture](#service-architecture)
  - [Service Communication Topology](#service-communication-topology)
  - [Component Architecture](#component-architecture)
  - [Messaging Infrastructure](#messaging-infrastructure)
  - [Data Flow Architecture](#data-flow-architecture)
  - [Shared Concepts](#shared-concepts)
- [Integration Points](#integration-points)
  - [External System Integration](#external-system-integration)
  - [Internal Component Integration](#internal-component-integration)
- [Constraints](#constraints)
  - [Architectural Constraints](#architectural-constraints)
  - [Scale Considerations](#scale-considerations)
  - [Operational Boundaries](#operational-boundaries)
- [Future Considerations](#future-considerations)
  - [Potential Enhancements](#potential-enhancements)
  - [Extension Points](#extension-points)

## Executive Summary

This document defines the technical architecture of the Catalyst Forge platform, including service structure, component interactions, and data flows. The architecture implements a three-service runtime model (Platform API, Worker Service, Dispatcher) that realizes four logical components (Pipeline, Project, Artifacts, Release & Deployment).

The platform follows an API-first architecture with asynchronous job execution through NATS JetStream messaging. All durable state resides in PostgreSQL and object storage, while the messaging layer remains intentionally ephemeral for operational simplicity.

## Core Concepts

### Architectural Model

The platform separates logical component organization from physical service deployment:

**Logical Components:** Organize responsibilities and concerns (Pipeline, Project, Artifacts, Release & Deployment)

**Physical Services:** Runtime deployments that implement component logic (Platform API, Worker Service, Dispatcher)

This separation enables clean architecture documentation while maintaining operational efficiency through consolidated services.

### Dual Persistence Model

The platform maintains state in two distinct layers:

**Argo Workflows:** Volatile execution state for active pipeline runs. Workflow definitions, step status, and retry state exist only during execution. When workflows complete or fail, this state is discarded.

**PostgreSQL:** Permanent audit trail and queryable history. All domain entities (PipelineRun, Release, Deployment, Artifact) persist with complete provenance. Users interact exclusively through this layer.

This separation maintains platform integrity independent of workflow state. Recovery involves resubmitting work, not reconstructing historical data.

### Ephemeral Messaging Layer

NATS JetStream provides job distribution without durability requirements. Messages represent in-flight work, not permanent state. Lost messages result in timeouts and retries, not data loss.

All artifacts, logs, and results are stored in object storage with references in the database. The messaging layer only coordinates work distribution between dispatcher pods and worker services.

## Architecture

### Service Architecture

The platform deploys as three core services with distinct responsibilities:

#### Platform API

**Purpose:** Single unified REST API for all platform operations

**Responsibilities:**
- Domain entity management (PipelineRun, Release, Deployment, Artifact)
- Authentication via Keycloak (GitHub OIDC token exchange, CLI users, service accounts)
- Webhook ingestion from GitHub Actions
- Asynchronous GitHub commit status updates
- Workflow submission to Argo Server

**Deployment:** Kubernetes Deployment with horizontal pod autoscaling

**External Interfaces:**
- REST API (HTTPS) for GitHub Actions, CLI, and internal services
- PostgreSQL connection pool for state persistence
- Argo Server REST API for workflow submission
- GitHub API for commit status updates
- Keycloak for authentication

**Scaling:** Horizontal based on CPU and memory utilization

#### Worker Service

**Purpose:** Execute all asynchronous platform jobs

**Responsibilities:**
- Consume jobs from NATS JetStream pull consumers
- Route jobs to internal handlers based on job type
- Maintain persistent Git repository caches
- Execute Earthly targets with warm connections
- Report results via request-reply pattern to dispatcher
- Update domain entities through Platform API

**Internal Job Handlers:**
- **Discovery Handler:** Repository traversal, CUE parsing, release trigger evaluation
- **CI Handler:** Earthly target execution for test and build phases
- **Artifact Handler:** Producer/publisher orchestration, SBOM generation, tracking image creation
- **Release Handler:** CUE to Kubernetes resource rendering, Release OCI packaging
- **Deployment Handler:** GitOps repository pointer file updates

**Deployment:** Multiple Kubernetes StatefulSets with different resource profiles

**StatefulSet Organization:**
- `worker-discovery`: Repository discovery and configuration parsing (3 replicas, 50Gi cache)
- `worker-ci`: Build and test execution (10 replicas, 200Gi cache)
- `worker-artifact`: Artifact publishing and tracking (5 replicas, 150Gi cache)
- `worker-release`: Release packaging and deployment (3 replicas, 100Gi cache)

**Persistent Storage:** Each StatefulSet pod maintains individual Git repository cache via volumeClaimTemplates

**Scaling:** KEDA-based autoscaling on NATS JetStream consumer lag per worker type

**External Interfaces:**
- NATS JetStream for job consumption and reply
- Object storage (S3/MinIO) for logs and large results
- Platform API for domain entity updates
- Container registries for artifact publication
- GitOps repository for deployment pointer updates

#### Dispatcher

**Purpose:** Bridge between Argo Workflows and Worker Service

**Responsibilities:**
- Publish jobs to NATS JetStream with unique reply subjects
- Await direct replies from worker via request-reply pattern
- Return results to Argo Workflows
- Handle timeouts and failures

**Deployment:** Ephemeral pods created by Argo Workflows

**Configuration:** Job type specified via environment variables (discovery, ci, artifact, release, deployment)

**Lifecycle:** Created per workflow task, destroyed after reply or timeout

**External Interfaces:**
- NATS JetStream for job publication and reply subscription
- Argo Workflows for result return

### Service Communication Topology

The three services interact through well-defined patterns:

```
External Trigger:
  GitHub Actions → Platform API (REST/HTTPS)
                → PostgreSQL (persist PipelineRun)
                → Argo Server (submit workflow)

Workflow Execution:
  Argo Workflows → Dispatcher pod (created ephemerally)
                 → NATS JetStream (publish with reply subject)
                 → Worker Service (pull from durable consumer)
                              → Execute job
                              → Update entities via Platform API
                              → Reply to dispatcher via NATS
                 ← Receive reply
  Dispatcher → Argo Workflows (return result)

Asynchronous Updates:
  Worker Service → Platform API (update domain entities)
  Platform API → GitHub API (commit status updates)
```

**Message Subjects:**
- Discovery jobs: `pipeline.discovery.<run-id>`
- CI jobs: `pipeline.tasks.<run-id>.<task-id>`
- Artifact jobs: `pipeline.artifacts.<run-id>.<artifact-id>`
- Release/Deployment jobs: `pipeline.releases.<run-id>`

**Request-Reply Pattern:**
- Dispatcher creates unique inbox subscription
- Publishes job with `ReplyTo` field set to inbox
- Worker processes job and replies to inbox
- Dispatcher receives reply or times out
- Job acknowledged only after successful reply

### Component Architecture

While physically realized through three services, the platform organizes logic into four components:

#### Pipeline Component

**Responsibilities:**
- Repository and project discovery
- Dynamic CI pipeline construction
- Phase orchestration and execution
- State persistence and audit trails
- Worker coordination and monitoring

**Physical Realization:**
- Platform API: Pipeline run management, workflow submission
- Dispatcher: Phase and task coordination
- Worker Service (Discovery Handler): Repository traversal and DAG construction
- Worker Service (CI Handler): Phase execution
- Argo Workflows: Orchestration engine

#### Project Component

**Responsibilities:**
- Configuration schema definition (CUE)
- Project discovery mechanisms
- Repository-level settings management
- Deployment specification validation
- Configuration to Kubernetes resource transformation

**Physical Realization:**
- Worker Service (Discovery Handler): CUE parsing and validation
- Worker Service (Release Handler): Resource rendering

#### Artifacts Component

**Responsibilities:**
- Artifact type management (container, binary, archive)
- Producer plugin coordination (Earthly)
- Publisher plugin coordination (Docker, PyPI, GitHub, S3)
- Artifact upload orchestration
- OCI tracking image creation for provenance
- Registry credential management

**Physical Realization:**
- Worker Service (Artifact Handler): Producer/publisher orchestration
- Worker Service (CI Handler): Artifact creation via Earthly
- Platform API: Artifact entity persistence

#### Release & Deployment Component

**Responsibilities:**
- Release trigger evaluation
- Snapshot creation including Kubernetes resource definitions
- OCI image packaging for GitOps consumption
- Environment progression rules
- Promotion and approval workflows
- GitOps repository updates

**Physical Realization:**
- Worker Service (Release Handler): Resource rendering, OCI packaging
- Worker Service (Deployment Handler): GitOps pointer file updates
- Platform API: Release and Deployment entity management
- Argo CD Custom Management Plugin: Release OCI extraction

### Messaging Infrastructure

NATS JetStream provides work queue semantics with durable streams and pull consumers.

**Stream Configuration:**

Four streams partition work by type:
- `pipeline-discovery`: Repository discovery jobs
- `pipeline-tasks`: CI execution jobs (test, build phases)
- `pipeline-artifacts`: Artifact handling jobs
- `pipeline-releases`: Release and deployment jobs

**Stream Characteristics:**
- Retention: Work queue (messages deleted after ack)
- Max age: 1 hour (stale jobs discarded)
- Max messages: 1000 per stream
- Storage: File-backed with replication
- Replicas: 3 for availability

**Consumer Configuration:**

Each Worker StatefulSet creates durable pull consumer:
- Acknowledgment policy: Explicit (manual ack after processing)
- Replay policy: Instant (no message replay)
- Max deliver: 3 attempts before DLQ
- Ack wait: Matches job type timeout

**Deployment:**

3-node NATS cluster with persistent volumes for stream storage. Ephemeral data only—no disaster recovery required. Lost streams result in job resubmission, not data loss.

### Data Flow Architecture

The platform processes work through well-defined flows from commit to deployment:

#### Pipeline Execution Flow

```
1. Trigger
   GitHub Actions → Platform API (create PipelineRun)
                 → PostgreSQL (persist entity)
                 → Argo Server (submit workflow)

2. Discovery
   Argo Workflow → Dispatcher pod
                 → NATS (pipeline.discovery.<run-id>)
                 → Worker Service (Discovery Handler)
                              → Clone/fetch repository
                              → Parse .forge/ CUE configs
                              → Build execution DAG
                              → Evaluate release triggers
                              → Reply with discovery output
                 ← Dispatcher receives reply
   Workflow continues with dynamic task generation

3. Phase Execution
   Argo Workflow → Phase loop (sequential)
                 → Task dispatchers (parallel within phase)
                 → NATS (pipeline.tasks.<run-id>.<task-id>)
                 → Worker Service (CI Handler)
                              → Execute Earthly target
                              → Stream logs to object storage
                              → Reply with result
                 ← Dispatcher receives replies
   Next phase begins only if all tasks succeed

4. Release Creation (conditional)
   Argo Workflow → Artifact dispatcher
                 → NATS (pipeline.artifacts.<run-id>)
                 → Worker Service (Artifact Handler)
                              → Execute producers (create artifacts)
                              → Execute publishers (distribute artifacts)
                              → Create OCI tracking images
                              → Reply with artifact references
                 ← Dispatcher receives reply

   Workflow → Release dispatcher
            → NATS (pipeline.releases.<run-id>)
            → Worker Service (Release Handler)
                         → Render CUE to Kubernetes YAML
                         → Reference artifact OCI images
                         → Package Release OCI image
                         → Reply with release metadata
            ← Dispatcher receives reply

5. Deployment Initiation
   Workflow → Deployment dispatcher
            → NATS (pipeline.releases.<run-id>)
            → Worker Service (Deployment Handler)
                         → Update ReleasePointer in GitOps repo
                         → Reply with deployment status
            ← Dispatcher receives reply
```

#### GitOps Deployment Flow

```
1. Pointer Update
   Worker Service → GitOps Repository (commit ReleasePointer change)

2. Change Detection
   Argo CD → Polls GitOps repository
          → Detects ReleasePointer change

3. Resource Extraction
   Argo CD → Custom Management Plugin
          → Read ReleasePointer spec.release (commit SHA)
          → Fetch Release OCI image from registry
          → Extract resources layer (YAML manifests)
          → Return to Argo CD

4. Resource Application
   Argo CD → Kubernetes API (apply resources)
          → Crossplane resources (XRs) → Composition functions
                                        → EnvironmentConfig resolution
                                        → Create managed resources
          → Standard Kubernetes resources → Controllers reconcile

5. Operator Reconciliation
   External Secrets Operator → Sync secrets from providers
   External DNS → Create DNS records
   Istio → Configure ingress routes (Gateway API) and service mesh policies
```

### Shared Concepts

#### OCI Image Formats

The platform uses two distinct OCI image types for artifact tracking and deployment.

**Artifact OCI Tracking Images:**

Purpose: Immutable provenance records for individual artifacts

Format: `[tracking-registry]/[repository]/[project]/[artifact]:sha-[commit-sha]`

Contents:
- Artifact metadata (name, type, project, commit)
- Published locations (all registries where artifact exists)
- Build information (pipeline run, producer details, timestamp)
- Provenance data (source materials, builder identity)
- SBOM layer (software bill of materials)

Usage:
- Security auditing and vulnerability tracking
- Supply chain verification
- Artifact metadata lookup
- Tracing artifacts back to source commits

**Release OCI Images:**

Purpose: GitOps deployment packages with Kubernetes resource definitions

Format: `[release-registry]/[repository]/[project]:[commit-sha]`

Contents:
- Release metadata (release number, aliases, trigger information)
- Artifact references (OCI tracking image URIs for complete provenance)
- Rendered Kubernetes YAML (ready to apply to clusters)

Usage:
- Argo CD extracts and applies resources to clusters
- Provides complete deployment snapshot
- Enables reproducible deployments across environments
- Maintains audit trail from source to deployment

**Relationship:**

Release OCI images reference Artifact OCI images in metadata layer, creating complete audit trail:

```
Source Commit
  ↓
Pipeline Builds → Artifact OCI Images (per artifact)
  ↓
Release Trigger Evaluates
  ↓
CUE Configs Render → Release OCI Image (references Artifact OCIs)
  ↓
GitOps Deployment → Argo CD extracts Release OCI
  ↓
Resources Applied → Containers reference Artifact OCIs
```

This two-tier approach provides separation of concerns (artifacts independent from releases), reusability (same artifact in multiple releases), and complete provenance (trace deployment through release to artifacts to source).

#### GitOps Repository Structure

Single GitOps repository with hierarchical organization for deployment management:

```
gitops-repo/
├── environments/
│   ├── dev/
│   │   └── {repository}-{project}/
│   │       └── release.yaml
│   ├── staging/
│   │   └── {repository}-{project}/
│   │       └── release.yaml
│   └── production/
│       └── {repository}-{project}/
│           └── release.yaml
```

**ReleasePointer Format:**

ReleasePointer resources specify the deployed release via commit SHA in `spec.release`, with optional aliases for human readability. Metadata includes deployment timestamp and actor.

**Argo CD Integration:**

Custom Management Plugin processes ReleasePointers:
1. Read ReleasePointer from Git
2. Extract `spec.release` commit SHA
3. Fetch Release OCI image using SHA
4. Extract resources layer (tar archive of YAML)
5. Return resources to Argo CD for application

Single repository enables unified access control and audit logging. Environment-first hierarchy separates concerns by deployment target. Pointer indirection decouples Git commits from release versions. Canonical SHA ensures deployment reproducibility.

## Integration Points

### External System Integration

**GitHub:**
- Webhook trigger source for pipeline initiation
- API target for commit status updates
- OIDC token provider for authentication

**Keycloak:**
- Central authentication for all platform access
- GitHub OIDC token exchange endpoint
- Service account token issuance
- Federation to external identity providers

**Container Registries:**
- Artifact OCI image storage
- Release OCI image storage
- Implements OCI Distribution Spec

**Object Storage:**
- Pipeline logs and artifacts
- Large job results (>1MB)
- Implements S3-compatible API

**DNS Provider:**
- ExternalDNS-compatible record management
- Platform integrates via standard APIs

### Internal Component Integration

Components interact through well-defined contracts:

**Pipeline → Project:**
- Discovery Handler reads CUE configurations
- Configuration validation during discovery
- Project definitions drive workflow generation

**Pipeline → Artifacts:**
- Artifact Handler executes during release phase
- Producers create artifacts via Earthly
- Publishers distribute to registries
- Tracking images created for provenance

**Pipeline → Release & Deployment:**
- Release Handler packages Kubernetes resources
- Deployment Handler updates GitOps pointers
- Release triggers evaluated during discovery

**Artifacts → Release:**
- Release OCI images reference Artifact OCI images
- CUE `@artifact()` functions resolve to tracking image URIs
- Complete provenance chain maintained

All inter-component calls flow through Platform API for audit trail and persistence.

## Constraints

### Architectural Constraints

- Sequential phase execution (phases cannot execute in parallel)
- Single pipeline run per commit (no concurrent runs for same commit)
- Ephemeral messaging (lost NATS messages require job resubmission)
- Stateful workers (pod eviction loses cache warmth)

### Scale Considerations

- Worker Service scaling limited by NATS consumer lag response time
- Git cache effectiveness degrades with repository count over ~100
- NATS JetStream stream limits (1000 messages per stream)
- PostgreSQL connection pool sizing affects API concurrency

### Operational Boundaries

- Platform API must remain accessible for worker status updates
- NATS cluster downtime halts all job distribution (but not running jobs)
- Argo Workflows single point of failure for orchestration
- PostgreSQL downtime prevents new pipeline runs (existing runs continue)

## Future Considerations

### Potential Enhancements

**Distributed Worker Caching:**
Shared Git cache using Redis or distributed filesystem to improve cache hit rates across worker pods.

**Multi-Region Workers:**
Worker Service deployments in multiple regions for reduced latency and disaster recovery.

**Alternative Execution Engines:**
Support for execution backends beyond Earthly through consistent abstraction.

**Enhanced Observability:**
Distributed tracing through service mesh for complete request flow visibility.

### Extension Points

- Message queue contract enables alternative implementations (Kafka, RabbitMQ)
- Worker Service handlers can be extended for new job types
- Dispatcher routing logic supports additional stream subjects
- Platform API versioning enables contract evolution

---

**Related Documents:**
- Architecture Overview - Ports and adapters pattern and architectural principles
- Platform Contracts - Infrastructure contract specifications and adapters
- Implementation Guide - Deployment architecture and infrastructure abstractions
- Developer Guide - Configuration patterns and deployment workflows

**Last Updated:** 2025-10-05
