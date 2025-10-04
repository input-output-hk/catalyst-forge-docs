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

All build work executes on remote Earthly runners. Worker service maintains persistent connections to Earthly infrastructure, eliminating connection overhead and maximizing throughput.

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

**Worker Service**
- Persistent pods with cached repositories and warm Earthly connections
- Discovery Handler: Performs repository discovery with cached checkouts
- CI Handler: Executes Earthly targets with pre-established connections
- Consumes jobs from SQS queues
- Reports status to DynamoDB and API

**AWS Infrastructure**
- SQS: Distributes jobs to worker service
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

Dispatchers submit jobs to SQS and poll DynamoDB for completion, while actual work happens in the Worker service.

### Worker Service Architecture

The Worker service provides unified execution for all platform job types, maintaining efficiency through shared resources and intelligent routing.

#### Unified Worker Design

The Worker service consolidates all asynchronous execution into a single codebase that:
- Polls SQS queues for jobs
- Routes to appropriate internal handlers based on job type
- Maintains a shared Git repository cache
- Manages Earthly connections for CI operations
- Reports status to DynamoDB and S3

**Internal Job Handlers**:
- **Discovery Handler**: Traverses repositories, parses CUE configurations, evaluates release triggers
- **CI Handler**: Executes Earthly targets for test and build phases
- **Artifact Handler**: Orchestrates producers and publishers, generates SBOMs, creates tracking images
- **Release Handler**: Renders CUE to Kubernetes resources, packages Release OCI images
- **Deployment Handler**: Updates GitOps repository pointer files

#### Queue Consumption Pattern

Each worker StatefulSet is configured to poll from exactly one SQS queue:

```yaml
# Worker environment configuration
worker-discovery:
  env:
    QUEUE_URL: "https://sqs.region.amazonaws.com/account/pipeline-discovery"
    WORKER_TYPE: "discovery"

worker-ci:
  env:
    QUEUE_URL: "https://sqs.region.amazonaws.com/account/pipeline-tasks"
    WORKER_TYPE: "ci"

worker-artifact:
  env:
    QUEUE_URL: "https://sqs.region.amazonaws.com/account/pipeline-artifacts"
    WORKER_TYPE: "artifact"

worker-release:
  env:
    QUEUE_URL: "https://sqs.region.amazonaws.com/account/pipeline-releases"
    WORKER_TYPE: "release"
```

**Message Processing**:
```go
func (w *Worker) processMessages() {
    for {
        // Long poll from designated queue
        result, err := w.sqsClient.ReceiveMessage(ctx, &sqs.ReceiveMessageInput{
            QueueUrl:         &w.queueURL,
            WaitTimeSeconds:  20,
            MaxNumberOfMessages: 1,
        })

        if err != nil || len(result.Messages) == 0 {
            continue
        }

        message := result.Messages[0]
        job := parseJob(message.Body)

        // Process based on worker type
        switch w.workerType {
        case "discovery":
            w.handleDiscoveryJob(job)
        case "ci":
            w.handleCIJob(job)
        case "artifact":
            w.handleArtifactJob(job)  // Build + publish atomic
        case "release":
            w.handleReleaseJob(job)  // Includes deployment
        }

        // Delete message on success
        w.sqsClient.DeleteMessage(ctx, &sqs.DeleteMessageInput{
            QueueUrl:      &w.queueURL,
            ReceiptHandle: message.ReceiptHandle,
        })
    }
}
```

This ensures workers only process jobs appropriate for their resource allocation and configuration.

#### Deployment Strategy

The Worker service deploys as multiple Kubernetes **StatefulSets** to provide persistent Git caches:

```yaml
# Discovery workloads - memory optimized for Git operations
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: worker-discovery
spec:
  replicas: 3
  serviceName: worker-discovery
  volumeClaimTemplates:
  - metadata:
      name: git-cache
    spec:
      accessModes: ["ReadWriteOnce"]
      storageClassName: gp3
      resources:
        requests:
          storage: 50Gi
  template:
    spec:
      containers:
      - name: worker
        resources:
          requests: { cpu: 1, memory: 2Gi }
          limits: { cpu: 2, memory: 4Gi }
        volumeMounts:
        - name: git-cache
          mountPath: /cache/repos

# CI workloads - CPU/memory intensive for builds
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: worker-ci
spec:
  replicas: 10
  serviceName: worker-ci
  volumeClaimTemplates:
  - metadata:
      name: git-cache
    spec:
      accessModes: ["ReadWriteOnce"]
      storageClassName: gp3
      resources:
        requests:
          storage: 200Gi
  template:
    spec:
      containers:
      - name: worker
        resources:
          requests: { cpu: 4, memory: 8Gi }
          limits: { cpu: 8, memory: 16Gi }
        volumeMounts:
        - name: git-cache
          mountPath: /cache/repos

# Artifact workloads - CPU/memory intensive like CI, with Docker daemon
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: worker-artifact
spec:
  replicas: 5
  serviceName: worker-artifact
  volumeClaimTemplates:
  - metadata:
      name: git-cache
    spec:
      accessModes: ["ReadWriteOnce"]
      storageClassName: gp3
      resources:
        requests:
          storage: 150Gi
  template:
    spec:
      containers:
      - name: worker
        resources:
          requests: { cpu: 3, memory: 6Gi }
          limits: { cpu: 6, memory: 12Gi }
        volumeMounts:
        - name: git-cache
          mountPath: /cache/repos
        - name: docker-socket
          mountPath: /var/run/docker.sock  # For Docker daemon access
      volumes:
      - name: docker-socket
        hostPath:
          path: /var/run/docker.sock

# Release workloads - balanced for packaging operations
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: worker-release
spec:
  replicas: 3
  serviceName: worker-release
  volumeClaimTemplates:
  - metadata:
      name: git-cache
    spec:
      accessModes: ["ReadWriteOnce"]
      storageClassName: gp3
      resources:
        requests:
          storage: 100Gi
  template:
    spec:
      containers:
      - name: worker
        resources:
          requests: { cpu: 2, memory: 4Gi }
          limits: { cpu: 4, memory: 8Gi }
        volumeMounts:
        - name: git-cache
          mountPath: /cache/repos
```

**StatefulSet Advantages**:
- Each worker pod gets a stable persistent volume (e.g., `worker-ci-0`, `worker-ci-1`)
- Git caches persist across pod restarts and updates
- Cache gradually warms up as workers process jobs for different repositories
- With 10-15 total repositories, each worker will cache most/all repos over time

#### Persistent Git Cache

Each worker maintains its own persistent Git repository cache:

**Cache Characteristics**:
- Individual persistent volumes per worker pod (via StatefulSet)
- Bare repositories to minimize storage
- Worktrees for concurrent operations
- Cache persists across pod restarts and rolling updates

**Cache Operations**:
```bash
# On job start, worker checks if repo exists
if [ ! -d /cache/repos/$REPO.git ]; then
    git clone --bare $REPO_URL /cache/repos/$REPO.git
fi

# Fetch latest changes for the specific commit
cd /cache/repos/$REPO.git
git fetch origin $COMMIT_SHA

# Create worktree for this job
git worktree add /workspace/$JOB_ID $COMMIT_SHA

# After job completion
git worktree remove /workspace/$JOB_ID
```

**Cache Warmup**:
- Initial jobs experience cache misses (clone required)
- Within hours, most workers cache all active repositories
- Subsequent jobs hit cache ~90% of time (fetch only)
- Rolling updates preserve caches (StatefulSet persistent volumes)

**Storage Sizing**:
- Discovery workers: 50Gi (multiple small repos)
- CI workers: 200Gi (monorepos with history)
- Release workers: 100Gi (moderate usage)

#### Scaling with KEDA

Each worker StatefulSet scales independently based on its queue depth:

```yaml
# Discovery worker scaler
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: worker-discovery-scaler
spec:
  scaleTargetRef:
    name: worker-discovery
  minReplicaCount: 1
  maxReplicaCount: 5
  triggers:
  - type: aws-sqs-queue
    metadata:
      queueURL: https://sqs.region.amazonaws.com/account/pipeline-discovery
      queueLength: "5"  # Target 5 messages per worker

# CI worker scaler
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: worker-ci-scaler
spec:
  scaleTargetRef:
    name: worker-ci
  minReplicaCount: 2
  maxReplicaCount: 20
  triggers:
  - type: aws-sqs-queue
    metadata:
      queueURL: https://sqs.region.amazonaws.com/account/pipeline-tasks
      queueLength: "2"  # Target 2 messages per worker (builds are slow)

# Artifact worker scaler
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: worker-artifact-scaler
spec:
  scaleTargetRef:
    name: worker-artifact
  minReplicaCount: 1
  maxReplicaCount: 10
  triggers:
  - type: aws-sqs-queue
    metadata:
      queueURL: https://sqs.region.amazonaws.com/account/pipeline-artifacts
      queueLength: "2"  # Target 2 messages per worker (build + publish)

# Release worker scaler
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: worker-release-scaler
spec:
  scaleTargetRef:
    name: worker-release
  minReplicaCount: 1
  maxReplicaCount: 10
  triggers:
  - type: aws-sqs-queue
    metadata:
      queueURL: https://sqs.region.amazonaws.com/account/pipeline-releases
      queueLength: "3"  # Target 3 messages per worker
```

Independent scaling ensures each job type gets appropriate resources without over-provisioning.

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

Argo Workflows run lightweight dispatcher pods that interface between orchestration and execution. The dispatcher is a **single platform-provided Go binary** that routes jobs to appropriate queues based on job type.

**Queue Routing Logic**:
```go
func (d *Dispatcher) submitJob(job Job) error {
    var queueURL string

    switch job.Type {
    case JobTypeDiscovery:
        queueURL = d.discoveryQueueURL  // pipeline-discovery
    case JobTypeCI:
        queueURL = d.taskQueueURL       // pipeline-tasks
    case JobTypeArtifact:
        queueURL = d.artifactQueueURL   // pipeline-artifacts
    case JobTypeRelease, JobTypeDeployment:
        queueURL = d.releaseQueueURL    // pipeline-releases
    default:
        return fmt.Errorf("unknown job type: %s", job.Type)
    }

    return d.sqsClient.SendMessage(ctx, &sqs.SendMessageInput{
        QueueUrl:    &queueURL,
        MessageBody: aws.String(job.ToJSON()),
    })
}
```

**Dispatcher Configuration via Environment Variables**:
```yaml
- name: dispatcher
  container:
    image: forge.io/platform/dispatcher:v1.0.0
    env:
    - name: DISCOVERY_QUEUE_URL
      value: "https://sqs.region.amazonaws.com/account/pipeline-discovery"
    - name: TASK_QUEUE_URL
      value: "https://sqs.region.amazonaws.com/account/pipeline-tasks"
    - name: ARTIFACT_QUEUE_URL
      value: "https://sqs.region.amazonaws.com/account/pipeline-artifacts"
    - name: RELEASE_QUEUE_URL
      value: "https://sqs.region.amazonaws.com/account/pipeline-releases"
    - name: JOB_TYPE
      value: "discovery"  # Set by Argo based on workflow stage
```

#### Release Phase Orchestration

The release phase requires special orchestration to handle artifact parallelization:

**Workflow Structure**:
```
1. Dispatcher evaluates release triggers from discovery
2. For each triggered project:
   a. Queue all artifact jobs in parallel
   b. Wait for all artifacts to complete
   c. Collect artifact metadata from S3
   d. Queue release job with artifact data
   e. Wait for release completion
```

**Multi-Job Dispatcher Logic**:
```go
func (d *Dispatcher) executeReleasePhase(project Project) error {
    // Phase 1: Parallel artifact creation
    artifactJobs := []JobResult{}
    for _, artifact := range project.Artifacts {
        jobID := d.queueJob(JobTypeArtifact, artifact)
        artifactJobs = append(artifactJobs, JobResult{ID: jobID})
    }

    // Phase 2: Wait and collect results
    artifactData := []ArtifactResult{}
    for _, job := range artifactJobs {
        result := d.waitForJobWithData(job.ID)
        artifactData = append(artifactData, result)
    }

    // Phase 3: Create release with artifact data
    releaseJobID := d.queueJob(JobTypeRelease, ReleasePayload{
        Project:   project,
        Artifacts: artifactData,
    })

    // Phase 4: Wait for release completion
    return d.waitForJob(releaseJobID)
}
```

This ensures artifacts build in parallel while maintaining data flow to the release step.

### AWS Infrastructure

For AWS authentication patterns, see [Core Architecture: Authentication & Authorization Model](01-core-architecture.md#authentication--authorization-model).

#### SQS Configuration

The platform uses four separate SQS queues for different job types:

```yaml
queues:
  # Discovery operations - project discovery and configuration parsing
  pipeline-discovery:
    name: pipeline-discovery
    visibilityTimeout: 300  # 5 minutes
    messageRetention: 3600  # 1 hour
    consumers: worker-discovery StatefulSet

  # CI operations - test and build phase execution
  pipeline-tasks:
    name: pipeline-tasks
    visibilityTimeout: 1800  # 30 minutes (builds can be slow)
    messageRetention: 7200  # 2 hours
    consumers: worker-ci StatefulSet

  # Artifact operations - build and publish artifacts atomically
  pipeline-artifacts:
    name: pipeline-artifacts
    visibilityTimeout: 1200  # 20 minutes (build + multi-publisher)
    messageRetention: 3600  # 1 hour
    consumers: worker-artifact StatefulSet

  # Release operations - release packaging and deployment
  pipeline-releases:
    name: pipeline-releases
    visibilityTimeout: 900   # 15 minutes
    messageRetention: 3600  # 1 hour
    consumers: worker-release StatefulSet
```

**Queue Selection Logic**:
- `discovery` jobs → `pipeline-discovery`
- `ci` jobs (test, build phases) → `pipeline-tasks`
- `artifact` jobs (build + publish atomic) → `pipeline-artifacts`
- `release` and `deployment` jobs → `pipeline-releases`

**Parallelization**: The artifact queue enables parallel building of multiple artifacts during the release phase, with each worker handling one complete artifact lifecycle (build + publish).

### Result Handling Patterns

The platform uses two distinct patterns for job result handling:

#### Data-Returning Jobs

Jobs that produce data needed by subsequent workflow steps:

**Discovery Jobs**:
- Output: Project list, phase groups, release triggers
- Storage: S3 at `pipeline-runs/{run_id}/discovery/output.json`
- Usage: Argo Workflows uses for dynamic workflow generation

**Artifact Jobs**:
- Output: Artifact URIs, tracking image references, metadata
- Storage: S3 at `pipeline-runs/{run_id}/artifacts/{project}/{artifact}/result.json`
- Usage: Release job uses to create Release entity

**Flow**:
```
Worker → S3 (result.json) → DynamoDB (status + S3 location)
Dispatcher polls DynamoDB → Fetches from S3 → Returns data to Argo
```

#### Status-Only Jobs

Jobs that perform operations and update domain entities:

**CI Tasks** (test, build phases):
- Updates: StepExecution entities via Platform API
- Returns: Success/failure status only

**Release Jobs**:
- Updates: Release entity via Platform API
- Returns: Success/failure status only

**Deployment Jobs**:
- Updates: Deployment entity via Platform API
- Returns: Success/failure status only

**Flow**:
```
Worker → Platform API (entity updates) → DynamoDB (status only)
Dispatcher polls DynamoDB → Returns status to Argo
```

#### DynamoDB Schema

```
Table: pipeline-jobs
Fields:
- job_id: string (PK)
- status: enum (pending, running, completed, failed)
- error_message: string (optional - for failures)
- result_location: string (optional - S3 URI for data-returning jobs)
- updated_at: timestamp
- ttl: timestamp (auto-cleanup after 7 days)
```

#### S3 Storage

**Bucket Structure**:
```
forge-platform-bucket/
└── pipeline-runs/
    └── {run_id}/
        ├── discovery/
        │   └── output.json          # Discovery results
        ├── artifacts/
        │   └── {project}/{artifact}/
        │       └── result.json       # Artifact metadata
        └── logs/
            └── {phase}/{project}/{step}.log  # Execution logs
```

**Lifecycle Management**:

AWS S3 Lifecycle Policies handle automatic cleanup:

```yaml
lifecycle_rules:
  - id: discovery-cleanup
    prefix: pipeline-runs/*/discovery/
    expiration:
      days: 7

  - id: artifact-results-cleanup
    prefix: pipeline-runs/*/artifacts/
    expiration:
      days: 7

  - id: logs-cleanup
    prefix: pipeline-runs/*/logs/
    expiration:
      days: 90
```

**Access Patterns**:
- Workers write directly using IRSA credentials
- Platform API generates presigned URLs for user access
- Dispatchers read results using IRSA credentials

No custom cleanup service required - AWS manages lifecycle automatically.

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
   - Worker service (discovery handler) picks up job
   - Worker uses cached repo to perform discovery
   - Worker writes result to S3, updates DynamoDB
   - Dispatcher polls, retrieves result, sets as output parameter
   - API updates GitHub status: "Discovering projects..."

3. **Phase Execution**
   - Workflow iterates through phase groups sequentially
   - API updates GitHub status: "Running phase X of Y"
   - For each phase, spawns parallel task dispatchers
   - Dispatchers submit all project steps to SQS
   - Worker service (CI handler) executes Earthly targets
   - Workers report status to API and DynamoDB
   - Logs stream directly to S3

4. **Phase Progression**
   - Dispatchers wait for all tasks to complete
   - API updates GitHub status with phase completion
   - Workflow proceeds to next phase group
   - Process repeats until all phases complete

5. **Release Creation** (if triggered)
   - Workflow conditionally executes release dispatchers
   - Artifact jobs submitted to worker service
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

### Worker Service Configuration

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
- Worker service utilization (gauge)
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

**Worker Service Failures:**
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

- Multi-region worker service deployments for disaster recovery
- Alternative execution engines beyond Earthly
- Cross-repository pipeline dependencies
- Predictive scaling based on historical patterns
- Cost optimization through spot instances

### Optimization Opportunities

- Distributed cache using Redis/Elasticache for build artifacts
- Incremental discovery for monorepos
- GPU-enabled workers for ML workloads

---

**Component Owner**: Pipeline & Execution
**Domain Model**: See [Domain Model & API Reference: Pipeline Execution Hierarchy](06-domain-model-api-reference.md#pipeline-execution-hierarchy)
**Last Updated**: 2025-10-02
