# Domain Model

## Table of Contents

- [Executive Summary](#executive-summary)
- [Core Concepts](#core-concepts)
  - [Entity Ownership](#entity-ownership)
  - [Canonical Identifiers](#canonical-identifiers)
  - [Database Technology](#database-technology)
- [Domain Entities](#domain-entities)
  - [Pipeline Execution Domain](#pipeline-execution-domain)
  - [Release Domain](#release-domain)
  - [Deployment Domain](#deployment-domain)
- [Entity Relationships](#entity-relationships)
  - [Pipeline Execution Hierarchy](#pipeline-execution-hierarchy)
  - [Release Lifecycle](#release-lifecycle)
  - [Cross-Domain Relationships](#cross-domain-relationships)
- [Integration Points](#integration-points)
  - [Component Access Patterns](#component-access-patterns)
  - [External System References](#external-system-references)
- [Constraints](#constraints)
  - [Data Integrity](#data-integrity)
  - [Business Rules](#business-rules)
  - [Scale Considerations](#scale-considerations)
- [Future Considerations](#future-considerations)
  - [Potential Enhancements](#potential-enhancements)

## Executive Summary

This document defines the domain entities and relationships within the Catalyst Forge platform. Every entity has exactly one authoritative definition in this document. Components may reference entities but must not redefine them.

The domain model organizes entities by ownership: Pipeline Component owns execution entities, Release Component owns release and deployment entities, and Artifacts Component contributes artifact tracking entities. All entities use PostgreSQL for persistent storage with UUID primary keys and canonical triplet business identifiers.

## Core Concepts

### Entity Ownership

Entities are owned by the component responsible for their lifecycle:

**Pipeline Component:** PipelineRun, PhaseExecution, TaskExecution, StepExecution

**Release Component:** Release, ReleaseAlias, ReleaseTrigger, ReleaseApproval, Deployment

**Artifacts Component:** ReleaseArtifact (contributes to Release domain)

Ownership determines which component has create, update, and delete authority for an entity. Other components may read entities but must coordinate writes through the owning component.

### Canonical Identifiers

All entities use UUIDs as primary keys for global uniqueness and referential integrity. Business identifiers follow the canonical triplet pattern: `repository/project/commit-sha`.

This pattern provides:
- **Repository scope:** Organizational namespace
- **Project scope:** Deliverable unit within repository
- **Commit SHA:** Immutable version identity

The triplet appears consistently across releases, artifacts, deployments, and namespace naming.

### Database Technology

The platform uses PostgreSQL for all persistent storage. Entity schemas reflect relational model with foreign key constraints for referential integrity. Indexes support common query patterns.

## Domain Entities

### Pipeline Execution Domain

These entities track the complete execution history of CI/CD pipelines.

#### PipelineRun

**Owner:** Pipeline Component

**Purpose:** Represents a single execution of the CI/CD pipeline for a commit

**Key Fields:**
- `id` (UUID, PK): Global unique identifier
- `repository` (string): Repository identifier
- `branch` (string): Source branch
- `commit_sha` (string): Source commit
- `triggered_by` (string): Actor identifier
- `trigger_type` (enum): push, pull_request, manual
- `status` (enum): pending, running, success, failed, cancelled
- `started_at` (timestamp): Execution start time
- `completed_at` (timestamp, nullable): Execution end time
- `argo_workflow_name` (string): Reference to Argo Workflow
- `discovery_output` (jsonb): Cached discovery results

**Unique Constraints:**
- `(repository, commit_sha)`: One pipeline run per commit

**Indexes:**
- `(repository, branch, created_at)`: Query branch history
- `(status, started_at)`: Active runs

**Relationships:**
- Has many PhaseExecutions (1:N)
- Has zero or one Release (1:0..1)

#### PhaseExecution

**Owner:** Pipeline Component

**Purpose:** Tracks execution of a single phase within a pipeline run

**Key Fields:**
- `id` (UUID, PK): Global unique identifier
- `pipeline_run_id` (UUID, FK → PipelineRun): Parent pipeline run
- `phase_name` (string): Phase identifier
- `group_number` (integer): Sequential group for ordering
- `status` (enum): pending, running, success, failed, skipped
- `started_at` (timestamp, nullable): Phase start time
- `completed_at` (timestamp, nullable): Phase end time

**Unique Constraints:**
- `(pipeline_run_id, phase_name)`: One execution per phase

**Indexes:**
- `(pipeline_run_id, group_number)`: Phase ordering

**Relationships:**
- Belongs to PipelineRun (N:1)
- Has many TaskExecutions (1:N)

#### TaskExecution

**Owner:** Pipeline Component

**Purpose:** Tracks execution of all steps for a specific project within a phase

**Key Fields:**
- `id` (UUID, PK): Global unique identifier
- `phase_execution_id` (UUID, FK → PhaseExecution): Parent phase
- `project_name` (string): Project identifier
- `project_path` (string): Repository-relative path
- `status` (enum): pending, running, success, failed, skipped
- `started_at` (timestamp, nullable): Task start time
- `completed_at` (timestamp, nullable): Task end time

**Unique Constraints:**
- `(phase_execution_id, project_name)`: One task per project per phase

**Relationships:**
- Belongs to PhaseExecution (N:1)
- Has many StepExecutions (1:N)

#### StepExecution

**Owner:** Pipeline Component

**Purpose:** Tracks execution of a single step within a task

**Key Fields:**
- `id` (UUID, PK): Global unique identifier
- `task_execution_id` (UUID, FK → TaskExecution): Parent task
- `step_name` (string): Step identifier
- `action` (string): Execution action (e.g., "earthly")
- `target` (string): Target specification (e.g., "+test")
- `status` (enum): pending, running, success, failed, skipped
- `started_at` (timestamp, nullable): Step start time
- `completed_at` (timestamp, nullable): Step end time
- `exit_code` (integer, nullable): Process exit code
- `logs_s3_key` (string, nullable): Log storage reference

**Unique Constraints:**
- `(task_execution_id, step_name)`: One step per name per task

**Relationships:**
- Belongs to TaskExecution (N:1)

### Release Domain

These entities represent immutable snapshots of projects and their deployment lifecycle.

#### Release

**Owner:** Release Component

**Purpose:** Immutable snapshot of a project at a specific commit

**Key Fields:**
- `id` (UUID, PK): Global unique identifier
- `repository` (string): Repository identifier
- `project` (string): Project identifier
- `commit_sha` (string): Canonical version identifier
- `release_number` (integer): Auto-incrementing per project
- `pipeline_run_id` (UUID, FK → PipelineRun, nullable): Optional pipeline reference
- `oci_uri` (string): Release OCI image reference
- `oci_digest` (string): SHA256 digest
- `created_at` (timestamp): Creation time
- `created_by` (string): Actor identifier

**Unique Constraints:**
- `(repository, project, commit_sha)`: Canonical triplet uniqueness
- `(repository, project, release_number)`: Numeric uniqueness per project

**Indexes:**
- `(created_at)`: Chronological listing

**Relationships:**
- Optionally references PipelineRun (N:0..1): pipeline_run_id may be null for manually created releases
- Has many ReleaseAliases (1:N)
- Has one ReleaseTrigger (1:1)
- Has many ReleaseArtifacts (1:N)
- Has many Deployments (1:N)
- Has many ReleaseApprovals (1:N)

**Business Logic:**
- `release_number` auto-increments within `(repository, project)` scope
- Releases are immutable once created
- `pipeline_run_id` may be null for CLI-created releases

#### ReleaseAlias

**Owner:** Release Component

**Purpose:** Human-readable aliases for releases

**Key Fields:**
- `id` (UUID, PK): Global unique identifier
- `release_id` (UUID, FK → Release): Parent release
- `alias` (string): Alias value
- `alias_type` (enum): numeric, tag, branch, custom

**Unique Constraints:**
- `(release_id, alias)`: One alias value per release

**Indexes:**
- `(alias, alias_type)`: Lookup by alias

**Relationships:**
- Belongs to Release (N:1)

**Business Logic:**
- Numeric aliases correspond to release_number
- Tag aliases track Git tag releases
- Branch aliases track branch-based releases

#### ReleaseTrigger

**Owner:** Release Component

**Purpose:** Records what triggered release creation

**Key Fields:**
- `release_id` (UUID, PK, FK → Release): Parent release (also primary key)
- `trigger_type` (enum): branch_push, tag, manual
- `branch` (string, nullable): Branch name if branch_push
- `tag` (string, nullable): Tag name if tag
- `triggered_by` (string): Actor identifier
- `triggered_at` (timestamp): Trigger time

**Relationships:**
- Belongs to Release (1:1)

**Business Logic:**
- Exactly one trigger per release
- Only one of branch or tag is populated based on trigger_type

#### ReleaseArtifact

**Owner:** Artifacts Component (contributes to Release domain)

**Purpose:** Links releases to their build artifacts

**Key Fields:**
- `id` (UUID, PK): Global unique identifier
- `release_id` (UUID, FK → Release): Parent release
- `artifact_name` (string): Artifact identifier
- `artifact_type` (enum): container, binary, archive
- `tracking_oci_uri` (string): Artifact OCI tracking image
- `tracking_oci_digest` (string): SHA256 digest
- `primary_published_uri` (string): Main registry location
- `created_at` (timestamp): Creation time

**Unique Constraints:**
- `(release_id, artifact_name)`: One artifact per name per release

**Relationships:**
- Belongs to Release (N:1)

**Business Logic:**
- Full artifact metadata (build info, provenance, SBOM) stored in Artifact OCI tracking image, not in database
- Database contains only references and primary location

### Deployment Domain

These entities track deployments across environments.

#### Deployment

**Owner:** Release Component (shared with Pipeline)

**Purpose:** Tracks deployment of a release to an environment

**Key Fields:**
- `id` (UUID, PK): Global unique identifier
- `release_id` (UUID, FK → Release): Deployed release
- `environment` (string): Environment identifier
- `status` (enum): pending, active, superseded
- `deployed_at` (timestamp): Deployment time
- `deployed_by` (string): Actor identifier
- `gitops_commit_sha` (string): GitOps repository commit

**Unique Constraints:**
- Only one deployment per `(environment, release_id)` with `status='active'`

**Indexes:**
- `(environment, release_id)`: Track deployments per environment
- `(environment, status)`: Query active deployment
- `(release_id, environment)`: Deployment history per release

**Relationships:**
- Belongs to Release (N:1)
- May have one ReleaseApproval (0..1:1)

**Business Logic:**
- When new deployment becomes active, previous deployment for same environment transitions to superseded
- Pending deployments await infrastructure reconciliation
- Active deployments represent current state

#### ReleaseApproval

**Owner:** Release Component

**Purpose:** Records approval for deploying release to environment

**Key Fields:**
- `id` (UUID, PK): Global unique identifier
- `release_id` (UUID, FK → Release): Approved release
- `environment` (string): Target environment
- `approver` (string): Keycloak identity
- `approved_at` (timestamp): Approval time
- `expires_at` (timestamp, nullable): Expiration time
- `justification` (text): Approval reason
- `status` (enum): active, expired, revoked

**Indexes:**
- `(release_id, environment)`: Lookup approval for deployment
- `(approver, approved_at)`: Audit trail

**Relationships:**
- Belongs to Release (N:1)
- May be referenced by Deployment (1:0..1)

**Business Logic:**
- Production deployments require active approval
- Approvals may expire based on policy
- Revoked approvals prevent deployment

## Entity Relationships

### Pipeline Execution Hierarchy

```
PipelineRun (1)
    ↓ has many
PhaseExecution (N)
    ↓ has many
TaskExecution (N)
    ↓ has many
StepExecution (N)
```

This hierarchy reflects the execution model:
- One pipeline run per commit
- Multiple phases execute sequentially
- Multiple tasks execute in parallel within phase
- Multiple steps execute per task

### Release Lifecycle

```
PipelineRun (0..1) ← optional, may be null
    ↓ may create
Release (1) ← canonical: repository/project/commit_sha
    ├─ has many → ReleaseAlias (N)
    ├─ has one → ReleaseTrigger (1)
    ├─ has many → ReleaseArtifact (N)
    ├─ has many → Deployment (N)
    └─ has many → ReleaseApproval (N)
         ↑
         └─ referenced by → Deployment
```

This structure supports:
- Automated and manual release creation
- Multiple aliases per release (tag, branch, numeric)
- Artifact tracking with provenance
- Multi-environment deployment
- Approval workflows for controlled environments

### Cross-Domain Relationships

```
PipelineRun → Release (optionally creates, 1:0..1)
Release → ReleaseArtifact (contains, 1:N)
Release → Deployment (deployed as, 1:N)
Deployment → ReleaseApproval (optionally requires, 0..1:1)
```

These relationships enable:
- Tracing releases back to pipeline runs
- Tracking all artifacts within release
- Managing deployment across environments
- Enforcing approval requirements

## Integration Points

### Component Access Patterns

**Pipeline Component:**
- Creates and updates PipelineRun, PhaseExecution, TaskExecution, StepExecution
- Reads Release for status reporting
- Creates Release when release trigger fires

**Release Component:**
- Creates and updates Release, ReleaseAlias, ReleaseTrigger, Deployment, ReleaseApproval
- Reads PipelineRun for release creation context
- Reads ReleaseArtifact for packaging

**Artifacts Component:**
- Creates ReleaseArtifact during release phase
- Reads Release for artifact coordination

All cross-component access flows through Platform API for authentication and audit logging.

### External System References

**Argo Workflows:**
- `PipelineRun.argo_workflow_name`: Reference to workflow execution
- Platform does not store Argo state, only references

**Object Storage:**
- `StepExecution.logs_s3_key`: Log file location
- Full logs stored in S3, not database

**OCI Registries:**
- `Release.oci_uri`: Release OCI image location
- `ReleaseArtifact.tracking_oci_uri`: Artifact tracking image location
- Full metadata stored in OCI images, not database

## Constraints

### Data Integrity

- All foreign keys enforced by PostgreSQL constraints
- Unique constraints prevent duplicate entries
- Nullable fields explicitly marked in schemas
- Enums constrain status values to valid states

### Business Rules

- Pipeline runs immutable after completion
- Releases immutable once created
- Only one active deployment per environment-release pair
- Release numbers auto-increment without gaps
- Approvals required for production deployments

### Scale Considerations

- Pipeline execution hierarchy can grow deep for long-running pipelines
- Release artifact count limited by storage capacity
- Deployment history grows linearly with release frequency
- Indexes optimize common query patterns

## Future Considerations

### Potential Enhancements

**Event Sourcing:**
Domain events for complete audit trails and temporal queries. Event stream per aggregate root enables reconstruction of entity state at any point in time.

**Read Models:**
Denormalized projections for query optimization. Separate read and write models enable independent scaling and performance tuning.

**Domain Extensions:**
Additional entities for feature enhancements (FeatureFlags, AuditLog, UserPreferences). Entity model remains extensible through standard PostgreSQL schema evolution.

---

**Related Documents:**
- System Architecture - Component architecture, service model, and data flows
- API & Operations Reference - REST API specifications and entity operations
- Developer Guide - Configuration patterns and entity relationships

**Last Updated:** 2025-10-05
