# Crossplane Versioning Strategy

## Executive Summary

This document defines the platform's approach to versioning Crossplane infrastructure abstractions (XRDs, Compositions, and Functions). The strategy enables controlled, predictable rollouts of infrastructure changes while providing escape hatches for critical services that require strict version control.

The versioning model treats each XRD family (Database, Network, Storage, etc.) as an independently versioned package with semantic versioning. Changes progress through channels (dev, staging, stable) to enable gradual rollout, while override mechanisms allow specific instances to pin to exact versions when needed.

**Target Audience:** Platform Engineers responsible for maintaining infrastructure abstractions and managing platform deployments.

## Core Concepts

### Why Version Infrastructure Abstractions

Infrastructure abstractions evolve over time. Bug fixes, new features, and behavioral changes require a versioning strategy that balances two competing needs:

1. **Default safety:** Most applications should receive improvements automatically without manual intervention
2. **Explicit control:** Critical services need deterministic behavior and controlled upgrade windows

Traditional approaches force an all-or-nothing choice: either all instances upgrade automatically (risky) or all instances require manual intervention (operationally expensive). Our strategy provides both through channel-based progression with selective pinning.

### The Channel Model

Channels represent rollout stages, not environments. The platform uses two channels:

- **dev:** Pre-release versions for validation and testing. Can be used in any environment.
- **stable:** Production-ready versions that have been validated.

This model decouples infrastructure version progression from environment promotion. You can safely test new Database XRD versions in production by creating a single dev-channel instance, validating behavior, then promoting the version to stable when ready.

### Major Versions and Composition Names

Breaking changes to XRD schemas or Composition behavior require a new major version. Following Crossplane best practices, major version changes are reflected in the Composition name itself:

- `DatabaseV1` for v1.x.x releases
- `DatabaseV2` for v2.x.x releases (breaking changes)
- `NetworkV1`, `NetworkV2`, etc.

This approach makes breaking changes explicit and prevents accidental upgrades. Applications must explicitly migrate to the new major version by changing their XR's `kind` field (e.g., `Database` becomes `DatabaseV2`).

Within a major version, all releases are backward compatible. Minor and patch releases progress through channels without requiring application changes.

### Immutable Composition Revisions

Every change to a Composition creates an immutable CompositionRevision. These revisions are labeled with metadata (channel, release version) that enables selection. Composite Resources (XRs) bind to revisions through label selectors, not object names, providing flexibility in how versions are assigned.

## Versioning Model

### Package Structure

Each XRD family (e.g., DatabaseV1) ships as three coordinated packages:

**Configuration Package (XRD + Composition):**
- Contains the XRD definition and Composition
- Tagged with semantic version (v1.0.0, v1.1.0-dev, etc.)
- Declares dependency on Function package
- Published to OCI registry

**Function Package:**
- Contains the function runtime implementation
- Tagged with matching semantic version
- Published to OCI registry
- Built from Earthfile

**Function Object (Cluster Resource):**
- Kubernetes resource that references the Function package
- Named with encoded version (e.g., `db-func-v1-0-0`)
- Allows side-by-side execution of multiple versions
- Created manually in cluster, not bundled in packages

### Version Alignment

All three packages share the same semantic version. When releasing DatabaseV1 v1.2.0-dev:
- Configuration package is tagged `v1.2.0-dev`
- Function package is tagged `v1.2.0-dev`
- Function object is named `db-func-v1-2-0-dev` and references `v1.2.0-dev` package

When promoting to stable, the same artifacts are published with stable tags:
- Configuration package is tagged `v1.2.0`
- Function package is tagged `v1.2.0`
- Function object is named `db-func-v1-2-0` and references `v1.2.0` package

This alignment simplifies reasoning about compatibility and ensures Compositions always reference compatible Functions.

### Semantic Versioning Semantics

Follow standard semantic versioning with platform-specific interpretation:

**Major version (v2.0.0):** Breaking changes require a new Composition name (DatabaseV1 → DatabaseV2). Applications must explicitly migrate.
**Minor version (v1.1.0):** New features, non-breaking changes, or significant behavioral improvements
**Patch version (v1.0.1):** Bug fixes and minor improvements with no observable behavioral changes

**Pre-release identifiers** distinguish dev from stable releases:
- **Dev releases:** `v1.0.0-dev`, `v1.1.0-dev` - pre-release versions for validation
- **Stable releases:** `v1.0.0`, `v1.1.0` - production-ready versions

Pre-release versions automatically sort before their stable counterparts (`v1.0.0-dev < v1.0.0`) and are automatically labeled `channel: dev` during publication.

## Rollout Strategy

### Channel-Based Progression

New versions progress through two channels to enable gradual rollout:

1. **dev:** Pre-release versions (e.g., `v1.1.0-dev`). Opt-in for validation in any environment.
2. **stable:** Production-ready releases (e.g., `v1.1.0`). Default for most instances.

Channels are implemented as labels on CompositionRevisions:

```yaml
apiVersion: apiextensions.crossplane.io/v1
kind: Composition
metadata:
  name: database-v1
  labels:
    channel: dev
    release: v1.1.0-dev
```

These labels are automatically applied based on the version's pre-release identifier during publication.

### Default Channel Selection

The Argo CD Custom Management Plugin (CMP) automatically configures XRs to select compositions based on the environment's default channel, which is defined in the environment's pointer file:

```yaml
# environments/production/pointer.yaml
apiVersion: forge.projectcatalyst.io/v1alpha1
kind: Pointer
metadata:
  name: production
spec:
  channel: stable  # Default channel for this environment
  overrides: {}    # Version pins (if any)
```

The pointer file's `channel` value is initially sourced from the environment's EnvironmentConfig during environment setup.

When the CMP processes an XR, it injects the appropriate selector:

```yaml
spec:
  crossplane:
    compositionUpdatePolicy: Automatic
    compositionRevisionSelector:
      matchLabels:
        channel: stable  # Injected from pointer file
```

XRs automatically track the latest revision labeled with their environment's channel. When a new version is promoted from dev to stable, XRs bind to the new revision without manual intervention.

### Controlled Promotion

Promoting a version from dev to stable is accomplished by publishing a stable release:

**Initial release to dev:**
```bash
# Tag and publish with pre-release identifier
git tag v1.1.0-dev
# Build and push packages with v1.1.0-dev tag
# CompositionRevision automatically labeled: channel=dev, release=v1.1.0-dev
```

**Promotion to stable (when ready):**
```bash
# Tag and publish without pre-release identifier
git tag v1.1.0
# Build and push packages with v1.1.0 tag
# CompositionRevision automatically labeled: channel=stable, release=v1.1.0
```

This creates a new CompositionRevision labeled `channel: stable`. All XRs selecting `channel: stable` will bind to the newly promoted revision on their next reconciliation.

The explicit publish step provides a promotion gate and creates an audit trail (Git tags show when versions were promoted from dev to stable).

**Note:** Not every dev version requires promotion to stable. You may iterate through multiple dev releases (e.g., `v1.1.0-dev`, `v1.1.1-dev`, `v1.1.2-dev`) before selecting one to promote to stable. Only versions that pass validation and are deemed production-ready should be published as stable releases.

### Manual Activation for Change Windows

Configuration packages support manual activation to defer when new versions become available:

```yaml
apiVersion: pkg.crossplane.io/v1
kind: Configuration
metadata:
  name: cfg-database
spec:
  package: registry.example.com/platform/cfg-database:v1.1.0
  revisionActivationPolicy: Manual
```

With manual activation:
1. Install or upgrade the Configuration package
2. New revision creates CompositionRevisions but they remain inactive
3. During your change window, activate the revision
4. CompositionRevisions become selectable by XRs

This provides an additional safety gate for high-risk changes or coordinated rollouts across multiple XRD families.

## Override Mechanism

### When to Use Overrides

Channel-based selection works well for most services, but some scenarios require explicit version pinning:

- Critical production services with zero-tolerance for unexpected changes
- Services undergoing incident response where stability trumps improvements
- Validation of specific versions before broader rollout
- Temporary workarounds for discovered issues in newer versions

### Pointer File Structure

Each environment's GitOps directory contains a `pointer.yaml` file that defines the default channel and version overrides:

```yaml
# environments/production/pointer.yaml
apiVersion: forge.projectcatalyst.io/v1alpha1
kind: Pointer
metadata:
  name: production
spec:
  channel: stable              # Default channel for this environment
  overrides:
    database-v1: v1.0.0        # Pin DatabaseV1 XRs to v1.0.0
    network-v1: v1.2.1         # Pin NetworkV1 XRs to v1.2.1
    # Other XRDs follow channel
```

### Override Behavior

When the CMP processes an XR:

1. Check if an override exists for this Composition in the environment's `pointer.yaml`
2. If override exists, inject `matchLabels: {release: <pinned-version>}` instead of channel
3. If no override exists, inject `matchLabels: {channel: <env-default>}` from pointer file

Example: With the pointer file above, a DatabaseV1 XR in production would receive:

```yaml
spec:
  crossplane:
    compositionUpdatePolicy: Automatic
    compositionRevisionSelector:
      matchLabels:
        release: v1.0.0  # Pinned via override
```

This XR will only bind to the CompositionRevision labeled `release: v1.0.0`, ignoring any channel progression.

### Removing Overrides

Overrides are temporary by design. To return an XR to channel-based selection, remove its entry from `pointer.yaml`. On the next reconciliation, the CMP will switch the XR back to selecting by the environment's default channel.

## Operational Procedures

### Releasing a New Version

**Step 1: Build and publish Function package (dev)**

From the function directory:
```bash
# Build function image using Earthfile for dev release
earthly +docker --tag=v1.1.0-dev

# Build and push function package
crossplane xpkg build --package-root=. --package-file=function.xpkg
crossplane xpkg push --package-file=function.xpkg registry.example.com/platform/db-func:v1.1.0-dev
```

**Step 2: Update Composition to reference new Function**

Create new Function object with versioned name:
```yaml
apiVersion: pkg.crossplane.io/v1
kind: Function
metadata:
  name: db-func-v1-1-0-dev
spec:
  package: registry.example.com/platform/db-func:v1.1.0-dev
```

Update Composition to reference new Function name:
```yaml
apiVersion: apiextensions.crossplane.io/v1
kind: Composition
metadata:
  name: database-v1
  labels:
    channel: dev           # Automatically applied based on version
    release: v1.1.0-dev    # Automatically applied based on version
spec:
  mode: Pipeline
  pipeline:
    - step: generate
      functionRef:
        name: db-func-v1-1-0-dev  # Updated reference
```

**Note:** The `channel` and `release` labels are automatically added to the Composition metadata during the build process based on the version string. Your CI/CD pipeline should parse the version tag and inject these labels before building the Configuration package.

**Step 3: Build and publish Configuration package (dev)**

```bash
crossplane xpkg build --package-root=. --package-file=config.xpkg
crossplane xpkg push --package-file=config.xpkg registry.example.com/platform/cfg-database-v1:v1.1.0-dev
```

**Step 4: Install Configuration in target environments**

Update the GitOps repository to reference the new Configuration version:

```yaml
# environments/dev/configurations/database-v1.yaml
apiVersion: pkg.crossplane.io/v1
kind: Configuration
metadata:
  name: cfg-database-v1
spec:
  package: registry.example.com/platform/cfg-database-v1:v1.1.0-dev
  revisionActivationPolicy: Automatic  # or Manual for controlled windows
```

Crossplane automatically creates a CompositionRevision with labels `channel: dev` and `release: v1.1.0-dev` based on the Composition metadata.

### Validating a New Version

Validation occurs naturally through environment-based channel selection. Development environments are configured to use the `dev` channel in their pointer files:

```yaml
# environments/dev/pointer.yaml
apiVersion: forge.projectcatalyst.io/v1alpha1
kind: Pointer
metadata:
  name: dev
spec:
  channel: dev  # Dev environment always tracks bleeding edge
  overrides: {}
```

When you publish a new dev version (e.g., `v1.1.0-dev`), all XRs in the dev environment automatically bind to the new revision. Validation consists of:

1. **Publish dev version** - New revision becomes available in dev environments
2. **Observe behavior** - Monitor applications using the new version in dev
3. **Verify correctness** - Ensure the version meets requirements
4. **Promote to stable** - When ready, publish stable version for higher environments

This natural progression ensures new versions are battle-tested in dev environments before reaching staging or production, which use the `stable` channel.

### Promoting to Stable

When a dev version is validated and ready for production:

**Step 1: Build and publish stable versions**

```bash
# Build function image with stable tag
earthly +docker --tag=v1.1.0

# Build and push function package (stable)
crossplane xpkg build --package-root=. --package-file=function.xpkg
crossplane xpkg push --package-file=function.xpkg registry.example.com/platform/db-func:v1.1.0
```

**Step 2: Create stable Function object**

```yaml
apiVersion: pkg.crossplane.io/v1
kind: Function
metadata:
  name: db-func-v1-1-0
spec:
  package: registry.example.com/platform/db-func:v1.1.0
```

**Step 3: Update Composition for stable**

```yaml
apiVersion: apiextensions.crossplane.io/v1
kind: Composition
metadata:
  name: database-v1
  labels:
    channel: stable       # Updated from dev
    release: v1.1.0       # Removed -dev suffix
spec:
  mode: Pipeline
  pipeline:
    - step: generate
      functionRef:
        name: db-func-v1-1-0  # Updated to stable function
```

**Note:** Your CI/CD pipeline automatically updates these labels based on the stable version tag (no `-dev` suffix).

**Step 4: Build and publish stable Configuration**

```bash
crossplane xpkg build --package-root=. --package-file=config.xpkg
crossplane xpkg push --package-file=config.xpkg registry.example.com/platform/cfg-database-v1:v1.1.0
```

**Step 5: Install stable Configuration**

Update GitOps to reference the stable version:

```yaml
# environments/production/configurations/database-v1.yaml
apiVersion: pkg.crossplane.io/v1
kind: Configuration
metadata:
  name: cfg-database-v1
spec:
  package: registry.example.com/platform/cfg-database-v1:v1.1.0
  revisionActivationPolicy: Manual  # Control activation window
```

When activated, Crossplane creates a new CompositionRevision labeled `channel: stable, release: v1.1.0`. All XRs selecting `channel: stable` will bind to this revision on their next reconciliation (typically within 60 seconds).

### Emergency Rollback

If a promoted version causes issues:

**Option 1: Pin affected services via override**
```yaml
# environments/production/pointer.yaml
spec:
  channel: stable
  overrides:
    database-v1: v1.0.0  # Pin back to known-good version
```

This immediately reverts affected XRs to the specified version. Remove the override when the issue is resolved.

**Option 2: Republish previous stable version**

If the issue affects all environments, republish the previous stable version:

```bash
# Rebuild and republish v1.0.0 as the current stable
# This creates a new CompositionRevision with channel: stable
crossplane xpkg push registry.example.com/platform/cfg-database-v1:v1.0.0
```

Install the reverted Configuration in affected environments and activate when ready.

## Best Practices

### Version Progression

- Always release to dev channel first (with `-dev` pre-release identifier)
- Validate in at least one production-like instance before promoting to stable
- Promotion requires explicit publish of stable version (no `-dev` suffix)
- Keep patch releases moving quickly through channels (they're low-risk by definition)
- Use Git tags to track when versions were promoted from dev to stable

### Major Version Changes

- Breaking changes require new Composition names (DatabaseV1 → DatabaseV2)
- Applications must explicitly opt into major version upgrades
- Keep previous major versions available during migration period
- Document migration paths in release notes

### Override Discipline

- Document why an override exists (commit message, issue reference)
- Set calendar reminders to revisit overrides
- Prefer channel-based selection for 95%+ of instances
- Use overrides for incident response, not permanent configuration

### Function Versioning

- Never modify an existing Function object's package reference in-place
- Always create new Function objects with versioned names (including pre-release suffix)
  - Dev: `db-func-v1-1-0-dev`
  - Stable: `db-func-v1-1-0`
- Keep old Function objects deployed while any Composition references them
- Garbage collect Function objects only after all references are removed
- Major version changes create new function families (e.g., `db-func-v2-0-0`)

### Configuration Dependencies

The Configuration package's `dependsOn` section ensures required Functions are available:

```yaml
spec:
  dependsOn:
    - apiVersion: pkg.crossplane.io/v1
      kind: Function
      package: registry.example.com/platform/db-func
      version: ">=v1.1.0"
```

This dependency is on the package (OCI reference), not the Function object name. The cluster-side Function object must still be created manually.

### Automatic Channel Labeling

Channel and release labels are automatically derived from the version string:

**Dev releases:**
```yaml
# version: v1.1.0-dev
metadata:
  labels:
    channel: dev
    release: v1.1.0-dev
```

**Stable releases:**
```yaml
# version: v1.1.0
metadata:
  labels:
    channel: stable
    release: v1.1.0
```

This automatic labeling is handled during the build/publish process. Never manually label CompositionRevisions - the labels should always reflect the published version.

## Repository Organization

### Publisher Repository (per XRD family)

```
database-v1/
  ├── crossplane.yaml              # Configuration manifest
  ├── database-v1-xrd.yaml         # XRD definition
  ├── database-v1-composition.yaml # Composition (pipeline mode)
  ├── function/
  │   ├── Earthfile                # Function image build
  │   ├── package/
  │   │   └── crossplane.yaml      # Function package manifest
  │   └── cmd/
  │       └── main.go              # Function implementation
  └── Makefile
```

### Platform GitOps Repository

```
environments/
  production/
    ├── configurations/
    │   └── database-v1.yaml       # Configuration installation
    ├── functions/
    │   ├── db-func-v1-0-0.yaml
    │   └── db-func-v1-1-0.yaml
    └── pointer.yaml               # Channel + version overrides
apps/
  my-service/
    └── xr/
        └── database.cue           # Rendered to XR YAML via CMP
```

The CMP processes `database.cue`, reads `pointer.yaml` to determine the default channel and any version overrides, then injects the appropriate composition selector into the rendered XR.

## Summary

The platform's Crossplane versioning strategy balances automatic improvement adoption with explicit version control:

- **Pre-release identifiers:** Dev releases use `-dev` suffix (e.g., `v1.1.0-dev`), stable releases omit it (e.g., `v1.1.0`)
- **Automatic labeling:** Channel labels are automatically derived from version strings during publication
- **Two-channel progression:** Changes move from dev to stable through explicit publish steps
- **Default behavior:** XRs automatically track channel-based versions via pointer file configuration
- **Selective control:** Overrides provide surgical version pinning when needed
- **Major versions:** Breaking changes require new Composition names (DatabaseV1 → DatabaseV2)
- **Change windows:** Manual Configuration activation defers when new versions become available
- **Operational simplicity:** CMP automation eliminates manual selector configuration while maintaining flexibility

This approach minimizes operational overhead for routine changes while preserving escape hatches for scenarios requiring strict version control.

---

**Related Documents:**
- Implementation Guide: Infrastructure Abstractions - XRD catalog and EnvironmentConfig specifications
- Developer Guide: Deployment Configuration - How applications declare infrastructure requirements

**Last Updated:** 2025-10-06