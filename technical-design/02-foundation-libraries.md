# Foundation Libraries Architecture

## Table of Contents

- [Executive Summary](#executive-summary)
- [Core Concepts](#core-concepts)
  - [Library Organization](#library-organization)
  - [Dependency Management](#dependency-management)
  - [Interface Design Principles](#interface-design-principles)
- [Library Catalog](#library-catalog)
  - [Core Libraries](#core-libraries)
  - [Abstraction Libraries](#abstraction-libraries)
  - [Domain Logic Libraries](#domain-logic-libraries)
  - [Implementation Libraries](#implementation-libraries)
  - [Service Integration Libraries](#service-integration-libraries)
- [Dependency Architecture](#dependency-architecture)
  - [Dependency Layers](#dependency-layers)
  - [Dependency Rules](#dependency-rules)
- [Key Interfaces](#key-interfaces)
  - [Filesystem Abstraction](#filesystem-abstraction)
  - [Message Queue Abstraction](#message-queue-abstraction)
  - [Authentication and Authorization](#authentication-and-authorization)
  - [Secret Provider](#secret-provider)
  - [Publisher Interface](#publisher-interface)
  - [Worker Handler Interface](#worker-handler-interface)
- [Cross-Cutting Concerns](#cross-cutting-concerns)
  - [Error Handling](#error-handling)
  - [Observability](#observability)
  - [Configuration](#configuration)
  - [Testing](#testing)
- [Integration Patterns](#integration-patterns)
  - [Service Integration](#service-integration)
  - [Library Versioning](#library-versioning)
- [Constraints](#constraints)
  - [Design Constraints](#design-constraints)
  - [Dependency Constraints](#dependency-constraints)
- [Future Considerations](#future-considerations)
  - [Potential Libraries](#potential-libraries)
  - [Extension Points](#extension-points)

## Executive Summary

This document defines the foundation libraries that provide shared functionality across all Catalyst Forge platform services. These libraries, housed in the `catalyst-forge-libs` repository, implement core domain logic, infrastructure abstractions, utility functions, and service integration patterns used by the Platform API, Worker Service, and Dispatcher.

The library architecture follows a layered approach with clear dependency rules, ensuring maintainability and testability. All libraries implement the ports and adapters pattern where appropriate, providing clean abstractions with multiple implementations.

## Core Concepts

### Library Organization

Libraries are organized into five categories based on their purpose and dependency requirements:

**Core Libraries:** Foundation types with no dependencies on other platform libraries. These define domain entities, error types, and other fundamental structures.

**Abstraction Libraries:** Define interfaces (ports) and provide implementations (adapters) for infrastructure concerns. These libraries depend only on core libraries.

**Domain Logic Libraries:** Implement platform-specific business logic around projects, repositories, and discovery. These orchestrate lower-level libraries to provide higher-level functionality.

**Implementation Libraries:** Provide specific functionality for build tools, artifact management, and specialized operations. These may depend on multiple other library types.

**Service Integration Libraries:** Provide integration with external services and coordination logic for complex workflows. These libraries bridge between domain logic and external systems.

### Dependency Management

Libraries follow strict dependency rules to prevent circular dependencies and maintain clear architectural boundaries:

- Libraries can only depend on libraries in the same or lower layers
- No circular dependencies permitted
- External dependencies minimized and explicitly declared
- Interfaces used to break dependency cycles where necessary

### Interface Design Principles

**Abstraction First:** Define interfaces before implementations. Services depend on interfaces, not concrete types.

**Provider Pattern:** Use provider/adapter pattern for external service integration (secrets, auth, storage).

**Context Propagation:** All operations accept `context.Context` for cancellation and tracing.

**Error Handling:** Return explicit errors rather than panic. Errors include classification for retry logic.

## Library Catalog

### Core Libraries

#### `domain`
**Purpose:** Define all domain entities, value objects, and event schemas used across the platform.

**Responsibilities:**
- Domain entity definitions with struct tags (JSON, SQL, YAML)
- Value objects and enumerations
- Event schema definitions
- Version constants and compatibility

**Key Types:**
- `PipelineRun`, `Release`, `Deployment`, `Artifact`
- `PipelineStatus`, `ReleaseType`, `Environment`
- `PipelineEvent`, `ReleaseEvent`, `DeploymentEvent`

**Dependencies:** None

---

#### `errors`
**Purpose:** Provide consistent error handling across all platform components.

**Responsibilities:**
- Error type definitions and error codes
- Error classification (retryable, permanent, user error)
- Error wrapping with context preservation
- Error serialization for API responses

**Key Types:**
- `PlatformError` interface
- `ErrorCode` enumeration
- `ErrorClassification` (Retryable, Permanent)

**Dependencies:** None

### Abstraction Libraries

#### `fs`
**Purpose:** Universal filesystem abstraction supporting local and object storage.

**Responsibilities:**
- Filesystem interface definition
- Local filesystem implementation
- S3/MinIO object storage implementations
- In-memory filesystem for testing
- Result storage routing (small in-memory, large in object store)
- Presigned URL generation for object storage

**Implementations:**
- `LocalFS` - Standard filesystem operations
- `S3FS` - Amazon S3 operations
- `MinIOFS` - MinIO operations
- `MemoryFS` - In-memory for testing

**Dependencies:** `errors`

---

#### `mq`
**Purpose:** Message queue abstraction for asynchronous job processing with NATS-specific patterns.

**Responsibilities:**
- Queue interface definition
- NATS JetStream implementation with pull consumers
- Request-reply pattern with inbox management
- Consumer group management and lifecycle
- Stream configuration helpers
- Message acknowledgment and retry
- Dead letter queue handling

**Implementations:**
- `NATSQueue` - NATS JetStream implementation with full pattern support
- `MemoryQueue` - In-memory for testing

**Dependencies:** `domain`, `errors`

---

#### `database`
**Purpose:** Database operations and connection management.

**Responsibilities:**
- PostgreSQL connection pooling
- Migration execution
- Transaction management
- Query building with domain entities
- Connection retry logic

**Dependencies:** `domain`, `errors`

---

#### `auth`
**Purpose:** Authentication and authorization for all platform operations.

**Responsibilities:**
- Identity abstraction
- Authorization decisions ("can identity X do action Y on resource Z")
- Token validation and introspection
- Service account management
- Role/permission mapping

**Implementations:**
- `KeycloakAuthorizer` - Full Keycloak integration with OIDC token validation, service accounts, and role mapping
- `StaticAuthorizer` - Testing/development with hardcoded permissions

**Dependencies:** `errors`

---

#### `secrets`
**Purpose:** Secret retrieval from multiple providers.

**Responsibilities:**
- Secret provider interface
- AWS Secrets Manager integration
- HashiCorp Vault integration
- Kubernetes Secrets integration
- Secret reference resolution

**Implementations:**
- `AWSSecretsProvider` - AWS Secrets Manager
- `VaultProvider` - HashiCorp Vault
- `KubernetesProvider` - Kubernetes Secrets
- `StaticProvider` - Testing with hardcoded values

**Dependencies:** `errors`

---

#### `observability`
**Purpose:** Logging, metrics, and tracing instrumentation.

**Responsibilities:**
- Structured logging interface
- Prometheus metrics collection
- OpenTelemetry trace propagation
- Context enrichment
- Standard protocol implementations

**Dependencies:** `errors`

---

#### `execution`
**Purpose:** Task execution abstraction supporting multiple execution engines.

**Responsibilities:**
- Executor interface definition
- Built-in adapters (Earthly, Docker, Script)
- Executor registry and factory
- Execution lifecycle management
- Output streaming interface
- Artifact extraction interface

**Internal Adapters:**
- `earthlyAdapter` - Earthly execution (uses `earthly` package)
- `dockerAdapter` - Docker execution (future)
- `scriptAdapter` - Shell script execution (future)

**Dependencies:** `earthly`, `errors`, `fs`

### Domain Logic Libraries

#### `schemas`
**Purpose:** Generated Go types from CUE schemas and version management.

**Responsibilities:**
- Go struct definitions from CUE schemas
- Configuration version management
- Schema migration utilities
- Validation helpers

**Key Types:**
- `RepoConfig`, `ProjectConfig`
- `PipelinePhase`, `ArtifactSpec`
- Version constants and checkers

**Dependencies:** None

---

#### `repository`
**Purpose:** Repository-level configuration and conventions.

**Responsibilities:**
- Repository configuration parsing (`repo.cue`)
- Phase definitions management
- Publisher configuration
- Tagging strategy implementation
- Repository validation

**Dependencies:** `schemas`

---

#### `project`
**Purpose:** Project-level configuration and behavior.

**Responsibilities:**
- Project configuration parsing (`project.cue`)
- Phase participation rules
- Artifact configuration
- Deployment resource definitions
- Release trigger evaluation

**Dependencies:** `schemas`, `domain`

---

#### `discovery`
**Purpose:** Project discovery and pipeline DAG construction.

**Responsibilities:**
- Repository traversal
- Project detection (`.forge` directories)
- Execution DAG generation
- Dependency analysis
- Discovery output formatting
- Earthfile target analysis for DAG construction

**Dependencies:** `repository`, `project`, `git`, `earthly`

---

#### `git`
**Purpose:** Git repository operations and caching.

**Responsibilities:**
- Repository cloning and fetching
- Worktree management
- Cache management and invalidation
- Branch and tag operations
- Commit history traversal

**Dependencies:** `fs`, `errors`

---

#### `cue`
**Purpose:** CUE evaluation and Kubernetes resource generation.

**Responsibilities:**
- CUE schema validation
- Kubernetes YAML rendering
- Reference resolution (`@artifact()` functions)
- Environment-specific value injection
- CUE module management

**Dependencies:** `schemas`, `domain`

### Implementation Libraries

#### `earthly`
**Purpose:** Comprehensive Earthly support including parsing and execution.

**Responsibilities:**
- Earthfile syntax parsing
- Target dependency graph extraction
- Variable and import resolution
- Earthly binary invocation
- Build context preparation
- Log streaming
- Artifact extraction
- Cache management

**Note:** This is a standalone utility library used by the `execution` package's internal Earthly adapter.

**Dependencies:** `fs`, `errors`

---

#### `oci`
**Purpose:** OCI image manipulation and registry operations.

**Responsibilities:**
- Image layer construction
- Manifest generation
- Registry push/pull operations
- Multi-arch image support
- Registry authentication

**Implementations:**
- `ECRRegistry` - Amazon ECR operations
- `HarborRegistry` - Harbor operations
- `GenericRegistry` - Standard OCI registry

**Dependencies:** `fs`

---

#### `provenance`
**Purpose:** Build provenance and artifact tracking.

**Responsibilities:**
- Artifact OCI tracking image creation
- Release OCI image packaging
- SBOM generation
- Build metadata collection
- Provenance attestation

**Dependencies:** `oci`, `domain`

---

#### `publishers`
**Purpose:** Artifact distribution to various registries and storage systems.

**Responsibilities:**
- Publisher interface definition
- Registry credential management
- Idempotent publishing
- Multi-destination support

**Implementations:**
- `DockerPublisher` - Docker/OCI registries
- `GitHubPublisher` - GitHub Releases
- `PyPIPublisher` - Python Package Index
- `S3Publisher` - S3-compatible storage

**Dependencies:** `oci`, `fs`

---

#### `gitops`
**Purpose:** GitOps repository management and Argo CD integration.

**Responsibilities:**
- ReleasePointer resource management
- GitOps repository structure
- Argo CD Custom Management Plugin helpers
- Environment organization
- Deployment tracking

**Dependencies:** `git`, `domain`

### Service Integration Libraries

#### `argo`
**Purpose:** Argo Workflows integration for pipeline orchestration.

**Responsibilities:**
- Argo Server REST API client
- Workflow template generation from discovery output
- Workflow submission and monitoring
- Dynamic DAG construction for pipeline phases
- Workflow status synchronization
- Result retrieval

**Dependencies:** `domain`, `discovery`, `errors`

---

#### `github`
**Purpose:** GitHub API integration for platform operations.

**Responsibilities:**
- GitHub API v3/v4 client wrapper
- Commit status updates
- Webhook event parsing and validation
- GitHub OIDC token validation
- Repository configuration queries
- Release and deployment status reporting

**Dependencies:** `domain`, `errors`

---

#### `worker`
**Purpose:** Core task execution logic for all worker job types.

**Responsibilities:**
- Job handler interface and registry
- Handler implementations for all job types:
  - `DiscoveryHandler` - Repository discovery and DAG generation
  - `CIHandler` - Earthly target execution and log streaming
  - `ArtifactHandler` - Producer/publisher orchestration
  - `ReleaseHandler` - CUE rendering and Release OCI creation
  - `DeploymentHandler` - GitOps repository updates
- Result packaging (memory vs S3 based on size)
- Progress reporting to Platform API
- Git cache coordination
- Error classification and retry logic

**Dependencies:** `git`, `discovery`, `execution`, `publishers`, `cue`, `oci`, `provenance`, `gitops`, `fs`, `mq`, `domain`

---

#### `argocd`
**Purpose:** Argo CD Custom Management Plugin implementation.

**Responsibilities:**
- ReleasePointer resource parsing
- Release OCI image fetching from registry
- Resource layer extraction from OCI image
- Kubernetes manifest generation
- Plugin protocol implementation
- Environment-specific resource filtering

**Dependencies:** `oci`, `domain`, `errors`

## Dependency Architecture

### Dependency Layers

```
Layer 0 (Foundation):
├── domain            # Domain entities
├── errors            # Error handling
└── schemas           # Generated types

Layer 1 (Core Abstractions):
├── fs                # Filesystem abstraction
├── mq                # Message queue abstraction (with NATS patterns)
├── database          # Database operations
├── auth              # Authentication/authorization (with Keycloak)
├── secrets           # Secret management
└── observability     # Logging, metrics, tracing

Layer 2 (Domain Logic & Utilities):
├── repository        # Repository configuration
├── project           # Project configuration
├── git               # Git operations
├── cue               # CUE evaluation
├── earthly           # Earthly parsing and execution
└── oci               # OCI image operations

Layer 3 (Higher Level Abstractions):
├── discovery         # Project discovery
├── execution         # Task execution abstraction (uses earthly)
└── publishers        # Artifact publishing

Layer 4 (Specialized):
├── provenance        # Artifact tracking
└── gitops            # GitOps management

Layer 5 (Service Integration):
├── argo              # Argo Workflows integration
├── github            # GitHub API integration
├── worker            # Worker job execution
└── argocd            # Argo CD plugin
```

### Dependency Rules

1. **Layered Dependencies:** Libraries can only depend on libraries in the same or lower layers
2. **No Circular Dependencies:** Enforced through careful interface design
3. **Core Isolation:** Core libraries (Layer 0) have no dependencies on other platform libraries
4. **Interface Dependencies:** Higher layers depend on interfaces, not implementations
5. **Service Integration:** Layer 5 libraries can depend on any lower layer as they orchestrate complex workflows

## Key Interfaces

### Filesystem Abstraction

```go
package fs

type FS interface {
    Read(ctx context.Context, path string) (io.ReadCloser, error)
    Write(ctx context.Context, path string, r io.Reader) error
    Delete(ctx context.Context, path string) error
    List(ctx context.Context, prefix string) ([]FileInfo, error)
    Stat(ctx context.Context, path string) (FileInfo, error)
    SignedURL(ctx context.Context, path string, ttl time.Duration) (string, error)
}

type FileInfo interface {
    Name() string
    Size() int64
    ModTime() time.Time
    IsDir() bool
}
```

### Message Queue Abstraction

```go
package mq

type Queue interface {
    Publish(ctx context.Context, subject string, msg Message) error
    Subscribe(ctx context.Context, subject string) (Subscription, error)
    RequestReply(ctx context.Context, subject string, msg Message, timeout time.Duration) (Message, error)
    CreateInbox() string
}

type Consumer interface {
    Fetch(ctx context.Context, batch int) ([]Message, error)
    Ack(ctx context.Context, msg Message) error
    Nack(ctx context.Context, msg Message) error
}

type StreamManager interface {
    CreateStream(ctx context.Context, config StreamConfig) error
    CreateConsumer(ctx context.Context, stream string, config ConsumerConfig) (Consumer, error)
}

type Message struct {
    Subject string
    Data    []byte
    Headers map[string]string
    Reply   string
}
```

### Authentication and Authorization

```go
package auth

type Identity interface {
    Subject() string
    Groups() []string
    Claims() map[string]interface{}
}

type Authorizer interface {
    Authorize(ctx context.Context, identity Identity, action Action) error
    ValidateToken(ctx context.Context, token string) (Identity, error)
    CreateServiceAccountToken(ctx context.Context, serviceAccount string) (string, error)
}

type Action struct {
    Verb     string  // "create", "read", "update", "delete"
    Resource string  // "pipeline", "release", "deployment"
    Target   string  // "repo/project"
}
```

### Secret Provider

```go
package secrets

type Provider interface {
    Get(ctx context.Context, ref SecretRef) ([]byte, error)
    List(ctx context.Context, prefix string) ([]SecretRef, error)
}

type SecretRef struct {
    Provider string  // "aws", "vault", "kubernetes"
    Path     string  // Provider-specific path
    Key      string  // Optional key within secret
}
```

### Publisher Interface

```go
package publishers

type Publisher interface {
    Type() string
    Publish(ctx context.Context, artifact Artifact) (PublishResult, error)
    Verify(ctx context.Context, location string) error
}

type Artifact struct {
    Name     string
    Type     string  // "container", "binary", "archive"
    Path     string
    Metadata map[string]string
}

type PublishResult struct {
    URI      string
    Digest   string
    Location string
}
```

### Worker Handler Interface

```go
package worker

type Handler interface {
    Type() string
    Execute(ctx context.Context, job Job) (*Result, error)
}

type Job struct {
    ID          string
    Type        string
    Payload     []byte
    RunID       string
    Repository  string
    CommitSHA   string
}

type Result struct {
    Success  bool
    Data     []byte
    S3Key    string  // For large results
    Error    error
}

// Registry for handler management
type Registry interface {
    Register(handler Handler) error
    Get(jobType string) (Handler, error)
}
```

## Cross-Cutting Concerns

### Error Handling

All libraries follow consistent error handling patterns:
- Return explicit errors, never panic in library code
- Wrap errors with context using `fmt.Errorf` with `%w`
- Classify errors for retry logic
- Include relevant debugging information

### Observability

All libraries integrate with the `observability` package:
- Structured logging with contextual fields
- Metrics for operation duration and error rates
- Trace propagation through context
- Standard naming conventions for metrics

### Configuration

Libraries accept configuration through explicit structs:
- No global configuration
- Configuration validation in constructors
- Sensible defaults where appropriate
- Environment-specific overrides through options pattern

### Testing

All libraries provide testing utilities:
- Mock implementations of interfaces
- Test fixtures and builders
- Integration test helpers
- Deterministic behavior for tests

## Integration Patterns

### Service Integration

Services integrate libraries through dependency injection:

```go
// Platform API
type Server struct {
    db         database.DB
    auth       auth.Authorizer
    queue      mq.Queue
    secrets    secrets.Provider
    argo       argo.Client
    github     github.Client
}

// Worker Service
type Worker struct {
    queue      mq.Queue
    fs         fs.FS
    registry   worker.Registry
    publishers map[string]publishers.Publisher
}

// Worker uses registry to execute jobs
func (w *Worker) ProcessJob(ctx context.Context, msg mq.Message) error {
    var job worker.Job
    if err := json.Unmarshal(msg.Data, &job); err != nil {
        return err
    }

    handler, err := w.registry.Get(job.Type)
    if err != nil {
        return fmt.Errorf("unknown job type: %s", job.Type)
    }

    result, err := handler.Execute(ctx, job)
    // Handle result and reply via queue
}
```

### Library Versioning

Libraries follow semantic versioning:
- Major version changes for breaking API changes
- Minor version changes for new functionality
- Patch version changes for bug fixes
- All libraries versioned together in monorepo

## Constraints

### Design Constraints

- Libraries must not import service code
- No global state or singletons
- Interfaces over concrete types for dependencies
- Context required for all operations

### Dependency Constraints

- No circular dependencies between libraries
- Minimize external dependencies
- Vendor dependencies for stability
- Security scanning for all dependencies

## Future Considerations

### Potential Libraries

**`cache`:** Distributed caching abstraction for Git repositories and build artifacts

**`policy`:** Policy engine for admission control and resource validation

**`workflow`:** Workflow definition and validation (if moving beyond Argo)

### Extension Points

- Additional storage providers in `fs`
- New message queue implementations in `mq`
- Extended publisher types in `publishers`
- Alternative secret providers in `secrets`
- New execution adapters in `execution` (Bazel, Make, Gradle, etc.)
- Additional worker handlers in `worker` for new job types

---

**Related Documents:**
- Implementation Strategy - Build sequence and library development phases
- Architecture Overview - System design using these libraries
- Developer Guide - How applications use these libraries

**Repository:** All libraries reside in `github.com/input-output-hk/catalyst-forge-libs`

**Last Updated:** 2025-10-05