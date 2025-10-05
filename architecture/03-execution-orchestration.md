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
- Consumes jobs from NATS JetStream pull consumers
- Reports status via request-reply pattern to dispatcher

For infrastructure-level scaling architecture (KEDA configuration, Karpenter provisioning, node management), see [Platform Infrastructure: Workload Autoscaling Architecture](09-platform-infrastructure.md#workload-autoscaling-architecture).

**NATS Infrastructure**
- JetStream: Distributes jobs to worker service via work queues
- Request-reply: Workers respond directly to dispatcher inboxes
- S3: Stores large results (>1MB), logs, and artifacts
- IAM: Manages service authentication via IRSA (for AWS resources)

### Workflow Structure

Each pipeline run creates a single Argo Workflow with lightweight dispatcher templates:

```
PipelineRun Workflow (root)
├── Discovery Dispatcher (publishes to NATS with reply subject, awaits reply)
├── Phase Loop (sequential execution)
│   └── Task Dispatchers (parallel publish to NATS with reply subjects)
└── Release Dispatchers (conditional, based on discovery)
    └── Artifact Building Dispatchers
```

Dispatchers publish jobs to NATS with reply subjects and await direct replies from workers, while actual work happens in the Worker service.

### Worker Service Architecture

The Worker service provides unified execution for all platform job types, maintaining efficiency through shared resources and intelligent routing.

#### Unified Worker Design

The Worker service consolidates all asynchronous execution into a single codebase that:
- Pulls jobs from NATS JetStream streams
- Routes to appropriate internal handlers based on job type
- Maintains a shared Git repository cache
- Manages Earthly connections for CI operations
- Reports results via request-reply pattern to dispatcher

**Internal Job Handlers**:
- **Discovery Handler**: Traverses repositories, parses CUE configurations, evaluates release triggers
- **CI Handler**: Executes Earthly targets for test and build phases
- **Artifact Handler**: Orchestrates producers and publishers, generates SBOMs, creates tracking images
- **Release Handler**: Renders CUE to Kubernetes resources, packages Release OCI images
- **Deployment Handler**: Updates GitOps repository pointer files

#### Job Distribution Pattern

Workers consume jobs using NATS pull consumers:

```go
func (w *Worker) processJobs() {
    // Create durable consumer
    consumer, _ := js.PullSubscribe(
        fmt.Sprintf("pipeline.%s.*", w.workerType),
        fmt.Sprintf("%s-consumer", w.workerType),
        nats.Durable(fmt.Sprintf("%s-consumer", w.workerType)),
    )

    for {
        // Pull next job
        msgs, err := consumer.Fetch(1, nats.MaxWait(20*time.Second))
        if err != nil || len(msgs) == 0 {
            continue
        }

        msg := msgs[0]
        job := parseJob(msg.Data)

        // Process job
        result := w.executeJob(job)

        // Reply directly to dispatcher
        msg.Respond(result.ToJSON())

        // Acknowledge after successful reply
        msg.Ack()
    }
}
```

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
        env:
        - name: NATS_URL
          value: "nats://nats:4222"
        - name: CONSUMER_NAME
          value: "discovery-consumer"
        - name: STREAM_NAME
          value: "pipeline-discovery"
        - name: WORKER_TYPE
          value: "discovery"
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
        env:
        - name: NATS_URL
          value: "nats://nats:4222"
        - name: CONSUMER_NAME
          value: "ci-consumer"
        - name: STREAM_NAME
          value: "pipeline-tasks"
        - name: WORKER_TYPE
          value: "ci"
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
        env:
        - name: NATS_URL
          value: "nats://nats:4222"
        - name: CONSUMER_NAME
          value: "artifact-consumer"
        - name: STREAM_NAME
          value: "pipeline-artifacts"
        - name: WORKER_TYPE
          value: "artifact"
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
        env:
        - name: NATS_URL
          value: "nats://nats:4222"
        - name: CONSUMER_NAME
          value: "release-consumer"
        - name: STREAM_NAME
          value: "pipeline-releases"
        - name: WORKER_TYPE
          value: "release"
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

Each worker StatefulSet scales independently based on its NATS JetStream consumer lag:

```yaml
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: worker-discovery-scaler
spec:
  scaleTargetRef:
    name: worker-discovery
  triggers:
  - type: nats-jetstream
    metadata:
      account: "$G"
      stream: pipeline-discovery
      consumer: discovery-consumer
      lagThreshold: "10"
```

(Apply similar pattern for `worker-ci`, `worker-artifact`, and `worker-release` StatefulSets)

#### NATS Infrastructure

The platform uses NATS JetStream for job distribution and status tracking:

**Stream Configuration**:
```yaml
streams:
  pipeline-discovery:
    subjects: ["pipeline.discovery.*"]
    retention: workqueue
    max_age: 1h
    max_msgs: 1000
    storage: file
    replicas: 3

  pipeline-tasks:
    subjects: ["pipeline.tasks.*"]
    retention: workqueue
    max_age: 2h
    max_msgs: 5000
    storage: file
    replicas: 3

  pipeline-artifacts:
    subjects: ["pipeline.artifacts.*"]
    retention: workqueue
    max_age: 1h
    max_msgs: 1000
    storage: file
    replicas: 3

  pipeline-releases:
    subjects: ["pipeline.releases.*"]
    retention: workqueue
    max_age: 1h
    max_msgs: 1000
    storage: file
    replicas: 3
```

**Consumer Configuration**:
Each worker StatefulSet creates a durable pull consumer:
- Acknowledgment policy: Explicit
- Replay policy: Instant
- Max deliver: 3
- Ack wait: Matches job type timeout

**Deployment**: 3-node NATS cluster with EBS-backed PVCs for persistence (ephemeral data only, no disaster recovery required)

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

**Stream Routing Logic**:
```go
func (d *Dispatcher) submitJob(job Job) (*JobResult, error) {
    var subject string

    switch job.Type {
    case JobTypeDiscovery:
        subject = fmt.Sprintf("pipeline.discovery.%s", job.ID)
    case JobTypeCI:
        subject = fmt.Sprintf("pipeline.tasks.%s", job.ID)
    case JobTypeArtifact:
        subject = fmt.Sprintf("pipeline.artifacts.%s", job.ID)
    case JobTypeRelease, JobTypeDeployment:
        subject = fmt.Sprintf("pipeline.releases.%s", job.ID)
    default:
        return nil, fmt.Errorf("unknown job type: %s", job.Type)
    }

    // Create inbox for reply
    inbox := nats.NewInbox()
    sub, _ := d.nc.SubscribeSync(inbox)
    defer sub.Unsubscribe()

    // Publish job with reply subject
    job.ReplyTo = inbox
    d.js.Publish(subject, job.ToJSON())

    // Wait for reply with timeout
    msg, err := sub.NextMsg(job.Timeout)
    if err == nats.ErrTimeout {
        return nil, fmt.Errorf("job timeout after %v", job.Timeout)
    }

    return parseJobResult(msg.Data), nil
}
```

**Dispatcher Configuration via Environment Variables**:
```yaml
- name: dispatcher
  container:
    image: forge.io/platform/dispatcher:v1.0.0
    env:
    - name: NATS_URL
      value: "nats://nats:4222"
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

### Request-Reply Pattern

All jobs use a unified request-reply pattern:

**Flow**:
1. Dispatcher publishes job with reply subject
2. Worker processes job
3. Worker replies with result/status
4. Dispatcher receives reply or times out

**Result Handling**:
- **Small results (<1MB)**: Included directly in reply message
- **Large results (>1MB)**: Stored in S3, reply contains S3 reference

**Timeout Handling**:
```go
func (d *Dispatcher) submitJob(job Job) (*JobResult, error) {
    // Create inbox for reply
    inbox := nats.NewInbox()
    sub, _ := d.nc.SubscribeSync(inbox)
    defer sub.Unsubscribe()

    // Publish job with reply subject
    job.ReplyTo = inbox
    d.js.Publish(fmt.Sprintf("pipeline.%s.%s", job.Type, job.ID), job.ToJSON())

    // Wait for reply with timeout
    msg, err := sub.NextMsg(job.Timeout)
    if err == nats.ErrTimeout {
        return nil, fmt.Errorf("job timeout after %v", job.Timeout)
    }

    return parseJobResult(msg.Data), nil
}
```

This eliminates complex polling logic and intermediate state storage.

#### S3 Storage

**Bucket Structure**:
```
forge-platform-bucket/
└── pipeline-runs/
    └── {run_id}/
        └── logs/
            └── {phase}/{project}/{step}.log  # Execution logs
```

**Lifecycle Management**:

AWS S3 Lifecycle Policies handle automatic cleanup:

```yaml
lifecycle_rules:
  - id: logs-cleanup
    prefix: pipeline-runs/*/logs/
    expiration:
      days: 90
```

**Access Patterns**:
- Workers write directly using IRSA credentials
- Platform API generates presigned URLs for user access

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
   - Dispatcher publishes job to NATS with reply subject
   - Worker service (discovery handler) pulls job from stream
   - Worker uses cached repo to perform discovery
   - Worker replies directly to dispatcher with results
   - Dispatcher receives reply, sets as output parameter
   - API updates GitHub status: "Discovering projects..."

3. **Phase Execution**
   - Workflow iterates through phase groups sequentially
   - API updates GitHub status: "Running phase X of Y"
   - For each phase, spawns parallel task dispatchers
   - Dispatchers publish all project steps to NATS with reply subjects
   - Worker service (CI handler) executes Earthly targets
   - Workers report status to API and reply to dispatchers
   - Logs stream directly to S3

4. **Phase Progression**
   - Dispatchers wait for all task replies to complete
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
   - API reconciles final status from Argo
   - API updates GitHub commit status (success/failure/error)
   - If PR workflow, API posts summary comment
   - Database retains complete history

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

**NATS Message Failures:**
- Automatic redelivery on worker failure (JetStream guarantee)
- Maximum delivery attempts configurable per stream
- Idempotent job handlers prevent duplicate execution

**Worker Service Failures:**
- Kubernetes automatically restarts failed workers
- Jobs redelivered to consumer after ack wait timeout
- Circuit breakers for Earthly connections

**Dispatcher Failures:**
- Argo retries failed dispatchers
- Timeout and retry on lost replies (dispatcher logic)
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

### Port Contracts

#### Layer 7 Routing Contract

The platform utilizes the Kubernetes Gateway API as the standard ingress contract. This contract provides a portable, expressive model for HTTP routing, traffic management, and policy enforcement.

**Contract Specification:**
- Applications define ingress requirements using Gateway API resources
- Gateway implementations handle request routing, load balancing, and TLS termination
- The contract supports advanced features including request matching, header manipulation, and traffic splitting

**Adapter Implementation:**

**Envoy Gateway:** Serves as the Gateway API implementation across all environments. Envoy Gateway achieved Gateway API GA status and provides consistent behavior whether deployed on AWS/EKS or on-premises Kubernetes. The implementation handles HTTP routing, WebSocket support, and TLS termination. Configuration remains identical across environments, with only infrastructure-level differences such as LoadBalancer service bindings varying by deployment context.

#### DNS Management Contract

Applications declare hostnames as part of their Network XRD specifications. The DNS contract ensures these declarations result in properly configured DNS records in the environment's DNS provider.

**Contract Requirements:**

An adapter for this port must provide:
- Automatic DNS record creation from Kubernetes resource declarations
- Synchronization of record lifecycle with application lifecycle (create, update, delete)
- Support for both public and private DNS zones
- Authentication mechanism appropriate to the DNS provider
- Record cleanup on resource deletion to prevent stale entries

**Adapter Implementations:**

**ExternalDNS with Route53 (Production):** Currently implemented for AWS environments. Automatically synchronizes Gateway API and Service resources with Route53 hosted zones using IRSA authentication.

**ExternalDNS with Alternative Providers (On Premises):** Architecture supports DNS providers including cloud-based services, RFC2136-compatible authoritative nameservers, and other DNS systems. Implementations maintain the same declarative model with provider-specific authentication and zone configuration.

### AWS Services Integration

- **S3**: Logs and large result storage
- **IAM/IRSA**: Service authentication and authorization
- **CloudWatch**: Metrics and log aggregation
- **Secrets Manager**: CI secret retrieval (see [Core Architecture: Secret Management Patterns](01-core-architecture.md#secret-management-patterns))

### NATS Integration

- **JetStream**: Job distribution with work-queue retention
- **Request-Reply**: Direct status/result communication
- **Pull Consumers**: Worker-driven job consumption
- **TLS**: Per-service credentials for authentication

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
