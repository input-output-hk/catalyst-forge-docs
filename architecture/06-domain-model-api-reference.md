# Domain Model & API Reference

## Executive Summary
Single reference for all domain entities and APIs across the Catalyst Forge platform. This document consolidates entity definitions, ownership assignments, and API specifications from all components.

Every entity has exactly one authoritative definition in this document. Components may reference entities but must not redefine them.

## Core Concepts

### Entity Ownership Model

Entities are owned by the component responsible for their lifecycle:
- **Pipeline Component**: PipelineRun, PhaseExecution, TaskExecution, StepExecution
- **Release Component**: Release, ReleaseAlias, ReleaseTrigger, ReleaseApproval
- **Artifacts Component**: Artifact (contributes to Release Component's domain)
- **Deployment Component**: Deployment (shared between Pipeline and Release)

### Canonical Identifiers

All entities use UUIDs as primary keys for global uniqueness. Business identifiers follow the canonical triplet pattern: `repository/project/commit-sha`.

### Database Technology

The platform uses **PostgreSQL** for all persistent storage. Entity schemas below reflect the relational model.

## Domain Entity Catalog

### Pipeline Execution Entities

These entities track the complete execution history of CI/CD pipelines.

#### PipelineRun

**Owner**: Pipeline Component
**Purpose**: Represents a single execution of the CI/CD pipeline for a commit

**Schema**:
```sql
Fields:
- id: UUID (PK)
- repository: string
- branch: string
- commit_sha: string
- triggered_by: string (actor identifier)
- trigger_type: enum (push, pull_request, manual)
- status: enum (pending, running, success, failed, cancelled)
- started_at: timestamp
- completed_at: timestamp (nullable)
- argo_workflow_name: string (reference to Argo Workflows)
- discovery_output: jsonb (cached discovery results)

Indexes:
- (repository, commit_sha) - Unique constraint
- (repository, branch, created_at) - Query by branch history
- (status, started_at) - Active runs
```

**Relationships**:
- Has many PhaseExecutions (1:N)
- Has zero or one Release (1:0..1)

#### PhaseExecution

**Owner**: Pipeline Component
**Purpose**: Tracks execution of a single phase within a pipeline run

**Schema**:
```sql
Fields:
- id: UUID (PK)
- pipeline_run_id: UUID (FK → PipelineRun)
- phase_name: string
- group_number: integer
- status: enum (pending, running, success, failed, skipped)
- started_at: timestamp (nullable)
- completed_at: timestamp (nullable)

Indexes:
- (pipeline_run_id, phase_name) - Unique constraint
- (pipeline_run_id, group_number) - Phase ordering
```

**Relationships**:
- Belongs to PipelineRun (N:1)
- Has many TaskExecutions (1:N)

#### TaskExecution

**Owner**: Pipeline Component
**Purpose**: Tracks execution of all steps for a specific project within a phase

**Schema**:
```sql
Fields:
- id: UUID (PK)
- phase_execution_id: UUID (FK → PhaseExecution)
- project_name: string
- project_path: string
- status: enum (pending, running, success, failed, skipped)
- started_at: timestamp (nullable)
- completed_at: timestamp (nullable)

Indexes:
- (phase_execution_id, project_name) - Unique constraint
```

**Relationships**:
- Belongs to PhaseExecution (N:1)
- Has many StepExecutions (1:N)

#### StepExecution

**Owner**: Pipeline Component
**Purpose**: Tracks execution of a single step within a task

**Schema**:
```sql
Fields:
- id: UUID (PK)
- task_execution_id: UUID (FK → TaskExecution)
- step_name: string
- action: string (e.g., "earthly")
- target: string (e.g., "+test")
- status: enum (pending, running, success, failed, skipped)
- started_at: timestamp (nullable)
- completed_at: timestamp (nullable)
- exit_code: integer (nullable)
- logs_s3_key: string (nullable)

Indexes:
- (task_execution_id, step_name) - Unique constraint
```

**Relationships**:
- Belongs to TaskExecution (N:1)

### Release Entities

These entities represent immutable snapshots of projects and their deployment lifecycle.

#### Release

**Owner**: Release Component
**Purpose**: Immutable snapshot of a project at a specific commit

**Schema**:
```sql
Fields:
- id: UUID (PK)
- repository: string
- project: string
- commit_sha: string (canonical identifier)
- release_number: integer (auto-incrementing per project)
- pipeline_run_id: UUID (FK → PipelineRun, nullable)
- oci_uri: string (Release OCI image reference)
- oci_digest: string (SHA256 digest)
- created_at: timestamp
- created_by: string (actor identifier)

Indexes:
- (repository, project, commit_sha) - Unique constraint (canonical triplet)
- (repository, project, release_number) - Unique constraint
- (created_at) - Chronological listing

Constraints:
- release_number auto-increments within (repository, project) scope
```

**Relationships**:
- Optionally references PipelineRun (N:0..1) - pipeline_run_id may be null for manually created releases
- Has many ReleaseAliases (1:N)
- Has one ReleaseTrigger (1:1)
- Has many ReleaseArtifacts (1:N)
- Has many Deployments (1:N)
- Has many ReleaseApprovals (1:N)

#### ReleaseAlias

**Owner**: Release Component
**Purpose**: Human-readable aliases for releases (tags, branch names, etc.)

**Schema**:
```sql
Fields:
- id: UUID (PK)
- release_id: UUID (FK → Release)
- alias: string
- alias_type: enum (numeric, tag, branch, custom)

Indexes:
- (release_id, alias) - Unique constraint
- (alias, alias_type) - Lookup by alias
```

**Relationships**:
- Belongs to Release (N:1)

#### ReleaseTrigger

**Owner**: Release Component
**Purpose**: Records what triggered the release creation

**Schema**:
```sql
Fields:
- release_id: UUID (PK, FK → Release)
- trigger_type: enum (branch_push, tag, manual)
- branch: string (nullable)
- tag: string (nullable)
- triggered_by: string (actor identifier)
- triggered_at: timestamp
```

**Relationships**:
- Belongs to Release (1:1)

#### ReleaseArtifact

**Owner**: Artifacts Component (contributes to Release domain)
**Purpose**: Links releases to their build artifacts

**Schema**:
```sql
Fields:
- id: UUID (PK)
- release_id: UUID (FK → Release)
- artifact_name: string
- artifact_type: enum (container, binary, archive)
- tracking_oci_uri: string (Artifact OCI tracking image)
- tracking_oci_digest: string (SHA256 digest)
- primary_published_uri: string (main registry location)
- created_at: timestamp

Indexes:
- (release_id, artifact_name) - Unique constraint
```

**Relationships**:
- Belongs to Release (N:1)

**Note**: Full artifact metadata (build info, provenance, SBOM) stored in Artifact OCI tracking image, not in database.

### Deployment Entities

These entities track deployments across environments.

#### Deployment

**Owner**: Release Component (shared with Pipeline)
**Purpose**: Tracks deployment of a release to an environment

**Schema**:
```sql
Fields:
- id: UUID (PK)
- release_id: UUID (FK → Release)
- environment: string
- status: enum (pending, active, superseded)
- deployed_at: timestamp
- deployed_by: string (actor identifier)
- gitops_commit_sha: string (GitOps repo commit)

Indexes:
- (environment, release_id) - Track deployments per environment
- (environment, status) - Query active deployment
- (release_id, environment) - Unique active deployment per env

Constraints:
- Only one deployment per (environment, release_id) with status='active'
```

**Relationships**:
- Belongs to Release (N:1)
- May have one ReleaseApproval (0..1:1)

#### ReleaseApproval

**Owner**: Release Component
**Purpose**: Records approval for deploying release to environment

**Schema**:
```sql
Fields:
- id: UUID (PK)
- release_id: UUID (FK → Release)
- environment: string
- approver: string (Keycloak identity)
- approved_at: timestamp
- expires_at: timestamp (nullable)
- justification: text
- status: enum (active, expired, revoked)

Indexes:
- (release_id, environment) - Lookup approval for deployment
- (approver, approved_at) - Audit trail
```

**Relationships**:
- Belongs to Release (N:1)
- May be referenced by Deployment (1:0..1)

## Entity Relationship Diagrams

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

**Note**: Releases can be created either through automated pipelines (pipeline_run_id populated) or manually via CLI (pipeline_run_id is null).

### Cross-Component Relationships

```
PipelineRun → Release (optionally creates)
Release → ReleaseArtifact (contains)
Release → Deployment (deployed as)
Deployment → ReleaseApproval (optionally requires)
```

## API Specifications by Domain

All APIs require Keycloak authentication. See [Core Architecture: Authentication](01-core-architecture.md#authentication--authorization-model).

### Pipeline APIs

**Base Path**: `/v1`

#### Pipeline Management

**Create Pipeline Run**:
```http
POST /v1/runs
Authorization: Bearer {keycloak-token}
Content-Type: application/json

Request Body:
{
  "repository": "string",
  "branch": "string",
  "commit_sha": "string",
  "triggered_by": "string",
  "trigger_type": "push" | "pull_request" | "manual"
}

Response (201 Created):
{
  "id": "uuid",
  "status": "pending",
  "started_at": "iso8601-timestamp"
}
```

**Get Pipeline Run**:
```http
GET /v1/runs/{id}
Authorization: Bearer {keycloak-token}

Response (200 OK):
{
  "id": "uuid",
  "repository": "string",
  "branch": "string",
  "commit_sha": "string",
  "status": "running" | "success" | "failed",
  "started_at": "iso8601-timestamp",
  "completed_at": "iso8601-timestamp",
  "phases": [
    {
      "id": "uuid",
      "name": "test",
      "status": "success",
      "tasks": [...]
    }
  ]
}
```

**Cancel Pipeline Run**:
```http
DELETE /v1/runs/{id}
Authorization: Bearer {keycloak-token}

Response (204 No Content)
```

#### Execution Updates (Internal APIs)

**Update Task Status**:
```http
PUT /v1/tasks/{id}/status
Authorization: Bearer {keycloak-service-account-token}
Content-Type: application/json

Request Body:
{
  "status": "running" | "success" | "failed",
  "error": "string (optional)"
}

Response (200 OK)
```

**Update Step Status**:
```http
PUT /v1/steps/{id}/status
Authorization: Bearer {keycloak-service-account-token}
Content-Type: application/json

Request Body:
{
  "status": "running" | "success" | "failed",
  "exit_code": 0,
  "logs_s3_key": "string"
}

Response (200 OK)
```

### Release APIs

**Base Path**: `/v1`

#### Release Management

**Create Release**:
```http
POST /v1/releases
Authorization: Bearer {keycloak-token}
Content-Type: application/json

Request Body:
{
  "pipeline_run_id": "uuid (optional)",
  "repository": "string",
  "project": "string",
  "commit_sha": "string",
  "artifacts": [
    {
      "name": "string",
      "type": "container" | "binary" | "archive",
      "tracking_oci_uri": "string",
      "primary_published_uri": "string"
    }
  ]
}

Response (201 Created):
{
  "id": "uuid",
  "release_number": 42,
  "oci_uri": "string",
  "canonical_id": "repository/project/commit-sha"
}
```

**Get Release**:
```http
GET /v1/releases/{repo}/{project}/{release-id}
Authorization: Bearer {keycloak-token}

# release-id can be: commit-sha, numeric ID, tag, or alias

Response (200 OK):
{
  "id": "uuid",
  "repository": "string",
  "project": "string",
  "commit_sha": "string",
  "release_number": 42,
  "oci_uri": "string",
  "created_at": "iso8601-timestamp",
  "artifacts": [...],
  "aliases": [...]
}
```

**List Releases for Project**:
```http
GET /v1/releases/{repo}/{project}
Authorization: Bearer {keycloak-token}
Query Parameters:
  - limit: integer (default 20)
  - offset: integer (default 0)

Response (200 OK):
{
  "releases": [...],
  "total": 150,
  "limit": 20,
  "offset": 0
}
```

### Deployment APIs

**Base Path**: `/v1`

#### Deployment Management

**Create Deployment**:
```http
POST /v1/deployments
Authorization: Bearer {keycloak-token}
Content-Type: application/json

Request Body:
{
  "release_id": "uuid",
  "environment": "dev" | "staging" | "production",
  "approval_id": "uuid (optional, required for production)"
}

Response (201 Created):
{
  "id": "uuid",
  "deployment_id": "string",
  "gitops_commit_sha": "string",
  "status": "pending"
}
```

**Get Current Deployment**:
```http
GET /v1/deployments/{environment}/{repo}/{project}
Authorization: Bearer {keycloak-token}

Response (200 OK):
{
  "deployment_id": "uuid",
  "release_id": "uuid",
  "release_number": 42,
  "environment": "production",
  "status": "active",
  "deployed_at": "iso8601-timestamp",
  "deployed_by": "string"
}
```

#### Approval Management

**Create Approval**:
```http
POST /v1/approvals
Authorization: Bearer {keycloak-token}
Content-Type: application/json

Request Body:
{
  "release_id": "uuid",
  "environment": "production",
  "justification": "string"
}

Response (201 Created):
{
  "approval_id": "uuid",
  "expires_at": "iso8601-timestamp"
}
```

**Get Approval Status**:
```http
GET /v1/approvals/{release-id}/{environment}
Authorization: Bearer {keycloak-token}

Response (200 OK):
{
  "approval_id": "uuid",
  "approver": "string",
  "approved_at": "iso8601-timestamp",
  "status": "active" | "expired" | "revoked"
}
```

### Internal Service APIs

Internal service-to-service communication uses Keycloak service account tokens. See [Core Architecture: Authentication - Internal Services](01-core-architecture.md#authentication-boundaries).

## Integration Points

### Entity Storage Patterns
For database technology and connection patterns, see [Core Architecture: Technology Stack](01-core-architecture.md#technology-stack--decisions).

### API Interaction Patterns
For component-to-component API interactions, see [Integration Contracts](07-integration-contracts.md).

### Authentication
All APIs validate Keycloak tokens. See [Core Architecture: Authentication](01-core-architecture.md#authentication--authorization-model).

## Event Schemas

The platform emits events for key lifecycle transitions. Events are published to AWS EventBridge.

### Pipeline Events

**pipeline.run.created**:
```json
{
  "event_type": "pipeline.run.created",
  "timestamp": "iso8601",
  "data": {
    "run_id": "uuid",
    "repository": "string",
    "commit_sha": "string"
  }
}
```

**pipeline.run.completed**:
```json
{
  "event_type": "pipeline.run.completed",
  "timestamp": "iso8601",
  "data": {
    "run_id": "uuid",
    "status": "success" | "failed",
    "duration_seconds": 120
  }
}
```

### Release Events

**release.created**:
```json
{
  "event_type": "release.created",
  "timestamp": "iso8601",
  "data": {
    "release_id": "uuid",
    "repository": "string",
    "project": "string",
    "commit_sha": "string",
    "release_number": 42
  }
}
```

**deployment.created**:
```json
{
  "event_type": "deployment.created",
  "timestamp": "iso8601",
  "data": {
    "deployment_id": "uuid",
    "release_id": "uuid",
    "environment": "production",
    "deployed_by": "string"
  }
}
```

## Error Response Formats

All API errors follow consistent format:

**Standard Error Response**:
```json
{
  "error": {
    "code": "string",
    "message": "string",
    "details": {
      // Additional context (optional)
    }
  }
}
```

**Common Error Codes**:
- `authentication_required` (401)
- `permission_denied` (403)
- `not_found` (404)
- `validation_error` (422)
- `conflict` (409)
- `internal_error` (500)

## Examples

### Complete Pipeline Run with Nested Entities

```json
{
  "id": "abc-123",
  "repository": "company/monorepo",
  "branch": "main",
  "commit_sha": "def456",
  "status": "success",
  "started_at": "2024-01-01T00:00:00Z",
  "completed_at": "2024-01-01T00:15:00Z",
  "phases": [
    {
      "id": "phase-1",
      "name": "test",
      "group_number": 1,
      "status": "success",
      "tasks": [
        {
          "id": "task-1",
          "project_name": "auth-service",
          "status": "success",
          "steps": [
            {
              "id": "step-1",
              "step_name": "unit",
              "action": "earthly",
              "target": "+test",
              "status": "success",
              "exit_code": 0
            }
          ]
        }
      ]
    }
  ]
}
```

### Complete Release with Artifacts

```json
{
  "id": "release-123",
  "repository": "company/monorepo",
  "project": "auth-service",
  "commit_sha": "def456",
  "release_number": 42,
  "oci_uri": "registry.io/company/monorepo/auth-service:def456",
  "artifacts": [
    {
      "id": "artifact-1",
      "artifact_name": "api-server",
      "artifact_type": "container",
      "tracking_oci_uri": "tracking.io/company/monorepo/auth-service/api-server:sha-def456",
      "primary_published_uri": "ecr.aws/company/auth-service:v1.2.3"
    }
  ],
  "aliases": [
    {"alias": "42", "alias_type": "numeric"},
    {"alias": "v1.2.3", "alias_type": "tag"}
  ]
}
```

## Constraints & Limitations

### Entity Constraints

- **PipelineRun**: Maximum 1000 concurrent runs per repository
- **Release**: Release numbers auto-increment per project (no gaps on failure)
- **Deployment**: Only one active deployment per environment/project
- **ReleaseApproval**: Approvals expire after 24 hours (configurable)

### API Rate Limits

- **Pipeline Creation**: 100 requests/minute per repository
- **Release Lookup**: 1000 requests/minute globally
- **Deployment Creation**: 10 requests/minute per environment

### Data Retention

- **Pipeline Runs**: 90 days (configurable)
- **Releases**: Indefinite (immutable audit trail)
- **Deployments**: Indefinite
- **Logs**: 30 days in S3

## Future Enhancements

### Planned Entity Extensions

**Multi-Project Releases**:
- Support releasing multiple projects atomically
- Release groups with coordinated versioning

**Deployment Strategies**:
- Canary deployments with traffic splitting
- Blue-green deployment tracking
- Progressive rollout metrics

**Enhanced Approvals**:
- Multi-party approval requirements
- Conditional approval policies
- Automated approval based on test results

### Planned API Extensions

**Webhooks**:
- Event subscriptions for external systems
- Configurable webhook endpoints

**GraphQL API**:
- Efficient nested entity queries
- Real-time subscriptions

**Bulk Operations**:
- Batch deployment creation
- Batch approval management

---

**Component Owner**: Platform Core (Entity Catalog)
**Contributing Components**: All components contribute their entities and APIs
**Integration Contracts**: See [Integration Contracts](07-integration-contracts.md)
**Last Updated**: 2025-10-02
