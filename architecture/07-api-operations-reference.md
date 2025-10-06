# API & Operations Reference

## Table of Contents

- [Executive Summary](#executive-summary)
- [Core Concepts](#core-concepts)
  - [Authentication](#authentication)
  - [API Conventions](#api-conventions)
  - [Error Response Format](#error-response-format)
- [Pipeline APIs](#pipeline-apis)
  - [Base Path](#base-path)
  - [Create Pipeline Run](#create-pipeline-run)
  - [Get Pipeline Run](#get-pipeline-run)
  - [List Pipeline Runs](#list-pipeline-runs)
  - [Cancel Pipeline Run](#cancel-pipeline-run)
  - [Internal Pipeline APIs](#internal-pipeline-apis)
- [Release APIs](#release-apis)
  - [Base Path](#base-path-1)
  - [Create Release](#create-release)
  - [Get Release](#get-release)
  - [List Releases for Project](#list-releases-for-project)
- [Deployment APIs](#deployment-apis)
  - [Base Path](#base-path-2)
  - [Create Deployment](#create-deployment)
  - [Get Current Deployment](#get-current-deployment)
  - [Get Deployment History](#get-deployment-history)
- [Approval APIs](#approval-apis)
  - [Base Path](#base-path-3)
  - [Create Approval](#create-approval)
  - [Get Approval Status](#get-approval-status)
- [Event Schemas](#event-schemas)
  - [Pipeline Events](#pipeline-events)
  - [Release Events](#release-events)
  - [Deployment Events](#deployment-events)
- [Operational Procedures](#operational-procedures)
  - [Result Handling](#result-handling)
  - [Log Storage](#log-storage)
  - [Timeout Handling](#timeout-handling)
  - [NATS Message Patterns](#nats-message-patterns)
  - [Monitoring and Observability](#monitoring-and-observability)
- [Integration Points](#integration-points)
  - [External System Integration](#external-system-integration)
  - [Platform Component Integration](#platform-component-integration)
- [Constraints](#constraints)
  - [API Rate Limits](#api-rate-limits)
  - [Request Size Limits](#request-size-limits)
  - [Authentication Requirements](#authentication-requirements)
  - [Operational Boundaries](#operational-boundaries)
- [Future Considerations](#future-considerations)
  - [Potential Enhancements](#potential-enhancements)
  - [API Versioning](#api-versioning)

## Executive Summary

This document provides complete API specifications and operational procedures for the Catalyst Forge platform. All APIs require Keycloak authentication and follow RESTful conventions with JSON payloads. The platform provides APIs for pipeline management, release management, and deployment operations.

Operations include result handling patterns, storage structures, and event schemas for platform integrations.

## Core Concepts

### Authentication

All API requests require Keycloak Bearer tokens:

```http
Authorization: Bearer {keycloak-token}
```

**Token Types:**

**User Tokens:** Obtained through Keycloak authentication (GitHub OIDC, CLI, web console)

**Service Account Tokens:** Used for internal service-to-service communication

See Architecture Overview: Technology Stack for complete authentication architecture.

### API Conventions

**Base URL:** `https://api.catalyst-forge.example.com`

**Versioning:** All endpoints prefixed with `/v1` for version 1

**Content Type:** `application/json` for all request and response bodies

**HTTP Methods:**
- `GET`: Retrieve resources
- `POST`: Create resources
- `PUT`: Update resources
- `DELETE`: Delete or cancel resources

**Standard Status Codes:**
- `200 OK`: Successful request
- `201 Created`: Resource created
- `204 No Content`: Successful deletion
- `400 Bad Request`: Invalid input
- `401 Unauthorized`: Missing or invalid authentication
- `403 Forbidden`: Insufficient permissions
- `404 Not Found`: Resource does not exist
- `409 Conflict`: Resource conflict (duplicate)
- `500 Internal Server Error`: Platform error

### Error Response Format

All API errors follow consistent structure:

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

**Common Error Codes:**
- `invalid_request`: Malformed request body
- `authentication_failed`: Invalid token
- `authorization_failed`: Insufficient permissions
- `resource_not_found`: Entity does not exist
- `resource_conflict`: Duplicate resource
- `internal_error`: Platform error

## Pipeline APIs

### Base Path

`/v1/runs`

### Create Pipeline Run

Initiates a new pipeline execution for a commit.

**Endpoint:**
```http
POST /v1/runs
```

**Request Body:**
```json
{
  "repository": "string",
  "branch": "string",
  "commit_sha": "string",
  "triggered_by": "string",
  "trigger_type": "push" | "pull_request" | "manual"
}
```

**Response (201 Created):**
```json
{
  "id": "uuid",
  "status": "pending",
  "started_at": "iso8601-timestamp"
}
```

**Example:**
```bash
curl -X POST https://api.catalyst-forge.example.com/v1/runs \
  -H "Authorization: Bearer ${TOKEN}" \
  -H "Content-Type: application/json" \
  -d '{
    "repository": "mycompany/myrepo",
    "branch": "main",
    "commit_sha": "abc123def456",
    "triggered_by": "alice@company.com",
    "trigger_type": "push"
  }'
```

### Get Pipeline Run

Retrieves pipeline run details including phase and task status.

**Endpoint:**
```http
GET /v1/runs/{id}
```

**Response (200 OK):**
```json
{
  "id": "uuid",
  "repository": "string",
  "branch": "string",
  "commit_sha": "string",
  "status": "pending" | "running" | "success" | "failed" | "cancelled",
  "triggered_by": "string",
  "trigger_type": "push" | "pull_request" | "manual",
  "started_at": "iso8601-timestamp",
  "completed_at": "iso8601-timestamp",
  "phases": [
    {
      "id": "uuid",
      "name": "string",
      "group_number": 1,
      "status": "pending" | "running" | "success" | "failed" | "skipped",
      "started_at": "iso8601-timestamp",
      "completed_at": "iso8601-timestamp",
      "tasks": [
        {
          "id": "uuid",
          "project_name": "string",
          "status": "pending" | "running" | "success" | "failed",
          "steps": [
            {
              "id": "uuid",
              "name": "string",
              "status": "pending" | "running" | "success" | "failed",
              "exit_code": 0,
              "logs_url": "string"
            }
          ]
        }
      ]
    }
  ]
}
```

### List Pipeline Runs

Retrieves pipeline runs for a repository with pagination.

**Endpoint:**
```http
GET /v1/runs?repository={repo}&branch={branch}&limit={limit}&offset={offset}
```

**Query Parameters:**
- `repository` (required): Repository identifier
- `branch` (optional): Filter by branch
- `limit` (optional, default 20): Results per page
- `offset` (optional, default 0): Pagination offset

**Response (200 OK):**
```json
{
  "runs": [
    {
      "id": "uuid",
      "repository": "string",
      "commit_sha": "string",
      "status": "success",
      "started_at": "iso8601-timestamp"
    }
  ],
  "total": 150,
  "limit": 20,
  "offset": 0
}
```

### Cancel Pipeline Run

Cancels an active pipeline run.

**Endpoint:**
```http
DELETE /v1/runs/{id}
```

**Response (204 No Content)**

### Internal Pipeline APIs

These endpoints are used by Worker Services for status updates.

**Update Task Status:**
```http
PUT /v1/tasks/{id}/status
Authorization: Bearer {service-account-token}
Content-Type: application/json

{
  "status": "running" | "success" | "failed",
  "error": "string (optional)"
}
```

**Update Step Status:**
```http
PUT /v1/steps/{id}/status
Authorization: Bearer {service-account-token}
Content-Type: application/json

{
  "status": "running" | "success" | "failed",
  "exit_code": 0,
  "logs_s3_key": "string"
}
```

## Release APIs

### Base Path

`/v1/releases`

### Create Release

Creates a new release for a project at a specific commit.

**Endpoint:**
```http
POST /v1/releases
```

**Request Body:**
```json
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
```

**Response (201 Created):**
```json
{
  "id": "uuid",
  "release_number": 42,
  "oci_uri": "string",
  "canonical_id": "repository/project/commit-sha",
  "created_at": "iso8601-timestamp"
}
```

### Get Release

Retrieves release details by any identifier (commit SHA, numeric ID, tag, alias).

**Endpoint:**
```http
GET /v1/releases/{repo}/{project}/{release-id}
```

**Path Parameters:**
- `{repo}`: Repository identifier
- `{project}`: Project name
- `{release-id}`: Commit SHA, numeric ID, tag, or alias

**Response (200 OK):**
```json
{
  "id": "uuid",
  "repository": "string",
  "project": "string",
  "commit_sha": "string",
  "release_number": 42,
  "oci_uri": "string",
  "oci_digest": "string",
  "created_at": "iso8601-timestamp",
  "created_by": "string",
  "artifacts": [
    {
      "name": "string",
      "type": "container",
      "tracking_oci_uri": "string",
      "primary_published_uri": "string"
    }
  ],
  "aliases": [
    {
      "alias": "v1.2.3",
      "alias_type": "tag"
    },
    {
      "alias": "42",
      "alias_type": "numeric"
    }
  ],
  "trigger": {
    "trigger_type": "branch_push",
    "branch": "main",
    "triggered_by": "alice@company.com"
  }
}
```

### List Releases for Project

Retrieves releases for a project with pagination.

**Endpoint:**
```http
GET /v1/releases/{repo}/{project}?limit={limit}&offset={offset}
```

**Query Parameters:**
- `limit` (optional, default 20): Results per page
- `offset` (optional, default 0): Pagination offset

**Response (200 OK):**
```json
{
  "releases": [
    {
      "id": "uuid",
      "commit_sha": "string",
      "release_number": 42,
      "created_at": "iso8601-timestamp"
    }
  ],
  "total": 150,
  "limit": 20,
  "offset": 0
}
```

## Deployment APIs

### Base Path

`/v1/deployments`

### Create Deployment

Deploys a release to an environment.

**Endpoint:**
```http
POST /v1/deployments
```

**Request Body:**
```json
{
  "release_id": "uuid",
  "environment": "dev" | "staging" | "production",
  "approval_id": "uuid (optional, required for production)"
}
```

**Response (201 Created):**
```json
{
  "id": "uuid",
  "deployment_id": "string",
  "gitops_commit_sha": "string",
  "status": "pending"
}
```

### Get Current Deployment

Retrieves the current active deployment for a project in an environment.

**Endpoint:**
```http
GET /v1/deployments/{environment}/{repo}/{project}
```

**Response (200 OK):**
```json
{
  "deployment_id": "uuid",
  "release_id": "uuid",
  "release_number": 42,
  "environment": "production",
  "status": "active",
  "deployed_at": "iso8601-timestamp",
  "deployed_by": "string",
  "gitops_commit_sha": "string"
}
```

### Get Deployment History

Retrieves deployment history for a project in an environment.

**Endpoint:**
```http
GET /v1/deployments/{environment}/{repo}/{project}/history?limit={limit}&offset={offset}
```

**Response (200 OK):**
```json
{
  "deployments": [
    {
      "id": "uuid",
      "release_number": 42,
      "status": "superseded",
      "deployed_at": "iso8601-timestamp",
      "deployed_by": "string"
    }
  ],
  "total": 50,
  "limit": 20,
  "offset": 0
}
```

## Approval APIs

### Base Path

`/v1/approvals`

### Create Approval

Approves a release for deployment to an environment.

**Endpoint:**
```http
POST /v1/approvals
```

**Request Body:**
```json
{
  "release_id": "uuid",
  "environment": "production",
  "justification": "string"
}
```

**Response (201 Created):**
```json
{
  "approval_id": "uuid",
  "approver": "string",
  "approved_at": "iso8601-timestamp",
  "expires_at": "iso8601-timestamp",
  "status": "active"
}
```

### Get Approval Status

Retrieves approval status for a release in an environment.

**Endpoint:**
```http
GET /v1/approvals/{release-id}/{environment}
```

**Response (200 OK):**
```json
{
  "approval_id": "uuid",
  "approver": "string",
  "approved_at": "iso8601-timestamp",
  "expires_at": "iso8601-timestamp",
  "status": "active" | "expired" | "revoked",
  "justification": "string"
}
```

**Response (404 Not Found):**
If no approval exists for the release-environment pair.

## Event Schemas

The platform emits events for key lifecycle transitions. Events are published to the platform's event bus for external integrations.

### Pipeline Events

**pipeline.run.created:**
```json
{
  "event_type": "pipeline.run.created",
  "timestamp": "iso8601-timestamp",
  "data": {
    "run_id": "uuid",
    "repository": "string",
    "branch": "string",
    "commit_sha": "string",
    "triggered_by": "string"
  }
}
```

**pipeline.run.completed:**
```json
{
  "event_type": "pipeline.run.completed",
  "timestamp": "iso8601-timestamp",
  "data": {
    "run_id": "uuid",
    "repository": "string",
    "commit_sha": "string",
    "status": "success" | "failed",
    "duration_seconds": 120
  }
}
```

**pipeline.run.failed:**
```json
{
  "event_type": "pipeline.run.failed",
  "timestamp": "iso8601-timestamp",
  "data": {
    "run_id": "uuid",
    "repository": "string",
    "commit_sha": "string",
    "failed_phase": "string",
    "error_message": "string"
  }
}
```

### Release Events

**release.created:**
```json
{
  "event_type": "release.created",
  "timestamp": "iso8601-timestamp",
  "data": {
    "release_id": "uuid",
    "repository": "string",
    "project": "string",
    "commit_sha": "string",
    "release_number": 42,
    "trigger_type": "branch_push" | "tag" | "manual"
  }
}
```

### Deployment Events

**deployment.created:**
```json
{
  "event_type": "deployment.created",
  "timestamp": "iso8601-timestamp",
  "data": {
    "deployment_id": "uuid",
    "release_id": "uuid",
    "environment": "production",
    "deployed_by": "string",
    "gitops_commit_sha": "string"
  }
}
```

**deployment.completed:**
```json
{
  "event_type": "deployment.completed",
  "timestamp": "iso8601-timestamp",
  "data": {
    "deployment_id": "uuid",
    "environment": "production",
    "status": "active",
    "reconciliation_duration_seconds": 45
  }
}
```

## Operational Procedures

### Result Handling

The platform handles job results through size-based routing:

**Small Results (<1MB):**
- Included directly in NATS reply message
- Immediately available to dispatcher
- No additional storage or retrieval required

**Large Results (>1MB):**
- Stored in object storage (S3/MinIO)
- Reply message contains S3 reference
- Dispatcher retrieves from object storage

**Result Storage Pattern:**
```
{bucket}/pipeline-runs/{run_id}/results/{job_type}/{job_id}.json
```

### Log Storage

Step execution logs are stored in object storage:

**Storage Structure:**
```
{bucket}/pipeline-runs/{run_id}/logs/{phase}/{project}/{step}.log
```

**Lifecycle Management:**
- Development logs retained for 7 days
- Staging logs retained for 30 days
- Production logs retained for 90 days
- Lifecycle policies configured per bucket

**Access:**
Logs accessible via presigned URLs returned in step execution metadata.

### Timeout Handling

All jobs have configurable timeouts:

**Default Timeouts:**
- Discovery jobs: 10 minutes
- CI jobs: 30 minutes (configurable per phase)
- Artifact jobs: 20 minutes
- Release jobs: 15 minutes
- Deployment jobs: 10 minutes

**Timeout Behavior:**
1. Dispatcher waits for reply until timeout
2. On timeout, dispatcher returns error to workflow
3. Workflow terminates with failure status
4. Worker may continue processing (orphaned job)
5. NATS message redelivery prevented by max deliver limit

**Timeout Configuration:**
Override defaults in repository configuration:
```cue
phases: {
    build: {
        timeout: "45m"  // Override default
    }
}
```

### NATS Message Patterns

**Request-Reply Pattern:**
```
1. Dispatcher creates unique inbox subscription
2. Dispatcher publishes job with ReplyTo field set to inbox
3. Worker processes job
4. Worker publishes reply to inbox subject
5. Dispatcher receives reply or times out
6. Worker acknowledges NATS message after successful reply
```

**Subject Naming:**
- Discovery: `pipeline.discovery.{run-id}`
- CI: `pipeline.tasks.{run-id}.{task-id}`
- Artifacts: `pipeline.artifacts.{run-id}.{artifact-id}`
- Releases: `pipeline.releases.{run-id}`

**Consumer Configuration:**
- Pull-based consumers (not push)
- Durable consumers per worker type
- Explicit acknowledgment after processing
- Max deliver: 3 attempts before dead letter

### Monitoring and Observability

**Metrics:**
- Pipeline run duration and success rate
- Phase execution times
- Worker queue depth (NATS consumer lag)
- API request latency and error rates

**Logging:**
- Structured JSON logs from all services
- Centralized log aggregation (Grafana Loki)
- Log levels: DEBUG, INFO, WARN, ERROR

**Tracing:**
- Distributed tracing via Istio service mesh
- Request correlation across services
- Trace retention: 7 days

**Dashboards:**
- Pipeline execution metrics
- Worker utilization and scaling
- API performance and errors
- Infrastructure health

## Integration Points

### External System Integration

**GitHub:**
Webhook ingestion for pipeline triggers. API integration for commit status updates. OIDC token exchange for authentication.

**Keycloak:**
Token validation for all API requests. User authentication and service account management. Federation to external identity providers.

**Argo Workflows:**
Workflow submission and status monitoring. Result retrieval from completed workflows.

**Object Storage:**
Presigned URLs for log access. Direct uploads for large artifacts. Lifecycle policies for automated cleanup.

### Platform Component Integration

**Platform API to Worker Service:**
Asynchronous job submission via NATS JetStream. Status updates via REST callbacks. Result persistence in PostgreSQL.

**Platform API to GitHub:**
Commit status updates for pipeline runs. Release tag notifications. Deployment status reporting.

**Worker Service to Platform API:**
Entity updates (PipelineRun, Release, Deployment). Authentication via Keycloak service accounts.

## Constraints

### API Rate Limits

- Maximum 1000 requests per minute per authenticated user
- Maximum 100 concurrent pipeline runs per repository
- Maximum 50 concurrent deployments across all environments

### Request Size Limits

- Maximum request body size: 10MB
- Maximum response body size: 50MB
- Large results (>1MB) stored in object storage with reference URLs

### Authentication Requirements

- All endpoints require valid Keycloak Bearer tokens
- Service account tokens limited to worker service operations
- Token expiration: 60 minutes (access), 30 days (refresh)

### Operational Boundaries

- API availability depends on PostgreSQL and Keycloak availability
- NATS downtime prevents new job submissions (existing jobs continue)
- Pipeline run history retention: 90 days (configurable)

## Future Considerations

### Potential Enhancements

**GraphQL API:**
Unified query interface for complex data retrieval. Subscription support for real-time pipeline status updates.

**Webhooks:**
Configurable webhooks for pipeline events, release notifications, and deployment status. Retry logic and delivery guarantees.

**Enhanced Filtering:**
Advanced query capabilities for pipeline runs and releases. Full-text search across logs and metadata.

**Batch Operations:**
Bulk pipeline cancellation, mass release promotion, and batch approval workflows for operational efficiency.

### API Versioning

The platform supports API evolution through versioning:
- Current: `/v1` endpoints
- Future: `/v2` endpoints for breaking changes
- Deprecation policy: 6-month notice for endpoint removal
- Backward compatibility maintained within major versions

---

**Related Documents:**
- Domain Model - Entity definitions, relationships, and database schemas
- System Architecture - Service architecture, messaging patterns, and data flows
- Developer Guide - Pipeline configuration and deployment workflows

**Last Updated:** 2025-10-05
