# GitOps Deployment Implementation

## Executive Summary

This document defines the technical implementation of the Catalyst Forge GitOps deployment system using an Argo CD Custom Management Plugin. The plugin extracts Kubernetes resources from Release OCI images referenced by ReleasePointer files, enabling immutable, reproducible deployments across all environments.

## Core Concepts

### Why Custom Plugin is Necessary

Standard Argo CD config management plugins (Helm, Kustomize) are insufficient for the platform's requirements:

**Standard Plugin Limitations:**
- **Helm**: Requires templating in source repository; incompatible with Release OCI model
- **Kustomize**: Operates on directory structures; cannot extract from OCI images
- **Neither**: Supports commit SHA-based resource resolution from OCI registry

**Platform Requirements:**
- Extract resources from Release OCI images (not Git repositories)
- Use commit SHA as immutable deployment reference
- Support ReleasePointer CRD as deployment specification
- Integrate with platform's OCI registry structure

### ReleasePointer as Deployment Specification

ReleasePointer CRD serves as the deployment contract between the platform and Argo CD:

```yaml
apiVersion: forge.projectcatalyst.io/v1alpha1
kind: ReleasePointer
metadata:
  name: api-server
  namespace: platform
spec:
  release: abc123def456  # Commit SHA - immutable reference
```

**Design Rationale:**
- Commit SHA provides immutability (tags can be mutated)
- Direct traceability to source commit
- Decouples release version from Git reference
- Enables GitOps workflow without embedding deployment logic in source repo

### Release OCI as Resource Delivery Mechanism

Release OCI images package Kubernetes resources for deployment:

**Format**: `[registry]/[repo]/[project]:[commit-sha]`

**Contents**:
- Rendered Kubernetes YAML manifests (ready to apply)
- Artifact references (OCI tracking image URIs)
- Release metadata (release number, trigger information)

**Benefits**:
- Immutable deployment packages
- Version resources independently from artifacts
- GitOps pull model (Argo CD extracts on demand)
- Complete deployment reproducibility

## Plugin Design

### Deployment Configuration

The Custom Management Plugin is deployed as a sidecar container in the Argo CD repo-server deployment.

**ConfigMap Registration:**

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: cmp-plugin
  namespace: argocd
data:
  plugin.yaml: |
    kind: ConfigManagementPlugin
    metadata:
      name: catalyst-forge-cmp
    spec:
      discover:
        fileName: "release.yaml"
      generate:
        command: ["/usr/local/bin/catalyst-forge-cmp"]
        args: ["generate"]
```

**Discovery Contract:**
- Argo CD invokes discovery for each directory
- Plugin returns success if `release.yaml` exists
- Only ReleasePointer files trigger this plugin

**Generate Contract:**
- Argo CD invokes generate when discovery succeeds
- Plugin receives Git working directory as CWD
- Plugin returns YAML stream to stdout
- Errors written to stderr cause sync failure

**Credential Management:**

Registry and Platform API credentials are mounted from Kubernetes Secrets:
- Created during CMP deployment (see Implementation Strategy Phase 4)
- Mounted as environment variables in sidecar
- Used for OCI image pull authentication

### Discovery Script Implementation

**Purpose**: Validate that directory contains valid ReleasePointer

**Algorithm**:
1. Check for `release.yaml` file existence
2. Parse YAML content
3. Validate structure:
   - `apiVersion: forge.projectcatalyst.io/v1alpha1`
   - `kind: ReleasePointer`
   - `spec.release` field present and non-empty
4. Return exit code 0 (success) or 1 (failure)

**Validation Contract:**
- Must be a valid YAML file
- Must match ReleasePointer CRD schema
- `spec.release` must be valid commit SHA format (40-character hexadecimal)

**Why This Validation:**
- Prevents Argo CD from invoking generate on invalid files
- Provides early error feedback
- Ensures plugin only processes compatible resources

### Generate Script Implementation

**Purpose**: Extract Kubernetes resources from Release OCI image

**Algorithm**:
1. Parse `release.yaml` from current directory
2. Extract commit SHA from `spec.release` field
3. Construct OCI URI using platform registry conventions
4. Pull Release OCI image from registry
5. Extract resources layer (identified by media type)
6. Parse multi-document YAML
7. Order resources alphabetically by kind and name
8. Check for adjacent `env.yaml` (per-project EnvironmentConfig override)
9. Return combined resource stream to stdout

**Implementation Technology**: Custom Go binary using:
- Platform OCI client library for image operations
- YAML parser for manifest processing
- Registry authentication from environment credentials

**Note**: For general Argo CD plugin mechanics, see [Argo CD Plugin Documentation](https://argo-cd.readthedocs.io/en/stable/operator-manual/config-management-plugins/)

## Release OCI Extraction

### OCI Image Structure

Release OCI images follow standard OCI Image Specification with custom media types for layer identification.

**Expected Layer Organization:**

```
Release OCI Image
├── Config Layer (image metadata)
├── Metadata Layer (release info, artifact references)
└── Resources Layer (Kubernetes YAML manifests)
    └── application/vnd.forge.resources.v1.tar+gzip
```

**Platform-Specific Media Types:**
- Config: `application/vnd.oci.image.config.v1+json` (standard)
- Metadata: `application/vnd.forge.release.metadata.v1+json` (custom)
- Resources: `application/vnd.forge.resources.v1.tar+gzip` (custom)

### Image Pull Implementation

**Registry URI Construction:**

From ReleasePointer `spec.release` commit SHA, construct URI:

```
Format: {REGISTRY_URL}/{repository}/{project}:sha-{commit-sha}

Example:
  Repository: catalyst-forge
  Project: api-server
  Commit SHA: abc123def456

  Result: registry.forge.io/catalyst-forge/api-server:sha-abc123def456
```

**Pull Process:**

1. Construct OCI URI from ReleasePointer and registry configuration
2. Authenticate using mounted credentials (environment variables)
3. Pull image manifest from registry
4. Validate manifest structure (expected layers present)
5. Pull resources layer blob by digest
6. Verify layer media type matches `application/vnd.forge.resources.v1.tar+gzip`

**Library Usage:**

Platform uses Go libraries for OCI operations:
- `github.com/google/go-containerregistry` for OCI image pulling
- Standard library `archive/tar` and `compress/gzip` for extraction

**Why This Approach:**
- Leverages standard OCI distribution specification
- Reuses container registry infrastructure
- Provides content-addressable storage via layer digests
- Enables image caching and deduplication

### Layer Identification Logic

**Resources Layer Detection:**

The plugin identifies the resources layer by media type, not position:

```go
// Pseudocode
for each layer in manifest.Layers:
  if layer.MediaType == "application/vnd.forge.resources.v1.tar+gzip":
    resourcesLayer = layer
    break

if resourcesLayer == nil:
  return error("resources layer not found")
```

**Why Media Type Matching:**
- Position-independent (layers can be reordered)
- Explicit contract (media type declares intent)
- Extensible (can add new layer types without breaking)

### YAML Extraction Process

**Extraction Algorithm:**

1. Pull resources layer blob by digest
2. Decompress gzip stream
3. Extract tar archive to memory
4. Read all `.yaml` and `.yml` files from archive
5. Parse each file as multi-document YAML
6. Collect all Kubernetes resource documents

**Multi-Document YAML Handling:**

Each file may contain multiple YAML documents separated by `---`:

```yaml
---
apiVersion: v1
kind: Service
metadata:
  name: api-server
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api-server
```

**Parsing Strategy:**
- Use YAML streaming parser (`yaml.NewDecoder`)
- Split on document separators
- Validate each document has `apiVersion` and `kind`
- Skip empty documents and YAML comments

### Resource Processing

**Resource Ordering Strategy**

Platform uses **alphabetical ordering** by resource kind, then name:

```
Algorithm:
1. Parse all resources from extracted YAML
2. Sort by (kind, metadata.name) tuple alphabetically
3. Return ordered list

Example order:
  - ConfigMap/app-config
  - ConfigMap/feature-flags
  - Deployment/api-server
  - Service/api-server
```

**Rationale for Alphabetical Ordering:**
- Simple and predictable
- Sufficient for platform's resource types (no complex dependencies)
- Argo CD applies resources with built-in dependency resolution
- Custom Resources (XRs) don't require specific order
- Kubernetes controllers handle eventual consistency

**Alternative Considered:** Dependency-aware ordering (rejected)
- Adds complexity without clear benefit
- Kubernetes controllers already handle dependencies
- Would require dependency graph analysis
- Platform resources designed to be order-independent

**Output to Argo CD:**

Resources returned as multi-document YAML stream to stdout:

```yaml
---
apiVersion: v1
kind: ConfigMap
...
---
apiVersion: apps/v1
kind: Deployment
...
```

Argo CD receives this stream and applies resources to the target cluster.

## ReleasePointer Processing

### CRD Structure

The ReleasePointer CRD defines the deployment specification contract:

**Required Fields:**

```yaml
apiVersion: forge.projectcatalyst.io/v1alpha1  # Must be exactly this value
kind: ReleasePointer                            # Must be exactly this value
metadata:
  name: <project-name>                          # Project identifier
  namespace: <repo>-<project>                   # Target namespace
spec:
  release: <commit-sha>                         # 40-character hex string
```

**Optional Fields:**

```yaml
metadata:
  annotations:
    forge.projectcatalyst.io/release-number: "v1.2.3"  # Human-readable version
    forge.projectcatalyst.io/deployed-by: "user@example.com"
    forge.projectcatalyst.io/deployed-at: "2025-10-07T10:30:00Z"
```

**Validation Rules:**

1. `apiVersion` must match `forge.projectcatalyst.io/v1alpha1`
2. `kind` must be `ReleasePointer`
3. `spec.release` must be non-empty string
4. `spec.release` must match commit SHA format: `^[a-f0-9]{40}$`
5. `metadata.name` and `metadata.namespace` required (Kubernetes validation)

### Commit SHA to OCI URI Mapping

**Mapping Algorithm:**

```
Input: ReleasePointer with spec.release = "abc123def456..."

Steps:
1. Extract repository from namespace pattern: "<repo>-<project>"
   - Namespace: "catalyst-forge-api-server"
   - Repository: "catalyst-forge"
   - Project: "api-server"

2. Construct OCI tag from commit SHA:
   - Tag format: "sha-{commit-sha}"
   - Tag: "sha-abc123def456..."

3. Combine with registry configuration:
   - Registry URL from environment: "registry.forge.io"
   - Path: "{registry}/{repo}/{project}:{tag}"
   - Result: "registry.forge.io/catalyst-forge/api-server:sha-abc123def456..."
```

**Repository/Project Extraction:**

Namespace follows pattern `<repo>-<project>`:
- Split on first hyphen for single-word repo names
- For multi-segment repos, use Git directory structure as fallback
- Project name is last segment after final hyphen

**Why This Mapping:**
- Deterministic (same inputs always produce same URI)
- No external lookups required (self-contained)
- Aligns with GitOps repository structure
- Enables multi-project repository support

### Per-Project EnvironmentConfig Handling

**Discovery Logic:**

After extracting Release OCI resources, plugin checks for adjacent `env.yaml`:

```
Directory structure:
  dev/apps/catalyst-forge/api-server/
    ├── release.yaml          # ReleasePointer
    └── env.yaml (optional)   # Per-project EnvironmentConfig override
```

**Processing Algorithm:**

1. Parse and extract resources from Release OCI image
2. Check for `env.yaml` in same directory as `release.yaml`
3. If present:
   - Parse `env.yaml` as EnvironmentConfig resource
   - Validate structure (apiVersion, kind, data fields)
   - Append to resource stream
4. Return combined resources (Release resources + optional EnvironmentConfig)

**Output Example:**

```yaml
---
# Resources from Release OCI
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api-server
---
# Optional per-project EnvironmentConfig override
apiVersion: apiextensions.crossplane.io/v1alpha1
kind: EnvironmentConfig
metadata:
  name: api-server-overrides
  namespace: catalyst-forge-api-server
data:
  replicas: "5"
  cacheSize: "2Gi"
```

**Why Include EnvironmentConfig:**
- Enables per-project environment overrides
- Both resources applied to same namespace atomically
- Composition functions can access override values
- Maintains GitOps single-source-of-truth

### Error Handling

The plugin implements comprehensive error handling with clear diagnostics:

**Missing/Invalid Commit SHA:**

```
Error: Invalid commit SHA in ReleasePointer
  File: dev/apps/catalyst-forge/api-server/release.yaml
  Value: "invalid-sha"
  Expected: 40-character hexadecimal string
  Pattern: ^[a-f0-9]{40}$
```

**Malformed ReleasePointer:**

```
Error: Invalid ReleasePointer structure
  File: dev/apps/catalyst-forge/api-server/release.yaml
  Issue: Missing required field 'spec.release'
  Expected: Valid ReleasePointer CRD
```

**Failed OCI Pull:**

```
Error: Failed to pull Release OCI image
  URI: registry.forge.io/catalyst-forge/api-server:sha-abc123def456
  Registry: registry.forge.io
  Status: 404 Not Found
  Possible causes:
    - Release not yet published
    - Invalid commit SHA
    - Registry authentication failure
```

**Invalid Resource YAML:**

```
Error: Invalid Kubernetes resource in Release OCI
  File: deployment.yaml (from resources layer)
  Issue: Missing required field 'apiVersion'
  Resource cannot be applied to cluster
```

**Error Propagation:**

All errors:
1. Written to stderr with structured format
2. Cause plugin to exit with non-zero code
3. Trigger Argo CD Application sync failure
4. Display in Argo CD UI with full error context
5. Include sufficient detail for platform engineers to diagnose

**Retry Behavior:**

Argo CD handles retries:
- Transient OCI pull failures: Retry with exponential backoff
- Malformed ReleasePointer: No retry (requires Git fix)
- Missing resources layer: No retry (requires Release OCI rebuild)

## Integration Points

### Platform Components Integration

**Worker Service Integration:**

The Deployment Handler in Worker Service creates/updates ReleasePointer files:

```
Deployment Handler workflow:
1. Receive deployment job from NATS
2. Clone GitOps repository
3. Navigate to: {env}/apps/{repo}/{project}/
4. Create or update release.yaml:
   - Set spec.release to commit SHA
   - Add deployment metadata annotations
5. Commit and push to GitOps repository
6. Reply to dispatcher with deployment status
```

**GitOps Repository Structure:**

Plugin expects standardized directory structure:

```
gitops-repo/
├── dev/
│   └── apps/
│       └── {repo}/
│           └── {project}/
│               ├── release.yaml       # Required
│               └── env.yaml (optional)
├── staging/
│   └── apps/
│       └── {repo}/
│           └── {project}/
│               ├── release.yaml
│               └── env.yaml (optional)
└── production/
    └── apps/
        └── {repo}/
            └── {project}/
                ├── release.yaml
                └── env.yaml (optional)
```

**Argo CD Application Configuration:**

Applications must specify plugin in source configuration:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: dev-catalyst-forge-api-server
  namespace: argocd
spec:
  source:
    repoURL: https://github.com/org/gitops-repo
    path: dev/apps/catalyst-forge/api-server
    plugin:
      name: catalyst-forge-cmp  # References our plugin
  destination:
    server: https://kubernetes.default.svc
    namespace: catalyst-forge-api-server
  syncPolicy:
    automated: {}
    syncOptions:
      - CreateNamespace=true
```

### Data Flow

**Complete Deployment Flow:**

```
1. Trigger Deployment
   Developer/CLI → Platform API → Worker Service (Deployment Handler)

2. Update GitOps Repository
   Worker Service → GitOps Repo (commit ReleasePointer change)
                 → Reply to dispatcher

3. Change Detection
   Argo CD → Poll GitOps repository (default: 3 minutes)
          → Detect release.yaml change
          → Trigger reconciliation

4. Plugin Discovery
   Argo CD → Invoke CMP discovery script
          → Plugin validates release.yaml exists and is valid
          → Plugin returns success

5. Resource Extraction
   Argo CD → Invoke CMP generate script
          → Plugin reads release.yaml
          → Plugin constructs OCI URI from spec.release
          → Plugin pulls Release OCI from registry
          → Plugin extracts resources layer
          → Plugin parses and orders YAML
          → Plugin checks for env.yaml
          → Plugin returns resource stream to Argo CD

6. Resource Application
   Argo CD → Apply resources to target cluster/namespace
          → Update Application status
          → Mark sync as succeeded

7. Reconciliation
   Kubernetes Controllers → Reconcile applied resources
   Crossplane → Resolve EnvironmentConfig values
             → Create managed resources
```

**Error Flow:**

```
If plugin fails:
  Plugin → Exit non-zero with error to stderr
  Argo CD → Mark Application sync as failed
         → Display error in UI
         → Retry based on sync policy

If OCI pull fails:
  Plugin → Log detailed error with URI and status
  Argo CD → Show error: "Release OCI image not found"
  Platform Engineer → Verify release published correctly
```

## Design Decisions

### Why Commit SHA in ReleasePointer

**Decision**: Use commit SHA as immutable release reference

**Rationale:**
- **Immutability**: Commit SHA cannot be changed (unlike tags)
- **Traceability**: Direct link to source commit in Git
- **Reproducibility**: Same SHA always references same content
- **Audit Trail**: Clear provenance from source to deployment

**Alternatives Considered:**

1. **Git Tags** (rejected):
   - Tags can be moved to different commits
   - No guarantee of immutability
   - Would break deployment reproducibility

2. **Release Numbers** (rejected):
   - Requires additional mapping layer
   - Adds complexity and potential failure point
   - Commit SHA already provides unique identifier

3. **OCI Digest** (rejected):
   - Less human-readable
   - Breaks traceability to source commit
   - Requires external mapping to source

### Why Separate OCI Image for Releases

**Decision**: Package releases as dedicated OCI images, separate from artifact images

**Rationale:**
- **Version Independence**: Resources evolve separately from artifacts
- **GitOps Pull Model**: Argo CD extracts on-demand (not pushed)
- **Complete Deployment Package**: All resources in single immutable unit
- **Registry Infrastructure**: Reuse existing container registry
- **Caching**: OCI layer deduplication reduces storage

**Relationship to Artifact OCI Images:**

Release OCI images **reference** Artifact OCI images in resource definitions:

```yaml
# Inside Release OCI resources
apiVersion: forge.projectcatalyst.io/v1alpha1
kind: Deployment
spec:
  image: registry.forge.io/artifacts/api-server:sha-abc123@sha256:def456...
  #                                              ^-- Artifact OCI reference
```

This creates complete audit trail: Deployment → Release OCI → Artifact OCI → Source

### Resource Ordering Strategy

**Decision**: Use alphabetical ordering by (kind, name) tuple

**Rationale:**
- **Simplicity**: Easy to understand and predict
- **Sufficient**: Platform resources have no strict ordering requirements
- **Controller-Based**: Kubernetes handles dependencies via reconciliation
- **Crossplane Design**: XRs designed to be order-independent
- **Argo CD Support**: Built-in retry and dependency handling

**Alternatives Considered:**

1. **Dependency-Aware Ordering** (rejected):
   - Requires dependency graph analysis
   - Adds complexity without clear benefit
   - Platform resources don't have strict dependencies
   - Controllers already handle eventual consistency

2. **Explicit Ordering Annotations** (rejected):
   - Requires manual specification in every resource
   - Error-prone and maintenance burden
   - Unnecessary given controller-based reconciliation

3. **Argo CD Sync Waves** (deferred):
   - Could be added later if needed
   - Not required for current resource types
   - Would be applied in Release OCI generation, not plugin

**Impact on Deployment:**

Alphabetical ordering means:
- ConfigMaps/Secrets may apply after Deployments
- Deployments may start before dependencies ready
- Kubernetes controllers retry until dependencies available
- Eventual consistency guarantees correct final state

This is acceptable because:
- Crossplane XRs tolerate missing EnvironmentConfigs (retry)
- Standard resources have controller retry logic
- Argo CD marks sync complete when all resources healthy

## Future Considerations

### Caching Extracted Resources

**Opportunity**: Cache extracted resources by Release OCI digest

**Benefits:**
- Reduce registry bandwidth for re-syncs
- Faster sync operations for unchanged releases
- Lower registry API rate limit consumption

**Implementation Considerations:**
- Cache invalidation strategy (digest-based)
- Storage location (emptyDir, PVC, or external cache)
- Cache size limits and eviction policy
- Multi-pod cache coordination

### Alternative Registry Backends

**Opportunity**: Support non-OCI registries for Release storage

**Potential Backends:**
- Git LFS for large resource files
- Object storage (S3) with custom addressing
- Dedicated artifact repository (Artifactory, Nexus)

**Trade-offs:**
- Lose OCI ecosystem benefits (tooling, caching)
- Custom pull implementation required
- May improve performance for very large releases

### Multi-Cluster Extraction Optimization

**Opportunity**: Optimize for deployments across many clusters

**Approaches:**
- Shared cache layer across Argo CD instances
- Pre-warming cache before large rollouts
- Regional registry mirrors for geo-distributed clusters
- Parallel extraction for multiple environments

**Benefits:**
- Faster multi-environment deployments
- Reduced registry load during large rollouts
- Improved disaster recovery scenarios

---

**Related Documents:**
- System Architecture (architecture/03-system-architecture.md) - GitOps deployment flow context and architectural concepts
- Implementation Guide (architecture/05-implementation-guide.md) - Environment configuration model and bootstrapping

**Last Updated:** 2025-10-07
