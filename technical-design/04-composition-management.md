# Composition Management Design

## Executive Summary

Technical design for managing XRD and Composition evolution in the platform

## Core Concepts

### Versioning Requirements

**Why our compositions need versions**:

Infrastructure abstractions evolve over time. Bug fixes, new features, and behavioral changes require a versioning strategy that balances two competing needs:

1. **Default safety**: Most applications should receive improvements automatically without manual intervention
2. **Explicit control**: Critical services need deterministic behavior and controlled upgrade windows

Traditional approaches force an all-or-nothing choice: either all instances upgrade automatically (risky) or all instances require manual intervention (operationally expensive). Our strategy provides both through channel-based progression with selective pinning.

**Stability guarantees we provide**:

- **Within major version**: All releases are backward compatible (minor and patch releases)
- **Across major versions**: Breaking changes require explicit opt-in (changing XR kind)
- **Channel isolation**: Dev channel changes don't affect stable channel XRs
- **Version pinning**: Override mechanism provides surgical control when needed
- **Immutable revisions**: CompositionRevisions are immutable once created
- **Grace period**: Minimum 6 months between deprecation and removal of major versions

**Evolution without breaking applications**:

The platform enables composition evolution while maintaining application stability:

1. **Non-breaking changes**: Progress automatically via channel-based selection
   - Minor/patch releases add features without breaking compatibility
   - Applications receive improvements without code changes
   - Channel progression (dev → stable) provides gradual rollout

2. **Breaking changes**: Require explicit migration
   - New major version = new Composition name (DatabaseV1 → DatabaseV2)
   - Applications must change XR kind field to opt in
   - Previous major version remains available during grace period
   - Teams migrate on their own schedule

3. **Emergency control**: Override mechanism for incident response
   - Pin specific services to exact versions
   - Revert to known-good versions immediately
   - Temporary escape hatch from channel-based selection

This approach treats versioning as a product feature, not an operational burden.

**Note**: For general Crossplane composition concepts, see [Crossplane Composition Documentation](https://docs.crossplane.io/latest/concepts/compositions/)

## Our Versioning Scheme

### Version Number Design

**Format**: Semantic versioning (major.minor.patch) with pre-release identifiers

Our platform uses semantic versioning with platform-specific interpretation:

- **Major version (v2.0.0)**: Breaking changes require a new Composition name (DatabaseV1 → DatabaseV2). Applications must explicitly migrate by changing their XR's `kind` field.
- **Minor version (v1.1.0)**: New features, non-breaking changes, or significant behavioral improvements
- **Patch version (v1.0.1)**: Bug fixes and minor improvements with no observable behavioral changes

**Pre-release identifiers** distinguish dev from stable releases:
- **Dev releases**: `v1.0.0-dev`, `v1.1.0-dev` - pre-release versions for validation
- **Stable releases**: `v1.0.0`, `v1.1.0` - production-ready versions

Pre-release versions automatically sort before their stable counterparts (`v1.0.0-dev < v1.0.0`).

**Version Alignment**: All three packages (Configuration, Function package, Function object) share the same semantic version. When releasing DatabaseV1 v1.2.0-dev:
- Configuration package is tagged `v1.2.0-dev`
- Function package is tagged `v1.2.0-dev`
- Function object is named `db-func-v1-2-0-dev` and references `v1.2.0-dev` package

This alignment simplifies reasoning about compatibility and ensures Compositions always reference compatible Functions.

### Version Selection

**Channel-based selection**: XRs select compositions through channel labels rather than explicit version numbers.

The platform uses two channels representing rollout stages:
- **dev**: Pre-release versions for validation and testing. Can be used in any environment.
- **stable**: Production-ready versions that have been validated.

This model decouples infrastructure version progression from environment promotion. You can safely test new versions in production by creating a single dev-channel instance.

**Default selection mechanism**: The Argo CD Custom Management Plugin (CMP) automatically configures XRs to select compositions based on the environment's default channel, which is defined in the environment's version configuration file:

```yaml
# environments/production/version.yaml
apiVersion: forge.projectcatalyst.io/v1alpha1
kind: VersionConfig
metadata:
  name: production
spec:
  channel: stable  # Default channel for this environment
  overrides: {}    # Version pins (if any)
```

When the CMP processes an XR, it injects the appropriate selector:

```yaml
spec:
  crossplane:
    compositionUpdatePolicy: Automatic
    compositionRevisionSelector:
      matchLabels:
        channel: stable  # Injected from version file
```

XRs automatically track the latest revision labeled with their environment's channel.

**Override mechanism**: Each environment's version configuration can override the channel selection for specific compositions:

```yaml
spec:
  channel: stable              # Default channel for this environment
  overrides:
    database-v1: v1.0.0        # Pin DatabaseV1 XRs to v1.0.0
    network-v1: v1.2.1         # Pin NetworkV1 XRs to v1.2.1
```

When the CMP processes an XR:
1. Check if an override exists for this Composition in the environment's version.yaml
2. If override exists, inject `matchLabels: {release: <pinned-version>}` instead of channel
3. If no override exists, inject `matchLabels: {channel: <env-default>}` from version configuration

Overrides are temporary by design. To return an XR to channel-based selection, remove its entry from version.yaml.

### Storage and Naming

**Composition naming convention**: Major versions are reflected in the Composition name itself:
- `DatabaseV1` for v1.x.x releases
- `DatabaseV2` for v2.x.x releases (breaking changes)
- `NetworkV1`, `NetworkV2`, etc.

This approach makes breaking changes explicit and prevents accidental upgrades. Within a major version, all releases are backward compatible.

**Version tracking through labels**: Every change to a Composition creates an immutable CompositionRevision. These revisions are labeled with metadata that enables selection:

```yaml
apiVersion: apiextensions.crossplane.io/v1
kind: Composition
metadata:
  name: database-v1
  labels:
    channel: dev           # Automatically applied based on version
    release: v1.1.0-dev    # Automatically applied based on version
```

Labels are automatically applied based on the version's pre-release identifier during publication:
- **Dev releases** (`v1.1.0-dev`): `channel: dev`, `release: v1.1.0-dev`
- **Stable releases** (`v1.1.0`): `channel: stable`, `release: v1.1.0`

**Function object naming**: Function objects use encoded version names for side-by-side execution:
- Dev: `db-func-v1-1-0-dev` (references package `v1.1.0-dev`)
- Stable: `db-func-v1-1-0` (references package `v1.1.0`)
- Major version changes: `db-func-v2-0-0` (new function family)

This allows multiple versions to exist simultaneously without conflicts.

## Breaking vs Non-Breaking Changes

### Our Breaking Change Definition

**Breaking changes** require a new major version (and thus a new Composition name):

- **XRD schema changes** that remove or rename required fields
- **XRD schema changes** that change field types in incompatible ways
- **Behavioral changes** that alter the meaning or outcome of existing configurations
- **Composition changes** that modify resource generation in incompatible ways (e.g., changing resource naming patterns, removing resources)

**Non-breaking changes** can be released as minor or patch versions within the same major version:

- **Adding optional fields** to XRD schemas
- **Expanding enum values** (adding new allowed values)
- **Bug fixes** that correct incorrect behavior to match documented intent
- **Performance improvements** with no observable behavioral changes
- **Adding new resources** to Compositions without removing existing ones
- **Enhancing validation** that only rejects previously invalid configurations

**Rationale**: This categorization prioritizes stability for existing applications. Breaking changes must be explicit and opt-in, while non-breaking changes can progress automatically through channels.

### Examples from Platform XRDs

**Deployment XRD - Breaking Change Scenario**:
- Changing `replicas` field from integer to object with `min`/`max`/`desired` fields
- Requires new major version: DeploymentV1 → DeploymentV2
- Applications must explicitly migrate by updating their XR `kind`

**Network XRD - Non-Breaking Change Scenario**:
- Adding optional `timeoutSeconds` field to ingress configuration
- Released as minor version update (v1.1.0)
- Existing applications continue working without modification
- Applications can adopt new field incrementally

**Database XRD - Non-Breaking Evolution**:
- Adding new `performanceTier` enum value "ultra" to existing ["standard", "high"]
- Released as minor version (v1.2.0)
- Existing databases with "standard" or "high" continue working
- New applications can use "ultra" tier

**Database XRD - Breaking Change Scenario**:
- Removing support for deprecated `legacyBackupFormat` field
- Requires new major version: DatabaseV1 → DatabaseV2
- Provides migration period where V1 remains available

## Backward Compatibility Approach

### Multi-Version Support

**Simultaneous version support**: Yes, the platform supports multiple major versions of the same XRD family running simultaneously:

- **Major versions coexist**: DatabaseV1 and DatabaseV2 can both be installed and active
- **Side-by-side execution**: Function objects with versioned names allow multiple function versions to run concurrently
- **Independent lifecycles**: Each major version has its own channel progression (dev/stable)

**How applications select versions**:

1. **Implicit (channel-based)**: Most applications use channel-based selection within their major version
   - XR specifies `kind: Database` (implicitly using DatabaseV1)
   - VersionConfig determines channel: `channel: stable`
   - XR automatically gets latest stable revision of DatabaseV1

2. **Explicit (version pinning)**: Critical applications can pin to specific versions via version overrides
   - VersionConfig specifies: `database-v1: v1.2.0`
   - XR binds to exact revision labeled `release: v1.2.0`
   - Remains pinned until override is removed

3. **Explicit (major version)**: Applications explicitly choose major version by XR kind
   - Change `kind: Database` to `kind: DatabaseV2`
   - Application now tracks DatabaseV2 channel/version instead

**Default progression strategy**:
- Within a major version: Automatic progression via channel-based selection
- Across major versions: Explicit opt-in by changing XR kind
- Override mechanism available for surgical control when needed

### Deprecation Strategy

**Deprecation marking mechanism**:

Deprecated compositions use labels and annotations to signal lifecycle status:

```yaml
apiVersion: apiextensions.crossplane.io/v1
kind: Composition
metadata:
  name: database-v1
  labels:
    deprecated: "true"
  annotations:
    deprecated-date: "2025-06-01"
    removal-date: "2025-12-01"
```

**Deprecation metadata**:
- **`deprecated` label**: Marks composition as deprecated (machine-readable signal)
- **`deprecated-date` annotation**: Records when deprecation was announced
- **`removal-date` annotation**: Specifies planned removal date
- **Release notes**: Human-readable deprecation announcement

**Grace period design**:

The platform enforces minimum 6-month grace period between deprecation and removal:

- **Parallel version support**: Deprecated and replacement versions both active during transition
- **Migration guidance**: New major version documentation includes migration path
- **Usage tracking**: Platform metrics monitor deprecated version adoption
- **Notification mechanism**: Teams using deprecated versions receive migration reminders

**Rationale**:
- Provides sufficient time for teams to plan and execute migrations
- Enables gradual rollout without forcing simultaneous updates
- Maintains stability while allowing platform evolution

## XRD Schema Evolution

### Field Changes

**Adding optional fields** (non-breaking, minor version):

Fields can be added to XRD schemas as optional without breaking compatibility:

```yaml
# v1.0.0 schema
spec:
  type: object
  properties:
    replicas:
      type: integer

# v1.1.0 schema - added optional field
spec:
  type: object
  properties:
    replicas:
      type: integer
    maxReplicas:  # New optional field
      type: integer
```

- Released as minor version (v1.0.0 → v1.1.0)
- Existing XRs without `maxReplicas` continue working
- Composition must handle missing field gracefully (use default or skip feature)

**Deprecating fields in schemas** (non-breaking, minor version):

Fields are marked deprecated but remain functional:

```yaml
spec:
  type: object
  properties:
    replicas:
      type: integer
      description: "DEPRECATED: Use minReplicas/maxReplicas instead. Will be removed in v2.0.0"
    minReplicas:
      type: integer
    maxReplicas:
      type: integer
```

- Add deprecation notice to field description
- Field continues to work in all v1.x.x versions
- Composition implements both old and new field logic during transition
- Documentation explains migration path

**Removing fields** (breaking change, major version):

Field removal requires new major version:

```yaml
# DatabaseV2 schema - removed deprecated field
spec:
  type: object
  properties:
    # replicas field removed
    minReplicas:
      type: integer
    maxReplicas:
      type: integer
```

- Only allowed in new major version (DatabaseV1 → DatabaseV2)
- Previous major version remains available during grace period
- Migration guide documents how to update XRs

### Validation Evolution

**How validation rules can change**:

1. **Stricter validation** (potentially breaking - evaluate carefully):
   - Adding `required` fields → Breaking (major version)
   - Reducing allowed enum values → Breaking (major version)
   - Adding stricter regex patterns → Potentially breaking (evaluate existing usage)

2. **Relaxed validation** (non-breaking):
   - Removing `required` constraint → Minor version
   - Adding new enum values → Minor version
   - Loosening regex patterns → Minor version
   - Increasing maximum values → Minor version

**Compatibility determination**:

Platform engineers must evaluate whether validation changes are breaking:

- If existing XRs would fail new validation → Breaking change (requires major version)
- If all existing XRs pass new validation → Non-breaking change (minor version allowed)
- Evaluation performed manually before releasing schema changes

## Platform-Specific Patterns

### Our Composition Naming

**Convention**: `{xrd-family}-v{major-version}`

Examples:
- `database-v1` for DatabaseV1 (all v1.x.x releases)
- `database-v2` for DatabaseV2 (all v2.x.x releases)
- `network-v1` for NetworkV1
- `deployment-v1` for DeploymentV1

**Rationale**:
- Major version in name prevents accidental breaking changes
- Single composition name per major version simplifies management
- Minor/patch versions differentiated by CompositionRevision labels

**Matching to XRD instances**:

XRDs reference compositions by name in their `defaultCompositionRef`:

```yaml
apiVersion: apiextensions.crossplane.io/v1
kind: CompositeResourceDefinition
metadata:
  name: databases.forge.projectcatalyst.io
spec:
  group: forge.projectcatalyst.io
  names:
    kind: DatabaseV1
    plural: databases
  defaultCompositionRef:
    name: database-v1  # References our composition
```

XRs then select specific revisions via label selectors (injected by CMP):

```yaml
apiVersion: forge.projectcatalyst.io/v1alpha1
kind: Database
spec:
  crossplane:
    compositionUpdatePolicy: Automatic
    compositionRevisionSelector:
      matchLabels:
        channel: stable  # or release: v1.2.0 for pinned versions
```

**Label selectors we use**:

Two selector patterns:

1. **Channel-based** (default):
   ```yaml
   matchLabels:
     channel: stable  # or dev
   ```
   - Automatically tracks latest revision in channel
   - Used for 95%+ of XRs

2. **Version-pinned** (override):
   ```yaml
   matchLabels:
     release: v1.2.0  # exact version
   ```
   - Binds to specific revision
   - Used for critical services or during incidents

### Function Pipeline Versioning

**Our composition functions structure**:

All compositions use pipeline mode with versioned function references:

```yaml
apiVersion: apiextensions.crossplane.io/v1
kind: Composition
metadata:
  name: database-v1
spec:
  mode: Pipeline
  pipeline:
    - step: generate-resources
      functionRef:
        name: db-func-v1-2-0  # Versioned function object
    - step: patch-and-transform
      functionRef:
        name: db-func-v1-2-0  # Same version
```

**Versioned function objects**:

Each function version is a separate cluster resource:

```yaml
# Dev version
apiVersion: pkg.crossplane.io/v1
kind: Function
metadata:
  name: db-func-v1-2-0-dev
spec:
  package: registry.example.com/platform/db-func:v1.2.0-dev

# Stable version
apiVersion: pkg.crossplane.io/v1
kind: Function
metadata:
  name: db-func-v1-2-0
spec:
  package: registry.example.com/platform/db-func:v1.2.0
```

**Compatibility requirements**:

1. **Package alignment**: Configuration package version must match Function package version
   - Config v1.2.0-dev references Function v1.2.0-dev
   - Config v1.2.0 references Function v1.2.0

2. **Function object availability**: Function object must exist before Configuration is activated
   - Function objects created manually in cluster
   - Not bundled in Configuration packages
   - CI/CD pipeline ensures Function deployed before Configuration

3. **Input/output contract stability**:
   - Within major version (v1.x.x): Function input/output schemas remain compatible
   - Across major versions: Function can change input/output contract
   - Breaking function changes require new major version (db-func-v2-x-x)

4. **Side-by-side execution**: Multiple function versions run concurrently
   - v1.1.0 and v1.2.0 functions both active
   - Enables gradual migration between versions
   - Old functions removed only after all Compositions stop referencing them


## Design Decisions

### Why Allow Breaking Changes

**Platform evolution necessity**:

Infrastructure platforms must evolve to meet changing requirements:
- New cloud provider features require schema updates
- Security improvements may require behavioral changes
- Performance optimizations might alter resource structure
- Industry best practices evolve over time

Forbidding breaking changes would force:
- Accumulation of deprecated patterns indefinitely
- Complex workarounds to maintain compatibility
- Bloated schemas with unused fields
- Technical debt that degrades platform quality

**Controlled vs uncontrolled breakage**:

Our approach provides controlled breaking changes:

1. **Uncontrolled** (what we avoid):
   - In-place composition updates that break existing XRs
   - Silent behavioral changes without version indication
   - Breaking changes in patch releases

2. **Controlled** (what we implement):
   - Breaking changes only in new major versions
   - Explicit opt-in via XR kind change (Database → DatabaseV2)
   - Grace period with both versions available
   - Clear migration documentation

**Migration window approach**:

The platform provides structured migration:

- **6-month minimum grace period** between deprecation and removal
- **Parallel version support**: Both DatabaseV1 and DatabaseV2 active during transition
- **Selective migration**: Teams migrate on their schedule within grace period
- **Rollback capability**: Can revert to V1 if issues arise during migration

This balances platform evolution with application stability.

### Why This Version Scheme

**Semantic versioning rationale**:

Semantic versioning provides clear contract with users:
- **Major version change**: Expect breaking changes, plan migration
- **Minor version change**: New features available, no action required
- **Patch version change**: Bug fixes, safe to adopt automatically

This clarity enables:
- Automatic progression for safe changes (minor/patch)
- Explicit control for breaking changes (major)
- Predictable platform behavior

**Alignment with industry standards**:

Platform uses standard semantic versioning with pre-release identifiers:
- Follows semver.org specification
- Compatible with Crossplane's versioning model
- Matches OCI package versioning conventions
- Familiar to platform engineers from other ecosystems

**Tooling compatibility**:

Our version scheme integrates with existing tools:

1. **Git tags**: Standard format `v1.2.0` and `v1.2.0-dev`
2. **OCI registries**: Support pre-release tags natively
3. **Crossplane**: CompositionRevision selection via labels
4. **CI/CD**: Automated label injection based on version parsing

This reduces custom tooling and leverages mature ecosystems.

## Package Structure

### Three-Package Coordination

Each XRD family ships as three coordinated packages:

**1. Configuration Package (XRD + Composition)**:
- Contains XRD definition and Composition
- Tagged with semantic version (v1.0.0, v1.1.0-dev, etc.)
- Declares dependency on Function package
- Published to OCI registry
- Installed via Crossplane Configuration resource

**2. Function Package**:
- Contains function runtime implementation
- Tagged with matching semantic version
- Published to OCI registry
- Built from Earthfile
- Referenced by Function object

**3. Function Object (Cluster Resource)**:
- Kubernetes resource that references Function package
- Named with encoded version (e.g., `db-func-v1-0-0`)
- Allows side-by-side execution of multiple versions
- Created manually in cluster, not bundled in packages

All three share the same semantic version for alignment.

### Channel Progression and Promotion

**Channel model**:

Channels represent rollout stages, not environments:
- **dev**: Pre-release versions (v1.1.0-dev) for validation
- **stable**: Production-ready versions (v1.1.0) that have been validated

This decouples infrastructure version progression from environment promotion.

**Promotion mechanism**:

Versions progress from dev to stable through explicit publish steps:

1. **Initial release to dev**:
   - Tag: `v1.1.0-dev`
   - Build and push packages with `-dev` suffix
   - CompositionRevision labeled: `channel: dev`, `release: v1.1.0-dev`

2. **Promotion to stable** (when validated):
   - Tag: `v1.1.0` (no suffix)
   - Build and push packages without `-dev`
   - CompositionRevision labeled: `channel: stable`, `release: v1.1.0`

This creates new CompositionRevision with stable label. XRs selecting `channel: stable` bind to new revision automatically.

**Manual activation for change windows**:

Configuration packages support manual activation to defer availability:

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
1. Install/upgrade Configuration package
2. New revision creates CompositionRevisions but they remain inactive
3. During change window, activate revision
4. CompositionRevisions become selectable by XRs

Provides additional safety gate for high-risk changes.

## Integration with Platform Components

### CMP (Custom Management Plugin) Integration

**Version configuration sourcing**:

The CMP reads each environment's version configuration to determine version selection:

```yaml
# environments/production/version.yaml
apiVersion: forge.projectcatalyst.io/v1alpha1
kind: VersionConfig
metadata:
  name: production
spec:
  channel: stable              # Default channel from EnvironmentConfig
  overrides:
    database-v1: v1.0.0        # Optional version pins
```

The `channel` value is initially sourced from the environment's EnvironmentConfig during setup.

**Selector injection**:

When the CMP processes resources from Release OCI images, it:

1. Extracts YAML resources from Release OCI image
2. Reads environment's version.yaml file
3. For each XR in the resources, checks for version override
4. Injects appropriate selector into XR spec:
   - If override exists: `matchLabels: {release: <version>}`
   - If no override: `matchLabels: {channel: <env-channel>}`

This ensures XRs automatically track the correct composition version based on environment configuration.

## Future Considerations

- **Automated compatibility testing**: Expand test matrix to cover more edge cases
- **Version migration tooling**: Build automated XR migration tools for major version upgrades
- **Policy enforcement mechanisms**: OPA policies to enforce versioning rules at admission time
- **Drift detection**: Alert when XR versions diverge from environment defaults
- **Version analytics**: Track adoption rates of new versions across platform
