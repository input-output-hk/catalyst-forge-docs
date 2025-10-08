# Core Go Style Guide

## Table of Contents

- [Introduction](#introduction)
- [Code Organization](#code-organization)
- [Naming Conventions](#naming-conventions)
- [Error Handling](#error-handling)
- [Interface Design](#interface-design)
- [Context Usage](#context-usage)
- [Concurrency](#concurrency)
- [HTTP Clients and Servers](#http-clients-and-servers)
- [Logging](#logging)
- [Testing Standards](#testing-standards)
- [Documentation Standards](#documentation-standards)
- [Dependency Management](#dependency-management)
- [Modern Standard Library](#modern-standard-library)
- [Tooling](#tooling)
- [Code Formatting](#code-formatting)

## Introduction

This guide defines mandatory coding standards for all Go code in the Catalyst Forge platform. These rules apply equally to libraries and services. Deviations require explicit justification in code review.

Base requirement: All code must pass `go fmt`, `go vet`, and `golangci-lint` with the platform configuration.

## Code Organization

### Module Layout

Follow the [official Go module layout](https://go.dev/doc/modules/layout). Key conventions:

**Standard directories:**
- `cmd/` - Application entrypoints (one subdirectory per binary)
- `internal/` - Private packages (not importable by other modules)
- Root package - Public API if building a library

```
catalyst-forge/
├── cmd/
│   ├── api/          # Platform API server
│   ├── worker/       # Worker service
│   └── dispatcher/   # Dispatcher CLI
├── internal/
│   ├── storage/      # Database layer (private)
│   ├── handlers/     # HTTP/NATS handlers (private)
│   └── domain/       # Domain logic (private)
└── pkg/              # Public packages (if any)
```

### Package Structure

**One package per directory.** Package name matches directory name except for `main` packages.

**Package naming:** Lowercase, single word preferred. Use underscore only when required for clarity.
```go
// Good
package discovery
package artifactoci

// Avoid
package discovery_handler
package artifact_oci_handler
```

**File naming:** Lowercase with underscores. Group by logical concern, not by type.
```go
// Good
release.go          // Release entity and core logic
release_store.go    // Release persistence
release_test.go     // Release tests

// Bad
models.go          // Mixed entities
handlers.go        // Mixed handlers
utils.go           // Grab bag
```

### Import Organization

Imports must be grouped and ordered:
```go
import (
    // Standard library
    "context"
    "fmt"

    // Third-party
    "github.com/nats-io/nats.go"
    "github.com/lib/pq"

    // Platform libraries
    "github.com/input-output-hk/catalyst-forge-libs/domain"
    "github.com/input-output-hk/catalyst-forge-libs/errors"

    // Local packages
    "github.com/input-output-hk/catalyst-forge/internal/storage"
)
```

## Naming Conventions

### General Rules

**Acronyms:** Treat as words for mixed caps. All caps for exported, all lower for unexported.
```go
// Good
var imageOCI string
type HTTPClient interface
func parseJSON()

// Bad
var imageOci string
type HttpClient interface
func parseJson()
```

**Interfaces:** Single-method interfaces end with `-er`. Multi-method interfaces describe capability.
```go
// Good
type Publisher interface
type Storage interface

// Bad
type IPublisher interface
type StorageInterface interface
```

**Constructors:** Use `New` prefix for exported constructors, `new` for unexported.
```go
func NewClient(opts ...Option) *Client
func newWorker(config workerConfig) *worker
```

### Platform-Specific Conventions

**Entity types:** Singular nouns.
```go
type Release struct
type PipelineRun struct
```

**Repository types:** Suffix with `Store`.
```go
type ReleaseStore interface
type PipelineRunStore interface
```

**Handler types:** Suffix with `Handler`.
```go
type DiscoveryHandler struct
type ArtifactHandler struct
```

## Error Handling

### Error Creation

**Always wrap errors with context.** Use `fmt.Errorf` with `%w` verb.
```go
if err := store.Save(ctx, entity); err != nil {
    return fmt.Errorf("saving entity %s: %w", entity.ID, err)
}
```

**Platform errors:** Return classified errors for retryable operations.
```go
import "github.com/input-output-hk/catalyst-forge-libs/errors"

return errors.NewPlatformError(
    errors.CodeResourceNotFound,
    "release not found",
    errors.ClassificationPermanent,
)
```

### Error Handling

**Check errors immediately.** No naked returns with named error values.
```go
// Good
result, err := operation()
if err != nil {
    return nil, fmt.Errorf("operation failed: %w", err)
}

// Bad
if result, err = operation(); err != nil {
    return
}
```

**Error variables:** Prefix with `Err` and define as package-level sentinel errors.
```go
var (
    ErrNotFound = errors.New("not found")
    ErrInvalidState = errors.New("invalid state")
)
```

### Error Comparison

**Compare errors semantically,** never by string. Use `errors.Is` for sentinel errors and `errors.As` for typed errors.
```go
// Good - semantic comparison
if err := svc.Process(ctx); err != nil {
    // Check sentinel errors
    if errors.Is(err, ErrNotFound) {
        return nil, status.NotFound()
    }

    // Check typed errors
    var validationErr *ValidationError
    if errors.As(err, &validationErr) {
        return nil, status.InvalidArgument(validationErr.Field)
    }

    return nil, fmt.Errorf("service process: %w", err)
}

// Bad - string comparison
if err != nil && strings.Contains(err.Error(), "not found") {
    // Never do this
}
```

**Multiple errors:** Use `errors.Join` when operations may produce multiple errors.
```go
var errs []error
for _, item := range items {
    if err := validate(item); err != nil {
        errs = append(errs, fmt.Errorf("item %s: %w", item.ID, err))
    }
}
if len(errs) > 0 {
    return errors.Join(errs...)
}
```

## Interface Design

### Interface Definition

**Define interfaces at point of use,** not point of implementation.
```go
// Good - defined in consumer package
package worker

type Queue interface {
    Publish(ctx context.Context, msg Message) error
    Subscribe(ctx context.Context, subject string) (Subscription, error)
}

// Bad - defined in provider package
package nats

type Queue interface {
    // ...
}
```

**Keep interfaces minimal.** Single-method interfaces are ideal. Prefer many small interfaces over large ones.
```go
// Good - small, focused interfaces
type Reader interface {
    Read(ctx context.Context, path string) (io.ReadCloser, error)
}

type Writer interface {
    Write(ctx context.Context, path string, r io.Reader) error
}

// Compose when needed
type ReadWriter interface {
    Reader
    Writer
}

// Bad - large interface
type FileSystem interface {
    Read(ctx context.Context, path string) (io.ReadCloser, error)
    Write(ctx context.Context, path string, r io.Reader) error
    Delete(ctx context.Context, path string) error
    List(ctx context.Context, prefix string) ([]FileInfo, error)
    Stat(ctx context.Context, path string) (FileInfo, error)
    // ... many more methods
}
```

### Interface Implementation

**No explicit declaration.** Let compiler verify interface satisfaction.
```go
// Good
type natsQueue struct{}

func (q *natsQueue) Publish(ctx context.Context, msg Message) error {
    // implementation
}

// Bad
var _ Queue = (*natsQueue)(nil)  // Only use for compile-time verification when needed
```

## Context Usage

### Context Propagation

**Context is always first parameter.**
```go
func (s *Store) GetRelease(ctx context.Context, id string) (*Release, error)
```

**Never store context in structs.**
```go
// Good
func (h *Handler) Process(ctx context.Context, msg Message) error

// Bad
type Handler struct {
    ctx context.Context  // Never do this
}
```

### Context Values

**Use typed keys for context values.**
```go
type contextKey string

const (
    contextKeyTraceID contextKey = "trace-id"
    contextKeyUserID  contextKey = "user-id"
)

func WithTraceID(ctx context.Context, traceID string) context.Context {
    return context.WithValue(ctx, contextKeyTraceID, traceID)
}
```

**Keep context values minimal.** Only request-scoped data (trace ID, user ID). Never large objects.

### Context Timeouts

**Always defer cancel() when creating contexts with timeouts or cancelation.**
```go
// Good
ctx, cancel := context.WithTimeout(parentCtx, 30*time.Second)
defer cancel() // Prevents context leak

// Also good for manual cancelation
ctx, cancel := context.WithCancel(parentCtx)
defer cancel() // Ensures cleanup even if we don't explicitly cancel
```

## Concurrency

### Structured Concurrency

**Use `errgroup` for fan-out work** with shared cancelation.
```go
import "golang.org/x/sync/errgroup"

func ProcessItems(ctx context.Context, items []Item) error {
    g, ctx := errgroup.WithContext(ctx)

    // Bounded parallelism
    sem := make(chan struct{}, 10)

    for _, item := range items {
        item := item // Capture loop variable
        g.Go(func() error {
            sem <- struct{}{}        // Acquire
            defer func() { <-sem }() // Release

            return processItem(ctx, item)
        })
    }

    return g.Wait() // Waits for all goroutines and returns first error
}
```

### Channel Patterns

**Always handle context cancelation** in channel operations.
```go
select {
case result := <-ch:
    return result, nil
case <-ctx.Done():
    return nil, ctx.Err()
}
```

**Prefer channels for communication,** mutexes for state.

## HTTP Clients and Servers

### HTTP Clients

**Never use default client.** Always configure timeouts.
```go
// Bad
resp, err := http.Get(url) // Uses http.DefaultClient

// Good
client := &http.Client{
    Timeout: 30 * time.Second,
    Transport: &http.Transport{
        MaxIdleConns:        100,
        MaxIdleConnsPerHost: 10,
        IdleConnTimeout:     90 * time.Second,
    },
}
resp, err := client.Get(url)
```

### HTTP Servers

**Configure all server timeouts** and implement graceful shutdown.
```go
server := &http.Server{
    Addr:              ":8080",
    Handler:           mux,
    ReadHeaderTimeout: 5 * time.Second,
    ReadTimeout:       15 * time.Second,
    WriteTimeout:      15 * time.Second,
    IdleTimeout:       60 * time.Second,
}

// Graceful shutdown
go func() {
    <-ctx.Done()
    shutdownCtx, cancel := context.WithTimeout(context.Background(), 30*time.Second)
    defer cancel()
    server.Shutdown(shutdownCtx)
}()
```

## Logging

### Structured Logging

**Use `log/slog` for all logging** (Go 1.21+).
```go
import "log/slog"

// JSON logger for production
logger := slog.New(slog.NewJSONHandler(os.Stdout, &slog.HandlerOptions{
    Level: slog.LevelInfo,
}))

// Text logger for development
logger := slog.New(slog.NewTextHandler(os.Stdout, nil))
```

### Context-Aware Logging

**Attach request metadata** from context.
```go
func HandleRequest(ctx context.Context, req Request) error {
    logger := slog.With(
        "trace_id", TraceIDFromContext(ctx),
        "user_id", UserIDFromContext(ctx),
        "request_id", req.ID,
    )

    logger.Info("processing request")

    if err := process(req); err != nil {
        logger.Error("request failed", "error", err)
        return err
    }

    logger.Info("request completed", "duration_ms", time.Since(start).Milliseconds())
    return nil
}
```

**Log levels:** Use Error for failures requiring attention, Warn for recoverable issues, Info for state changes, Debug for detailed troubleshooting.

## Testing Standards

### Test Organization

**Test files adjacent to code.** Use `_test` suffix.
```go
release.go
release_test.go
release_internal_test.go  // For whitebox tests
```

**Test naming:** Use underscore to separate concerns.
```go
func TestRelease_Create_Success(t *testing.T)
func TestRelease_Create_InvalidInput(t *testing.T)
func TestRelease_Create_DuplicateError(t *testing.T)
```

### Table-Driven Tests

**Prefer table-driven tests for multiple scenarios.**
```go
func TestValidateConfig(t *testing.T) {
    tests := []struct {
        name      string
        config    Config
        wantError string
    }{
        {
            name:      "valid config",
            config:    Config{Port: 8080},
            wantError: "",
        },
        {
            name:      "invalid port",
            config:    Config{Port: -1},
            wantError: "invalid port",
        },
    }

    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {
            err := ValidateConfig(tt.config)
            if tt.wantError == "" {
                require.NoError(t, err)
            } else {
                require.ErrorContains(t, err, tt.wantError)
            }
        })
    }
}
```

### Fuzz Testing

**Add fuzz tests for parsers and validators** that handle untrusted input.
```go
func FuzzParseDiscoveryOutput(f *testing.F) {
    // Seed corpus with valid examples
    f.Add([]byte(`{"projects": [{"name": "api", "path": "services/api"}]}`))
    f.Add([]byte(`{"projects": []}`))

    f.Fuzz(func(t *testing.T, data []byte) {
        result, err := ParseDiscoveryOutput(data)
        if err != nil {
            return // Invalid input is expected
        }

        // Verify invariants for valid output
        if result == nil {
            t.Fatal("valid parse returned nil")
        }
        for _, project := range result.Projects {
            if project.Name == "" {
                t.Fatal("project with empty name")
            }
        }
    })
}
```

### Test Helpers

**Mark helpers with `t.Helper()`.**
```go
func createTestRelease(t *testing.T, store *ReleaseStore) *Release {
    t.Helper()
    release := &Release{
        Repository: "test/repo",
        Project:    "api",
    }
    require.NoError(t, store.Create(context.Background(), release))
    return release
}
```

### Race Detection

**Always run tests with race detector** during development.
```bash
go test -race ./...
```

**Test concurrent code explicitly.**
```go
func TestStore_ConcurrentAccess(t *testing.T) {
    store := NewStore()
    ctx := context.Background()

    // Run parallel operations
    t.Run("parallel", func(t *testing.T) {
        for i := 0; i < 10; i++ {
            t.Run(fmt.Sprintf("worker-%d", i), func(t *testing.T) {
                t.Parallel()

                id := fmt.Sprintf("item-%d", i)
                err := store.Save(ctx, id, []byte("data"))
                require.NoError(t, err)

                data, err := store.Load(ctx, id)
                require.NoError(t, err)
                require.Equal(t, []byte("data"), data)
            })
        }
    })
}
```

## Documentation Standards

### Package Documentation

**Every package must have a package comment.**
```go
// Package discovery implements repository traversal and project detection
// for the Catalyst Forge platform. It parses CUE configurations and builds
// execution DAGs for pipeline orchestration.
package discovery
```

### Exported Types and Functions

**Document all exported types and functions.**
```go
// Release represents an immutable snapshot of a project at a specific commit.
// Once created, a Release cannot be modified, only superseded by newer releases.
type Release struct {
    // ID is the globally unique identifier for this release.
    ID string

    // Repository identifies the source repository in format "org/name".
    Repository string
}

// NewRelease creates a new Release for the specified project and commit.
// Returns an error if the commit SHA is invalid or the project is unknown.
func NewRelease(repo, project, commitSHA string) (*Release, error) {
```

### Implementation Comments

**Explain why, not what.** Code should be self-documenting for the "what".
```go
// Good
// Use exponential backoff to avoid overwhelming the API during transient failures
for retries := 0; retries < maxRetries; retries++ {
    time.Sleep(time.Second * time.Duration(math.Pow(2, float64(retries))))
}

// Bad
// Sleep for exponentially increasing time
```

## Dependency Management

### Dependency Injection

**Constructor injection for required dependencies.**
```go
type Worker struct {
    queue     Queue
    store     Store
    publisher Publisher
}

func NewWorker(queue Queue, store Store, publisher Publisher) *Worker {
    return &Worker{
        queue:     queue,
        store:     store,
        publisher: publisher,
    }
}
```

**Options pattern for optional configuration.**
```go
type Option func(*Config)

func WithTimeout(d time.Duration) Option {
    return func(c *Config) {
        c.Timeout = d
    }
}

func NewClient(endpoint string, opts ...Option) *Client {
    config := defaultConfig()
    for _, opt := range opts {
        opt(&config)
    }
    return &Client{
        endpoint: endpoint,
        config:   config,
    }
}
```

### External Dependencies

**No external dependencies in interface definitions.**
```go
// Good
type Store interface {
    Save(ctx context.Context, data []byte) error
}

// Bad
type Store interface {
    Save(ctx context.Context, obj *dynamodb.PutItemInput) error  // AWS SDK type
}
```

## Modern Standard Library

**Use modern stdlib packages** (Go 1.21+) to reduce custom helpers.

### Slices Package

```go
import "slices"

// Before (custom helper)
func contains(items []string, target string) bool {
    for _, item := range items {
        if item == target {
            return true
        }
    }
    return false
}

// After (stdlib)
if slices.Contains(items, target) {
    // ...
}

// Other useful functions
slices.Sort(items)
slices.SortFunc(releases, func(a, b Release) int {
    return cmp.Compare(a.CreatedAt, b.CreatedAt)
})
filtered := slices.DeleteFunc(items, func(s string) bool {
    return strings.HasPrefix(s, "temp-")
})
```

### Maps Package

```go
import "maps"

// Clone a map
m2 := maps.Clone(m1)

// Merge maps
maps.Copy(dst, src)

// Get all keys
keys := maps.Keys(m)
slices.Sort(keys) // Often needed for deterministic iteration
```

### Cmp Package

```go
import "cmp"

// Compare with zero value fallback
name := cmp.Or(user.Name, "Anonymous")

// Ordered comparison
result := cmp.Compare(a.Priority, b.Priority)
```

## Tooling

### Required Tools

**go vet:** Run on all code before commit.
```bash
go vet ./...
```

**staticcheck:** Modern Go linter with excellent checks.
```bash
# Install
go install honnef.co/go/tools/cmd/staticcheck@latest

# Run
staticcheck ./...
```

**govulncheck:** Security scanning for known vulnerabilities.
```bash
# Install
go install golang.org/x/vuln/cmd/govulncheck@latest

# Run
govulncheck ./...
```

### Optional Tools

**golangci-lint:** Meta-linter aggregating multiple tools.
```yaml
# .golangci.yml
linters:
  enable:
    - gofmt
    - govet
    - staticcheck
    - errcheck
    - ineffassign
    - gosimple
```

### Development Workflow

1. Format code: `go fmt ./...` or `goimports -w .`
2. Run tests with race detection: `go test -race ./...`
3. Run linters: `go vet ./...` and `staticcheck ./...`
4. Check vulnerabilities: `govulncheck ./...`
5. Commit with confidence

## Code Formatting

### Line Length

Prefer readability over strict limits. Let `gofmt` handle formatting. If establishing a guideline, treat 100 characters as a soft limit for code, with exceptions for readability.

### Variable Declaration

**Use short declaration when possible.**
```go
// Good
msg := "hello"
count := 0

// Avoid unless needed for scope
var msg string = "hello"
var count int = 0
```

**Group related variables.**
```go
var (
    ErrNotFound = errors.New("not found")
    ErrInvalid  = errors.New("invalid")
)

const (
    DefaultTimeout = 30 * time.Second
    MaxRetries     = 3
)
```

### Return Statements

**Return early to reduce nesting.**
```go
// Good
func process(data []byte) error {
    if len(data) == 0 {
        return errors.New("empty data")
    }

    if !isValid(data) {
        return errors.New("invalid data")
    }

    return saveData(data)
}

// Bad
func process(data []byte) error {
    if len(data) > 0 {
        if isValid(data) {
            return saveData(data)
        } else {
            return errors.New("invalid data")
        }
    } else {
        return errors.New("empty data")
    }
}
```

---

**Enforcement:** All code must pass:
- `go fmt` - Standard formatting
- `go vet` - Correctness checks
- `staticcheck` - Advanced static analysis
- `govulncheck` - Security vulnerability scanning
- `go test -race ./...` - Race condition detection

Code review must verify compliance with patterns not enforceable by tooling (interface design, error handling patterns, context usage).

**Updates:** This guide is versioned with the platform. Changes require team consensus and migration plan for existing code.
