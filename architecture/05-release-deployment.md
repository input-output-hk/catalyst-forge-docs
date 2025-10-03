# Release & Deployment Specification

## Executive Summary
Defines release creation and environment progression, including release identification, artifact coordination, GitOps integration, and deployment workflows.

## Core Concepts

### Release Philosophy

#### Immutable Snapshots

Every release represents an immutable snapshot of a project at a specific commit. Once created, a release cannot be modified—only superseded by newer releases or redeployed to different environments. This immutability ensures reproducible deployments and enables reliable rollback operations.

#### Environment Agnosticism

Releases contain no environment-specific configuration. They package universal Kubernetes resource definitions that work across all environments, with Crossplane EnvironmentConfigs providing environment-specific values during deployment. This separation enables the same release to deploy to development, staging, and production without modification.

#### Policy-Driven Deployment

While all releases are equally valid snapshots, environment policies control where they can be deployed. The platform evaluates release metadata (trigger type, branch, tags) against environment-specific rules to enforce deployment standards without creating artificial release hierarchies.

### Release Identification & Aliasing

#### Canonical Identity

Every release has a unique canonical identifier composed of three parts:
- **Repository**: The source repository (e.g., `company/monorepo`)
- **Project**: The project within the repository (e.g., `auth-service`)
- **Commit SHA**: The full Git commit hash (e.g., `abc123def456...`)

This triplet guarantees global uniqueness: `<repository>/<project>/<commit-sha>`

#### OCI Image URI Format

The Release OCI image URI follows the canonical identifier pattern with a registry prefix:

```
<registry>/<repository>/<project>:<commit-sha>
```

For example: `registry.example.com/company/monorepo/auth-service:abc123def456...`

The key point is that the triplet (`repository/project/commit-sha`) uniquely identifies the release across all systems, while the registry prefix determines storage location.

#### Alias System

Releases support multiple aliases for human communication:

**Numeric Aliases**: Auto-incrementing counter per project, starting at 1. Provides simple reference like "Release 42 of auth-service."

**Tag Aliases**: Created from Git tags based on repository tagging strategy:
- `monorepo` strategy: Only tags matching `<project>/v*` create aliases
- `tag-all` strategy: Any tag creates aliases for all triggered projects

**Branch Aliases**: Optional aliases like `main-42` for releases from specific branches, useful for continuous deployment scenarios.

#### API Access Patterns

The API supports accessing releases through any valid identifier:

```
GET /api/v1/releases/<repo>/<project>/<release-id>
```

Where `<release-id>` can be:
- Full commit SHA: `abc123def456789...`
- Numeric alias: `42`
- Tag alias: `v1.2.3`
- Branch alias: `main-42`

Resolution follows a precedence order: exact alias match → numeric ID → SHA prefix → 404.

## Architecture

### Release Creation Workflow

The release creation and deployment flow follows distinct phases:

1. **Triggering**: Pipeline component evaluates release conditions and initiates release phase
2. **Artifact Building**: Artifacts component builds and publishes all configured artifacts
3. **Resource Rendering**: CUE configurations rendered to Kubernetes YAML with artifact references resolved
4. **OCI Packaging**: Resources and metadata packaged into Release OCI image
5. **Registration**: Release recorded in database with aliases and metadata
6. **Deployment**: Pointer files updated in GitOps repository based on policies
7. **Reconciliation**: Argo CD detects pointer changes and applies resources to cluster

#### Trigger Evaluation

Releases are triggered based on project configuration during pipeline execution:

```cue
release: {
  on: [
    {branch: "main"},        // Push to main branch
    {branch: "release/.*"},  // Push to release branches
    {tag: true}              // Tag creation (per repository strategy)
  ]
}
```

The pipeline discovery phase evaluates these conditions against the current Git context and marks projects for release creation.

#### Artifact Coordination

During the release phase, artifact references in CUE configurations are resolved to concrete values:

1. **Build Phase**: Artifacts are built and published, returning URIs and metadata
2. **Resolution Phase**: `@artifact("artifact-name.field")` references are replaced with actual values
3. **Rendering Phase**: CUE configurations are rendered to YAML with resolved values

Example transformation:
```cue
// Before resolution
spec: {
  image: @artifact("api-server.uri")
  imageDigest: @artifact("api-server.digest")
}

// After resolution
spec:
  image: "registry.company.com/auth-service:v1.2.3"
  imageDigest: "sha256:abc123..."
```

#### Resource Rendering

Kubernetes Resource Definitions are generated from CUE during the release phase:

1. CUE configurations evaluated with custom attributes resolved
2. Deployment resources extracted and converted to JSON/YAML
3. Resources packaged into Release OCI image
4. Argo CD extracts resources during deployment

#### Idempotency

Release creation is idempotent at the project-commit level. Attempting to create a release for an existing `<repository>/<project>/<commit-sha>` combination returns the existing release rather than creating a duplicate. This enables safe retries and simplifies error recovery.

### Release OCI Format

For complete OCI image format specifications, see [Core Architecture: OCI Image Formats - Release OCI Images](01-core-architecture.md#release-oci-images).

**Summary:**

**Image Structure**: Two-layer architecture with metadata and resources layers

**Metadata Layer Contains**:
- Release information (number, aliases, trigger)
- Source details (Git commit, author, message)
- Artifact references (including tracking OCI URIs)
- Resource summary

**Resources Layer Contains**:
- Rendered Kubernetes manifests as tar archive
- YAML files ready for cluster application
- Artifact references fully resolved

### GitOps Integration

The Release Component manages deployments through the platform's GitOps repository structure. For complete GitOps specifications, see [Core Architecture: GitOps Repository Structure](01-core-architecture.md#gitops-repository-structure).

#### Summary

**Repository Organization**: Hierarchical structure with environment-first organization

**Pointer Files**: Kubernetes-native `ReleasePointer` resources that reference release commit SHAs

**Argo CD Integration**: Custom Management Plugin extracts resources from Release OCI images

**Flow**:
1. Release Component updates pointer file in GitOps repository
2. Argo CD detects pointer file change
3. Custom Management Plugin fetches Release OCI by commit SHA
4. Plugin extracts and returns Kubernetes resources
5. Argo CD applies resources to target cluster

### Environment Promotion

#### Policies & Approvals

Environments define deployment policies stored as platform-level configuration:

```yaml
environments:
  dev:
    promotion_policy:
      allowed_triggers: ["branch_push", "tag", "manual"]
      allowed_branches: ["*"]
      auto_deploy: true

  staging:
    promotion_policy:
      allowed_triggers: ["branch_push", "tag", "manual"]
      allowed_branches: ["main", "release/*"]
      auto_deploy: false
      requires_approval: false

  production:
    promotion_policy:
      allowed_triggers: ["tag", "manual"]
      allowed_branches: ["main"]
      auto_deploy: false
      requires_approval: true
```

#### Deployment Workflows

Deployment follows a policy-driven workflow:

1. **Request**: API receives deployment request
2. **Policy Check**: Evaluate release trigger against environment policy
3. **Approval Check**: Verify approval exists if required
4. **GitOps Update**: Commit pointer file change
5. **Reconciliation**: Argo CD detects and applies changes

#### Rollback Operations

Rollbacks are implemented as forward deployments of older releases:

```bash
# Explicit rollback command
forge rollback production auth-service

# Or direct deployment of older release
forge deploy production auth-service v1.2.2
```

The system creates descriptive Git commits:
```
Rollback auth-service in production to release 41 (v1.2.3)

Previous release: 42 (v1.2.3)
Rolled back by: alice@company.com
```

## Configuration

For project-level release configuration, see [Configuration & Discovery: Project Configuration Schema - Release Configuration](02-configuration-discovery.md#project-configuration-schema).

Projects define release triggers in their CUE configuration:
```cue
release?: {
    on?: [
        {branch: "main"},
        {branch: "release/.*"},
        {tag: true}
    ]
    enabled?: true
}
```

## Operations

### Component Structure

The Release Component consists of three primary subsystems:

**Release Orchestrator** coordinates the entire release lifecycle. It manages artifact building, resource rendering, OCI packaging, and database persistence. The orchestrator ensures atomic release creation—either all components succeed or the entire release fails.

**OCI Builder** packages rendered Kubernetes resources and metadata into standardized OCI images. It creates a two-layer structure optimized for both human inspection and machine consumption, with metadata separated from resources for efficient retrieval.

**Deployment Manager** handles the progression of releases through environments. It evaluates deployment policies, manages approvals, and updates GitOps pointer files to trigger actual deployments via Argo CD.

### Error Handling

#### Release Creation Failures

Release creation follows fail-fast semantics. Any failure during artifact building, resource rendering, or OCI packaging causes the entire release to fail. The system maintains atomicity—partial releases are not persisted.

#### Deployment Failures

Deployment failures are handled at the GitOps level. If pointer file updates fail, the deployment is marked failed in the database. If Argo CD sync fails, the issue is visible in Argo CD UI while the pointer file remains updated.

#### Recovery Mechanisms

Failed releases can be retried by triggering a new pipeline run for the same commit. The idempotent design ensures existing successful components are reused rather than duplicated.

### Security Considerations

#### Release Integrity

Every Release OCI image includes digests for verification:
- Metadata layer digest in manifest
- Resources layer digest in metadata
- Individual artifact digests in metadata

#### Access Control

Release operations require authentication:
- Release creation: Pipeline service account only
- Deployment: User authentication with environment-specific permissions
- Approval: Named user with approval role for environment
- GitOps repository: Write access restricted to platform team

#### Audit Trail

Complete audit logging includes:
- Release creation events with actor and trigger
- Deployment operations with user and timestamp
- Approval decisions with justification
- Git commits provide immutable history

## Integration Points

For release OCI format standards, see [Core Architecture: OCI Image Formats](01-core-architecture.md#oci-image-formats).

For GitOps repository structure, see [Core Architecture: GitOps Repository Structure](01-core-architecture.md#gitops-repository-structure).

For pipeline and artifact integration, see [Execution & Orchestration](03-execution-orchestration.md) and [Build & Distribution](04-build-distribution.md).

For cross-component integration patterns, see [Integration Contracts](07-integration-contracts.md).

### Component Interactions

**Pipeline Component**: Triggers release creation during the release phase. Provides workspace, Git context, and execution environment. Release Component reports success/failure, updating pipeline status.

**Artifacts Component**: Artifacts are built exclusively during the release phase. Release Component coordinates with Artifacts Component to build, publish, and track all configured artifacts before packaging the release.

**Project Component**: Release configurations are defined in project CUE files. Release Component reads deployment resources, resolves artifact references, and renders final Kubernetes manifests from CUE specifications.

**Argo CD**: Consumes Release OCI images through its Custom Management Plugin. The plugin extracts resources from the OCI image based on pointer file references and applies them to target clusters.

## Reference

### API Endpoints

For complete API specifications, see [Domain Model & API Reference: Release APIs](06-domain-model-api-reference.md#release-apis).

**Release Endpoints**:
- `POST /v1/releases` - Create release
- `GET /v1/releases/{repo}/{project}/{release-id}` - Get release
- `GET /v1/releases/{repo}/{project}` - List releases

**Deployment Endpoints**:
- `POST /v1/deployments` - Create deployment
- `GET /v1/deployments/{environment}/{repo}/{project}` - Get deployment status

**Approval Endpoints**:
- `POST /v1/approvals` - Create approval
- `GET /v1/approvals/{release-id}/{environment}` - Get approval status

### Domain Model

For complete domain entity specifications, see [Domain Model & API Reference: Release Entities](06-domain-model-api-reference.md#release-entities).

**Core Entities:**
- Release
- ReleaseAlias
- ReleaseTrigger
- ReleaseArtifact
- Deployment
- ReleaseApproval

## Examples

For release and deployment examples, see [Integration Contracts: Release Creation and Deployment Flow](07-integration-contracts.md#release-creation-and-deployment-flow).

## Constraints & Limitations

### Architectural Constraints

- One release per project per commit (no multi-project releases)
- Releases are immutable once created
- Environment policies are global, not project-specific
- Maximum OCI image size limited by registry (typically 10GB)

### Operational Constraints

- No direct rollback mechanism (implemented as redeployment)
- Approvals are mutable database records
- GitOps repository must be centrally managed
- Artifact references must be resolved at release time

## Future Enhancements

### Planned Capabilities

**Release Composition**: Support for multi-project release groups that deploy together while maintaining individual release identities.

**Dynamic Environments**: On-demand environment creation for feature branches with automatic cleanup.

**Progressive Deployment**: Integration with Flagger or Argo Rollouts for canary and blue-green deployments.

**Enhanced Approvals**: Multi-party approval requirements with quorum support and automated approval based on quality gates.

### Extension Points

The architecture provides clear extension points:
- Additional alias types through the alias system
- Custom metadata fields in the OCI metadata layer
- Alternative GitOps tools through different pointer formats
- Enhanced policy rules through environment configuration

---

**Component Owner**: Release & Deployment
**Shared Concepts**: See [Core Architecture: OCI Image Formats](01-core-architecture.md#oci-image-formats), [GitOps Repository Structure](01-core-architecture.md#gitops-repository-structure)
**Domain Model**: See [Domain Model & API Reference: Release Entities](06-domain-model-api-reference.md#release-entities)
**Last Updated**: 2025-10-02
