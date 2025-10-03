# Execution & Orchestration Specification

## Executive Summary
Defines how work gets executed in the Catalyst Forge platform, including pipeline architecture, workflow orchestration, job distribution, and execution monitoring.

## Core Concepts

### Execution Philosophy

The platform enforces sequential phase execution with fail-fast behavior. Phases execute in group order, with only one phase group active at a time. Any failure immediately terminates the pipeline, aligning with the principle that CI must be "green" before proceeding.

### Dual Persistence Model

The Pipeline Component maintains state in two layers:
- **Argo Workflows**: Volatile execution state for active pipeline runs
- **Database**: Permanent audit trail and user-queryable history

Users interact exclusively through the database layer, having no direct Kubernetes access.

### Earthly Remote Execution

All build work executes on remote Earthly runners. Worker pools maintain persistent connections to Earthly infrastructure, eliminating connection overhead and maximizing throughput.

### OCI Image Types in Pipeline Context

The platform uses two distinct types of OCI images for tracking and deployment. For complete specifications, see [Core Architecture: OCI Image Formats](01-core-architecture.md#oci-image-formats).

**Brief Summary**:
- **Artifact OCI Tracking Images**: Immutable provenance records for individual artifacts (owned by Artifacts Component)
- **Release OCI Images**: Deployable snapshots with Kubernetes resource definitions (owned by Release Component)

Release OCI images reference Artifact OCI tracking images in their metadata layer, creating a complete audit trail from source commit through artifact publication to deployment.

## Architecture

### Pipeline Architecture

The Pipeline Component consists of four primary subsystems:

**API Server**
- REST API for pipeline management
- State persistence to database
- Authentication via GitHub OIDC and service accounts
- Worker coordination endpoints
- Job status synchronization with DynamoDB

**Argo Workflows Engine**
- Orchestrates pipeline execution as Kubernetes-native workflows
- Manages sequential phases with parallel task execution
- Handles retry logic, timeouts, and failure propagation
- Provides built-in UI for workflow visualization
- Runs lightweight dispatcher pods for job submission

**Hot Worker Pools**
- Persistent pods with cached repositories and warm Earthly connections
- Discovery Pool: Performs repository discovery with cached checkouts
- Task Pool: Executes Earthly targets with pre-established connections
- Consumes jobs from SQS queues
- Reports status to DynamoDB and API

**AWS Infrastructure**
- SQS: Distributes jobs to worker pools
- DynamoDB: Tracks job execution status
- S3: Stores discovery outputs, logs, and artifacts
- IAM: Manages service authentication via IRSA

### Workflow Structure

Each pipeline run creates a single Argo Workflow with lightweight dispatcher templates:

```
PipelineRun Workflow (root)
├── Discovery Dispatcher (submits to SQS, polls for result)
├── Phase Loop (sequential execution)
│   └── Task Dispatchers (parallel submission to SQS)
└── Release Dispatchers (conditional, based on discovery)
    └── Artifact Building Dispatchers
```

Dispatchers submit jobs to SQS and poll DynamoDB for completion, while actual work happens in hot worker pools.

### Worker Pool Architecture

#### Hot Discovery Pool

Maintains a fleet of persistent pods optimized for repository discovery:

**Capabilities:**
- Cached repository checkouts using Git bare repositories
- Git worktree management for concurrent operations
- CUE configuration parsing and validation
- Release trigger evaluation

**Optimization Strategies:**
- Maintains bare Git repositories with periodic fetches
- Creates lightweight worktrees for specific commits
- Caches parsed CUE configurations
- Pre-warms popular repository caches

#### Hot Task Pool

Maintains a fleet of persistent pods optimized for Earthly execution:

**Capabilities:**
- Pre-established Earthly runner connections
- Cached repository checkouts
- Certificate and credential caching
- Parallel step execution per project

**Optimization Strategies:**
- Connection pooling to Earthly runners
- Repository cache sharing with discovery pool
- Build cache persistence across executions
- Intelligent work routing based on cache warmth

#### Queue Processing

Workers continuously poll SQS for jobs:

```go
for {
    // Long polling (20 seconds)
    output, err := sqsClient.ReceiveMessage(ctx, &sqs.ReceiveMessageInput{
        QueueUrl:            &queueURL,
        WaitTimeSeconds:     20,
        MaxNumberOfMessages: 1,
    })

    if err == nil && len(output.Messages) > 0 {
        message := output.Messages[0]
        job := parseJob(message)
        result := executeJob(job)

        // Write result to S3
        s3Client.PutObject(ctx, &s3.PutObjectInput{
            Bucket: &bucket,
            Key:    &job.ResultPath,
            Body:   bytes.NewReader(result),
        })

        // Update DynamoDB status
        dynamoClient.UpdateItem(ctx, &dynamodb.UpdateItemInput{
            TableName: aws.String("pipeline-jobs"),
            Key: map[string]types.AttributeValue{
                "job_id": &types.AttributeValueMemberS{Value: job.ID},
            },
            UpdateExpression: aws.String("SET #status = :status, result_location = :location"),
            ExpressionAttributeNames: map[string]string{
                "#status": "status",
            },
            ExpressionAttributeValues: map[string]types.AttributeValue{
                ":status":   &types.AttributeValueMemberS{Value: "completed"},
                ":location": &types.AttributeValueMemberS{Value: s3Path},
            },
        })

        // Delete message
        sqsClient.DeleteMessage(ctx, &sqs.DeleteMessageInput{
            QueueUrl:      &queueURL,
            ReceiptHandle: message.ReceiptHandle,
        })
    }
}
```

### Phase & Step Execution

#### Dynamic Workflow Generation

The Discovery output drives the entire workflow through Argo's dynamic generation:

```json
{
  "repository": {
    "forgeVersion": "v1.0.0",
    "phases": {...},
    "publishers": {...}
  },
  "projects": [
    {
      "name": "string",
      "path": "string",
      "phases": {...},
      "releaseTriggered": false,
      "deploymentTriggered": false
    }
  ],
  "phaseGroups": [
    {
      "group": 1,
      "phases": ["validate", "test"]
    }
  ],
  "releaseEnabled": false,
  "releaseProjects": []
}
```

#### Dispatcher Templates

Argo Workflows run lightweight dispatcher pods that interface between orchestration and execution. These dispatchers are **platform-provided Go binaries** packaged in custom container images.

**Dispatcher Responsibilities**:
- Submit jobs to SQS queues with job specifications
- Poll DynamoDB for job completion status
- Fetch results from S3 when jobs complete
- Return outputs as Argo Workflow parameters
- Update API status for progress tracking

**Discovery Dispatcher Template**:
```yaml
- name: discovery-dispatcher
  container:
    image: forge.io/platform/dispatcher:v1.0.0  # Platform-provided Go binary
    command: ["/dispatcher"]
    args:
      - "discovery"
      - "--repo={{inputs.parameters.repo}}"
      - "--commit={{inputs.parameters.commit}}"
      - "--output=/tmp/discovery.json"
  outputs:
    parameters:
    - name: discovery-output
      valueFrom:
        path: /tmp/discovery.json
```

**Task Dispatcher Template**:
```yaml
- name: task-dispatcher
  container:
    image: forge.io/platform/dispatcher:v1.0.0  # Platform-provided Go binary
    command: ["/dispatcher"]
    args:
      - "task"
      - "--project={{inputs.parameters.project}}"
      - "--phase={{inputs.parameters.phase}}"
      - "--steps={{inputs.parameters.steps}}"
```

### AWS Infrastructure

For AWS authentication patterns, see [Core Architecture: Authentication & Authorization Model](01-core-architecture.md#authentication--authorization-model).

#### SQS Configuration

```yaml
queues:
  discovery:
    name: pipeline-discovery
    visibilityTimeout: 300
    messageRetention: 3600
    maxReceiveCount: 3
  tasks:
    name: pipeline-tasks
    visibilityTimeout: 1800
    messageRetention: 7200
    maxReceiveCount: 3
```

#### DynamoDB Job Tracking

**PipelineJob Table**
```
Fields:
- job_id: string (Primary Key)
- run_id: string
- type: enum (discovery, task, artifact)
- status: enum (pending, running, completed, failed)
- result_location: string (S3 URI)
- created_at: timestamp
- updated_at: timestamp
- ttl: timestamp (auto-cleanup)
```

Configuration:
```yaml
table: pipeline-jobs
billingMode: PAY_PER_REQUEST
ttl: 604800  # 7 days
```

#### S3 Storage

```yaml
buckets:
  results: pipeline-results
  logs: pipeline-logs
lifecycle:
  results: 7d
  logs: 90d
```

## Execution Flow

### Standard Pipeline Execution

1. **Initiation**
   - GitHub Actions sets initial pending status on commit
   - GitHub Actions invokes CLI with repository context
   - CLI calls `POST /v1/runs` with OIDC token and GitHub context
   - API persists PipelineRun entity with GitHub metadata
   - API updates GitHub commit status with Argo UI link
   - API submits Workflow to Argo Server
   - GitHub Actions completes immediately (fire-and-forget)

2. **Discovery**
   - Workflow executes discovery dispatcher
   - Dispatcher submits job to SQS
   - Hot discovery worker picks up job
   - Worker uses cached repo to perform discovery
   - Worker writes result to S3, updates DynamoDB
   - Dispatcher polls, retrieves result, sets as output parameter
   - API updates GitHub status: "Discovering projects..."

3. **Phase Execution**
   - Workflow iterates through phase groups sequentially
   - API updates GitHub status: "Running phase X of Y"
   - For each phase, spawns parallel task dispatchers
   - Dispatchers submit all project steps to SQS
   - Hot workers execute Earthly targets
   - Workers report status to API and DynamoDB
   - Logs stream directly to S3

4. **Phase Progression**
   - Dispatchers wait for all tasks to complete
   - API updates GitHub status with phase completion
   - Workflow proceeds to next phase group
   - Process repeats until all phases complete

5. **Release Creation** (if triggered)
   - Workflow conditionally executes release dispatchers
   - Artifact jobs submitted to worker pool
   - Workers build and publish artifacts
   - Release OCI generated and pushed
   - Optional deployment triggered

6. **Completion**
   - Workflow reaches terminal state
   - API reconciles final status from Argo and DynamoDB
   - API updates GitHub commit status (success/failure/error)
   - If PR workflow, API posts summary comment
   - Database retains complete history
   - DynamoDB entries expire after TTL

### Worker Execution Patterns

**Repository Caching:**
```bash
# Initial setup (once per worker)
git clone --bare $REPO_URL /cache/repos/$REPO_NAME.git

# Per job execution (fast)
cd /cache/repos/$REPO_NAME.git
git fetch origin
git worktree add -f /workspace/$JOB_ID $COMMIT_SHA
cd /workspace/$JOB_ID
# Execute work with fresh checkout
```

**Earthly Connection Management:**
```go
type EarthlyPool struct {
    connections []*EarthlyConnection
    mu          sync.Mutex
}

func NewEarthlyPool(addresses []string) *EarthlyPool {
    pool := &EarthlyPool{
        connections: make([]*EarthlyConnection, 0, len(addresses)),
    }
    for _, addr := range addresses {
        conn := pool.establishConnection(addr)
        pool.connections = append(pool.connections, conn)
    }
    return pool
}

func (p *EarthlyPool) Execute(target, workspace string) error {
    conn := p.getAvailableConnection()
    return conn.Run(target, workspace)
}
```

## Configuration

### Argo Workflows Configuration

```yaml
workflow:
  parallelism: 100  # Max parallel dispatchers
  activeDeadlineSeconds: 86400  # 24-hour max duration
  podGC:
    strategy: OnPodCompletion

templates:
  dispatcher:
    resources:
      requests:
        cpu: 50m
        memory: 128Mi
      limits:
        cpu: 200m
        memory: 256Mi
```

### Worker Pool Configuration

```yaml
discovery-pool:
  replicas: 5
  resources:
    requests:
      cpu: 500m
      memory: 2Gi
    limits:
      cpu: 2000m
      memory: 4Gi
  cache:
    repoPath: /cache/repos
    maxSize: 100Gi

task-pool:
  replicas: 20
  resources:
    requests:
      cpu: 1000m
      memory: 4Gi
    limits:
      cpu: 4000m
      memory: 8Gi
  earthly:
    maxConnections: 50
    connectionTimeout: 30s
```

## Operations

### Performance & Scaling

#### Caching Strategy

**Repository Cache:**
- Bare repositories reduce storage
- Worktrees enable concurrent checkouts
- LRU eviction for space management
- Pre-warming for frequently built repos

**Earthly Cache:**
- Persistent build caches across executions
- Layer caching for Docker builds
- Artifact caching between phases

#### Worker Autoscaling

```yaml
behavior:
  scaleUp:
    policies:
    - type: Pods
      value: 5
      periodSeconds: 60
  scaleDown:
    stabilizationWindowSeconds: 300
metrics:
- type: External
  external:
    metric:
      name: sqs_queue_depth
    target:
      type: AverageValue
      averageValue: "10"
```

### Monitoring & Observability

#### Metrics Collection

**Platform Metrics (Prometheus)**:
- Pipeline run duration (histogram)
- Phase execution time (histogram)
- Queue depth (gauge)
- Worker pool utilization (gauge)
- Cache hit rates (counter)
- Job processing time (histogram)

Platform components expose Prometheus metrics endpoints. Metrics are scraped by Prometheus and pushed to **Grafana Cloud** for visualization and alerting.

**AWS Metrics (CloudWatch)**:
- SQS message age and depth
- DynamoDB read/write capacity
- S3 request rates
- Worker autoscaling metrics

Grafana Cloud maintains a CloudWatch integration that polls AWS metrics periodically for unified observability.

#### Logging

**Application Logs**:
- Platform components log to Loki (via promtail)
- Loki streams logs to **Grafana Cloud** for centralized log aggregation
- Structured logging with consistent format across all components

**Build Logs**:
- Stored in S3 with presigned URLs for access
- Referenced in pipeline execution records

**Audit Logs**:
- Stored in PostgreSQL database
- Queryable via API for compliance and debugging

**Worker Logs**:
- Streamed to Loki for central aggregation
- Structured with job context for correlation

#### Monitoring Tools

- **Grafana Cloud**: Primary observability platform (metrics, logs, dashboards, alerts)
- **Argo UI**: Real-time workflow visualization and debugging
- **API health endpoints**: Readiness and liveness checks
- **GitHub commit status**: Developer-facing pipeline status
- **Pull request comments**: Execution summaries for PR workflows

### Error Handling

#### Failure Modes

**SQS Message Failures:**
- Automatic retries with exponential backoff
- DLQ for persistent failures
- CloudWatch alarms for DLQ depth

**Worker Pool Failures:**
- Kubernetes automatically restarts failed workers
- Jobs returned to queue after visibility timeout
- Circuit breakers for Earthly connections

**Dispatcher Failures:**
- Argo retries failed dispatchers
- Idempotent job submission prevents duplicates
- Timeout enforcement at workflow level

## Integration Points

### GitHub Integration

The platform implements a fire-and-forget pattern with asynchronous status updates to provide immediate developer feedback without blocking GitHub Actions workflows.

#### GitHub Actions Workflow

```yaml
name: Trigger CI Pipeline
on:
  push:
  pull_request:

jobs:
  trigger:
    runs-on: ubuntu-latest
    timeout-minutes: 5
    steps:
    - name: Set Initial Status
      run: |
        gh api --method POST \
          /repos/${{ github.repository }}/statuses/${{ github.sha }} \
          -f state=pending \
          -f description="Pipeline queued..." \
          -f context=ci/catalyst-forge

    - name: Trigger Pipeline
      run: |
        response=$(curl -X POST https://api.catalyst-forge.com/v1/runs \
          -H "Authorization: Bearer ${{ secrets.GITHUB_TOKEN }}" \
          -d '{
            "repository": "${{ github.repository }}",
            "commit_sha": "${{ github.sha }}",
            "branch": "${{ github.ref_name }}",
            "github_context": {
              "pr_number": ${{ github.event.pull_request.number || null }},
              "run_id": "${{ github.run_id }}",
              "context": "ci/catalyst-forge"
            }
          }')
        echo "✅ Pipeline triggered successfully"
    # Workflow completes immediately
```

#### GitHub Status Updates

**API Server Status Updates:**
```go
type GitHubStatusManager struct {
    client *github.Client
    domain string
}

func (m *GitHubStatusManager) UpdatePipelineStatus(ctx context.Context, run *PipelineRun, status, description string) error {
    state := m.mapStatus(status)
    targetURL := fmt.Sprintf("https://argo.%s/workflows/%s", m.domain, run.WorkflowName)

    context := run.GitHubContext.Context
    if context == "" {
        context = "ci/catalyst-forge"
    }

    _, _, err := m.client.Repositories.CreateStatus(ctx,
        run.Repository.Owner,
        run.Repository.Name,
        run.CommitSHA,
        &github.RepoStatus{
            State:       &state,
            TargetURL:   &targetURL,
            Description: &description,
            Context:     &context,
        },
    )
    return err
}

func (m *GitHubStatusManager) mapStatus(internalStatus string) string {
    statusMap := map[string]string{
        "pending":     "pending",
        "discovering": "pending",
        "running":     "pending",
        "succeeded":   "success",
        "failed":      "failure",
        "cancelled":   "error",
    }

    if mapped, ok := statusMap[internalStatus]; ok {
        return mapped
    }
    return "error"
}
```

**Status Update Points:**
- **Pipeline Created**: "Pipeline started - discovering projects..."
- **Discovery Complete**: "Running phase 1 of N"
- **Phase Complete**: "Running phase X of N (Y% complete)"
- **Pipeline Complete**: "Pipeline succeeded in Xm Ys" or failure message

**Pull Request Integration:**
```go
func (m *GitHubStatusManager) PostPRSummary(ctx context.Context, run *PipelineRun) error {
    if run.GitHubContext.PRNumber == 0 {
        return nil // Not a PR workflow
    }

    summary := generatePipelineSummary(run)
    body := fmt.Sprintf(`## Pipeline Results

**Status:** %s
**Duration:** %s
**Logs:** [View in Argo](https://argo.%s/workflows/%s)

%s`,
        run.Status,
        run.Duration.String(),
        m.domain,
        run.WorkflowName,
        summary,
    )

    _, _, err := m.client.Issues.CreateComment(ctx,
        run.Repository.Owner,
        run.Repository.Name,
        run.GitHubContext.PRNumber,
        &github.IssueComment{Body: &body},
    )
    return err
}
```

**GitHub Context Storage:**
```json
{
  "pr_number": 123,
  "run_id": "5678901234",
  "context": "ci/catalyst-forge",
  "check_run_id": "987654321"
}
```

#### Authentication & Triggers

For authentication patterns, see [Core Architecture: Authentication & Authorization Model](01-core-architecture.md#authentication--authorization-model).

- **OIDC**: Authentication for API calls
- **Actions**: Pipeline trigger mechanism
- **Repository Access**: SSH keys for cloning

### AWS Services Integration

- **SQS**: Job distribution with long polling and DLQ
- **DynamoDB**: Real-time job status tracking
- **S3**: Discovery outputs, logs, and artifact storage
- **IAM/IRSA**: Service authentication and authorization
- **CloudWatch**: Metrics and log aggregation
- **Secrets Manager**: CI secret retrieval (see [Core Architecture: Secret Management Patterns](01-core-architecture.md#secret-management-patterns))

### Earthly Remote Runners

- Connection pooling in hot workers
- Persistent build caches
- Automatic retry on connection failure
- Load balancing across runner fleet

### Component Interactions

For complete API specifications, see [Domain Model & API Reference: Pipeline APIs](06-domain-model-api-reference.md#pipeline-apis).

For cross-component integration patterns, see [Integration Contracts](07-integration-contracts.md).

## Reference

### API Endpoints

**Pipeline Management** (see [Domain Model & API Reference](06-domain-model-api-reference.md#pipeline-apis) for complete specifications):
- `POST /v1/runs` - Create pipeline run
- `GET /v1/runs/{id}` - Get pipeline run with nested executions
- `DELETE /v1/runs/{id}` - Cancel pipeline run

**Execution Updates** (Internal):
- `PUT /v1/tasks/{id}/status` - Update task status
- `PUT /v1/steps/{id}/status` - Update step status
- `GET /v1/jobs/{run_id}` - Get job status (proxies to DynamoDB)

### Domain Model

For complete domain entity specifications, see [Domain Model & API Reference: Pipeline Execution Hierarchy](06-domain-model-api-reference.md#pipeline-execution-hierarchy).

**Key Entities:**
- PipelineRun
- PhaseExecution
- TaskExecution
- StepExecution

## Examples

For execution examples and integration patterns, see [Integration Contracts: Complete Pipeline Flow](07-integration-contracts.md#complete-pipeline-flow).

## Constraints & Limitations

### Architectural Constraints

- Sequential phase execution only
- Single AWS region operation
- Earthly-only step execution (initial release)
- No cross-repository dependencies

### Scale Limits

- Maximum 100 projects per repository
- Maximum 50 parallel workers per pool
- Maximum 24-hour pipeline duration
- Maximum 1MB discovery output (S3 limit removed)
- Maximum 1000 concurrent pipelines
- GitHub API rate limits (5000 requests/hour for authenticated requests)

## Future Enhancements

### Planned Improvements

- Multi-region worker pools for disaster recovery
- Alternative execution engines beyond Earthly
- Cross-repository pipeline dependencies
- Predictive scaling based on historical patterns
- Cost optimization through spot instances

### Optimization Opportunities

- Distributed cache using Redis/Elasticache
- P2P cache sharing between workers
- Incremental discovery for monorepos
- GPU-enabled workers for ML workloads

---

**Component Owner**: Pipeline & Execution
**Domain Model**: See [Domain Model & API Reference: Pipeline Execution Hierarchy](06-domain-model-api-reference.md#pipeline-execution-hierarchy)
**Last Updated**: 2025-10-02
