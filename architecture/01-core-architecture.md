# Core Architecture

## Executive Summary
This document serves as the single source of truth for system-wide concepts, architectural decisions, and shared patterns across the Catalyst Forge platform. It provides the foundational understanding needed to work with any component of the system.

Catalyst Forge is an opinionated internal developer platform that manages the complete software development lifecycle from commit to deployment. Built for a team of 15-20 engineers with capacity for growth, the platform provides a streamlined, evolutionary approach to software delivery that balances developer autonomy with operational consistency.

## Core Concepts

### System Overview & Philosophy

#### What Catalyst Forge Is

Catalyst Forge manages the complete SDLC from commit to deployment for a growing startup organization. The platform enforces conventions where consistency matters (build systems, deployment targets, configuration patterns) while providing flexibility where teams need it (tool selection within phases, deployment strategies, release cadences). This approach reduces cognitive load for common cases while maintaining escape hatches for complex requirements.

**Scope**: Complete SDLC coverage for ~15-20 engineers with built-in capacity for growth
**Infrastructure**: Kubernetes-native on AWS (cloud) and Hetzner (bare-metal)

#### What Catalyst Forge Is Not

- Not a multi-cloud platform (AWS and Hetzner only)
- Not build-system agnostic (Earthly is required)
- Not a general-purpose CI/CD system with infinite flexibility
- Not a multi-orchestrator solution (Kubernetes only)

### Architectural Principles & Constraints

#### 1. Convention with Escape Hatches

The platform mandates workflow structure but not workflow contents. Teams must use Earthly for builds and deploy to Kubernetes, but they control how they configure their builds and what runs within each phase. Deployment abstractions through Crossplane are encouraged but raw Kubernetes resources remain available. This principle ensures consistency at architectural boundaries while preserving team autonomy for implementation details.

#### 2. Progressive Disclosure of Complexity

Projects start simple and evolve with need. A new project can hook into continuous integration with minimal configuration, then progressively adopt artifact publishing, release management, and automated deployments as it matures. Deployment complexity is hidden behind Crossplane Composite Resources that developers instantiate with minimal inputs. The platform grows with the project rather than forcing upfront complexity.

#### 3. Built for Change, Not Forever

The platform uses ports and adapters patterns to ensure technology choices can evolve. Core workflows remain stable while underlying implementations can be swapped as technologies mature or requirements change. Today's Argo CD could become tomorrow's Flux without disrupting developer workflows.

#### 4. Sequential Phase Execution with Fail-Fast

The platform enforces sequential phase execution with fail-fast behavior. Phases execute in group order, with only one phase group active at a time. Any failure immediately terminates the pipeline, aligning with the principle that CI must be "green" before proceeding.

#### 5. Immutability

Releases represent immutable snapshots of projects at specific commits. Once created, a release cannot be modified—only superseded by newer releases or redeployed to different environments. This immutability ensures reproducible deployments and enables reliable rollback operations.

#### 6. Environment Agnosticism

Releases contain no environment-specific configuration. They package universal Kubernetes resource definitions that work across all environments, with Crossplane EnvironmentConfigs providing environment-specific values during deployment. This separation enables the same release to deploy to development, staging, and production without modification.

### Component Architecture & Dependencies

The platform consists of four primary components that work together to provide complete SDLC management:

#### Pipeline Component
Discovers projects, constructs dynamic CI pipelines, and orchestrates build execution. Consists of an API server for state management, Argo Workflows for orchestration, and hot worker pools for actual execution.

**Key Responsibilities**:
- Repository and project discovery
- DAG construction from configurations
- Phase orchestration and execution
- State persistence and audit trails
- Worker coordination and monitoring

#### Project Component
Defines the configuration schema and discovery mechanisms for projects within GitHub repositories. Projects are identified by `.forge` folders containing CUE configuration files that include deployment specifications.

**Key Responsibilities**:
- Configuration schema definition (CUE)
- Project discovery mechanisms
- Repository-level settings management
- Deployment specification validation
- Configuration to Kubernetes resource transformation (primarily XRDs)

#### Artifacts Component
Manages build outputs and their publication to registries during the release phase. All artifacts are tracked using OCI images for universal provenance and metadata management.

**Key Responsibilities**:
- Artifact type management (container, binary, archive)
- Publisher plugin coordination
- Artifact upload orchestration (within release phase)
- OCI tracking image creation for provenance
- Registry credential management

#### Release & Deployment Component
Creates point-in-time snapshots of projects including their Kubernetes resource definitions (KRDs). Projects primarily leverage platform-provided Crossplane Composite Resources (XRs) defined in the infrastructure abstractions catalog, though raw Kubernetes resources are also supported. Integrates with Argo CD for GitOps-based deployments.

For infrastructure abstraction specifications, see [Infrastructure Abstractions](08-infrastructure-abstractions.md).

**Key Responsibilities**:
- Release trigger evaluation
- Snapshot creation including Kubernetes resource definitions
- OCI image packaging for GitOps consumption
- Environment progression rules
- Promotion and approval workflows

### Component Architecture & Dependencies

```
┌──────────────────────────────────────────────────────────────────────────┐
│                         External Dependencies                            │
│  ┌──────────┐  ┌────────────┐  ┌─────────┐  ┌──────────────────────┐     │
│  │  GitHub  │  │ PostgreSQL │  │ Argo CD │  │   AWS Services       │     │
│  │          │  │            │  │         │  │ (SQS/DynamoDB/S3/SM) │     │
│  └──────────┘  └────────────┘  └─────────┘  └──────────────────────┘     │
└──────────────────────────────────────────────────────────────────────────┘
                                    ▲
                                    │
┌───────────────────────────────────┼──────────────────────────────────────┐
│                    Platform Components                                   │
│                                   │                                      │
│  ┌──────────────────────────────────────────────────────────────┐        │
│  │           Pipeline Component (Orchestrator)                  │        │
│  │  ┌──────────────┐  ┌────────────────┐  ┌─────────────────┐   │        │
│  │  │  API Server  │  │ Argo Workflows │  │ Hot Worker Pools│   │        │
│  │  └──────────────┘  └────────────────┘  └─────────────────┘   │        │
│  │                                                              │        │
│  │  • Repository discovery     • Phase orchestration            │        │
│  │  • DAG construction         • State persistence              │        │
│  └──────────────────┬───────────────────┬───────────────────────┘        │
│                     │                   │                                │
│                     │ orchestrates      │ orchestrates                   │
│                     ▼                   ▼                                │
│  ┌──────────────────────────┐    ┌──────────────────────────────────┐    │
│  │  Project Component       │    │  Artifacts Component             │    │
│  │                          │    │                                  │    │
│  │  • CUE schema            │    │  • Artifact type management      │    │
│  │    definitions           │    │  • Publisher coordination        │    │
│  │  • Project discovery     │    │  • OCI tracking image creation   │    │
│  │  • Deployment spec       │    │  • Upload orchestration          │    │
│  │    validation            │    │                                  │    │
│  └──────┬───────────────────┘    └─────────────┬────────────────────┘    │
│         │                                      │                         │
│         │ provides config                      │                         │
│         │                                      │                         │
│         │                   ┌──────────────────┘                         │
│         │                   │                                            │
│         │                   │ provides artifacts                         │
│         │                   │                                            │
│         └───────────────────┼───────────────────┐                        │
│                             ▼                   │                        │
│                  ┌──────────────────────────────┴──────┐                 │
│                  │  Release & Deployment Component     │                 │
│                  │                                     │                 │
│                  │  • Release trigger evaluation       │                 │
│                  │  • Snapshot creation (KRDs)         │                 │
│                  │  • OCI packaging                    │                 │
│                  │  • Environment progression          │                 │
│                  └─────────────────────────────────────┘                 │
│                                                                          │
│  Data Flow:                                                              │
│    Pipeline → orchestrates all components                                │
│    Project → provides configuration to all components                    │
│    Artifacts → operates during release phase                             │
│    Release → coordinates Artifacts and Project data                      │
│    All components → use AWS services and PostgreSQL                      │
└──────────────────────────────────────────────────────────────────────────┘
```

### Unified Data Flow

The platform processes work through the following high-level flow from commit to deployment:

```
┌────────────┐   ┌────────────┐   ┌────────────┐   ┌────────────┐   ┌────────────┐   ┌────────────┐   ┌────────────┐
│   Step 1   │──▶│   Step 2   │──▶│   Step 3   │──▶│   Step 4   │──▶│   Step 5   │──▶│   Step 6   │──▶│   Step 7   │
│  Trigger   │   │ Discovery  │   │   Phase    │   │  Release   │   │   GitOps   │   │  Resource  │   │Reconcilia- │
│            │   │            │   │ Execution  │   │  Creation  │   │   Update   │   │ Extraction │   │   tion     │
└────────────┘   └────────────┘   └────────────┘   └─ ─ ─ ─ ─ ─ ┘   └────────────┘   └────────────┘   └────────────┘
                                                      (conditional)
     │                 │                 │                 │                 │                 │                 │
     │                 │                 │                 │                 │                 │                 │
     ▼                 ▼                 ▼                 ▼                 ▼                 ▼                 ▼

GitHub Actions   Pipeline:        Pipeline:        Release +         Release:         Argo CD CMP:    Kubernetes +
submits run     Discovery         Hot Worker       Artifacts:        Updates          Extracts        Crossplane:
via CLI         worker builds     Pools execute    Creates           ReleasePointer   resources       Applies
                execution DAG     phases per       artifacts &       files in Git     from Release    resources
                                  project config   packages KRDs                      OCI image       to cluster
                                                   as OCI image
```

1. **Trigger**: GitHub Actions workflow submits pipeline run via CLI
2. **Discovery**: Pipeline component discovers projects and builds execution DAG
3. **Phase Execution**: Hot worker pools execute phases based on project configuration
4. **Release Creation** (conditional): When triggered by event configuration, creates artifacts and packages Kubernetes resource definitions as OCI image
5. **GitOps Update**: Platform updates pointer files in Git repository
6. **Resource Extraction**: Argo CD Custom Management Plugin extracts Kubernetes resources from Release OCI image
7. **Reconciliation**: Resources are applied to cluster - Crossplane handles XRs while Kubernetes handles standard resources

### Technology Stack & Decisions

#### Core Technologies

1. **CUE for all configuration** - Type-safe, composable configuration with built-in validation and Kubernetes resource generation

2. **Earthly for builds** - Reproducible builds with familiar Dockerfile syntax and native monorepo support

3. **GitOps with Argo CD** - Declarative deployments with automatic drift detection and Git as audit log

4. **Crossplane for resource composition** - CNCF-graduated solution for abstracting Kubernetes complexity through Composite Resources

   Platform provides curated XRD catalog with smart defaults. See [Infrastructure Abstractions: XRD Catalog](08-infrastructure-abstractions.md#xrd-catalog) for complete specifications.

5. **GitHub Actions as trigger** - Native integration with existing SCM and zero infrastructure overhead

6. **Argo Workflows for orchestration** - Kubernetes-native workflow orchestration with built-in UI and retry logic

7. **PostgreSQL for persistence** - Reliable, well-understood relational database for audit trails and state management

#### Architectural Patterns

8. **API-first architecture** - Provides security boundary and enables persistence for querying

9. **Two-tier OCI strategy** - Artifacts and releases only; no separate deployment images needed

10. **Crossplane composition functions** - Python functions provide flexible resource composition while maintaining consistency

11. **Kubernetes operators** - Self-healing, declarative infrastructure using platform-native patterns

12. **Environment-agnostic releases** - Same release (containing KRDs) deploys to any environment via two-tier EnvironmentConfig system for XRs. See [Infrastructure Abstractions: Environment Configuration Model](08-infrastructure-abstractions.md#environment-configuration-model).

13. **Hot worker pools** - Persistent pods with cached repositories and warm connections to eliminate cold start penalties

14. **Dual persistence model** - Argo Workflows for volatile execution state, database for permanent audit trail

## Shared Concepts

### OCI Image Formats

The platform uses two distinct types of OCI images for tracking and deployment. This section defines the authoritative standards for both formats.

#### Artifact OCI Tracking Images

Artifact OCI tracking images serve as immutable provenance records for individual artifacts.

**Purpose**: Long-term artifact tracking and security attestation
**Created**: During release phase after artifact publication
**Owner**: Artifacts Component (see [Build & Distribution](04-build-distribution.md))

**Format**:
```
[tracking-registry]/[repository]/[project]/[artifact]:sha-[commit_sha]
```

**Rationale**: Project names are only unique within a repository, so the repository prefix is required to ensure global uniqueness.

**Contains**:
- Artifact metadata (name, type, project, commit)
- Published locations (all registries where artifact exists)
- Build information (pipeline run, producer details, build timestamp)
- Provenance data (source materials, builder identity)
- SBOM layer (software bill of materials)

**Usage**:
- Security auditing and vulnerability tracking
- Supply chain verification
- Artifact metadata lookup
- Tracing artifacts back to source commits

**Structure**:
```yaml
# OCI Manifest
layers:
  - mediaType: application/vnd.forge.artifact.metadata.v1+json
    annotations:
      org.catalyst-forge.layer.type: metadata
  - mediaType: application/vnd.syft+json
    annotations:
      org.catalyst-forge.layer.type: sbom
  - mediaType: application/vnd.in-toto+json
    annotations:
      org.catalyst-forge.layer.type: attestation
```

#### Release OCI Images

Release OCI images package deployable snapshots with Kubernetes resource definitions.

**Purpose**: GitOps deployment containing KRDs and artifact references
**Created**: During release phase after all artifacts are built
**Owner**: Release & Deployment Component (see [Release & Deployment](05-release-deployment.md))

**Format**:
```
[release-registry]/[repository]/[project]:[commit-sha]
```

**Note**: The canonical identifier is the commit SHA. Aliases (numeric, tag, branch) can also be used to reference releases, but the commit SHA is the authoritative identifier.

**Contains**:
- Release metadata (release number, aliases, trigger information, source details)
- Artifact references (including tracking OCI URIs for full provenance chain)
- Rendered Kubernetes YAML (ready to apply to clusters)

**Usage**:
- Argo CD extracts and applies resources to clusters
- Provides complete deployment snapshot
- Enables reproducible deployments across environments
- Maintains audit trail from source to deployment

**Structure**:
```yaml
# OCI Manifest
layers:
  - mediaType: application/vnd.forge.release.metadata.v1+json
    annotations:
      org.catalyst-forge.layer.type: metadata
  - mediaType: application/vnd.forge.release.resources.v1+tar
    annotations:
      org.catalyst-forge.layer.type: resources
```

#### Relationship Between OCI Image Types

Release OCI images reference Artifact OCI tracking images in their metadata layer, creating a complete audit trail:

```
┌──────────────────────────────────────────────────────────────────────────────┐
│                         TIER 1: ARTIFACT OCI IMAGES                          │
│                              (Build-time)                                    │
├──────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  Purpose: Immutable build artifacts with full provenance                     │
│  Created: During pipeline phase execution                                    │
│  Format:  [tracking-registry]/[repository]/[project]/[artifact]:sha-[sha]    │
│                                                                              │
│  ┌────────────────────────────────────────────────────────────────┐          │
│  │ Contents:                                                      │          │
│  │  • Compiled binaries, container images, or packaged archives   │          │
│  │  • Metadata: source commit SHA, build timestamp, SBOM          │          │
│  │  • Provenance: builder identity, source materials              │          │
│  └────────────────────────────────────────────────────────────────┘          │
│                                                                              │
│  Storage: OCI registry with commit SHA tagging                               │
│                                                                              │
└───────────────────────────────────┬──────────────────────────────────────────┘
                                    │
                                    │ referenced by
                                    ▼
┌──────────────────────────────────────────────────────────────────────────────┐
│                         TIER 2: RELEASE OCI IMAGES                           │
│                             (Deploy-time)                                    │
├──────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  Purpose: Deployable package linking resources to artifacts                  │
│  Created: During release creation process                                    │
│  Format:  [release-registry]/[repository]/[project]:[commit-sha]             │
│                                                                              │
│  ┌────────────────────────────────────────────────────────────────┐          │
│  │ Contents:                                                      │          │
│  │  • Kubernetes resource definitions (YAML)                      │          │
│  │  • Crossplane compositions and XRs                             │          │
│  │  • References to Artifact OCI images via @artifact() in CUE    │          │
│  │  • Metadata: release version, artifact refs, deployment target │          │
│  └────────────────────────────────────────────────────────────────┘          │
│                                                                              │
│  Storage: OCI registry with release identifiers (SHA, tags, aliases)         │
│                                                                              │
└──────────────────────────────────────────────────────────────────────────────┘

Provenance Chain:

  1. Source Commit                  repository/project @ commit-sha
          │
          ▼
  2. Pipeline Builds                Creates Artifact OCI image(s)
          │                         [tracking-registry]/[repository]/[project]/[artifact]:sha-[sha]
          ▼
  3. Release Trigger Evaluates      Initiates release creation
          │
          ▼
  4. CUE Configs Render             @artifact() functions resolve to Artifact OCI refs
          │                         Creates Release OCI image
          ▼                         [release-registry]/[repository]/[project]:[commit-sha]
  5. Release OCI Deployed
          │
          ▼
  6. Argo CD Extracts & Applies     Resources reference Artifact OCI images for containers/binaries
          │
          ▼
  7. Deployment Complete            Full provenance chain: commit → artifact → release → deployment

Key Relationships:
• Artifact OCI images are immutable and versioned by commit SHA
• Release OCI images contain references (not copies) of Artifact OCI images
• Multiple Release OCI images can reference the same Artifact OCI images
• Deployment progression (dev/staging/prod) uses same Release OCI, different pointers
```

**Canonical Triplet**: All identifiers follow `repository/project/commit-sha` pattern
- **Artifact OCI**: Extends with `/artifact` suffix for artifact-specific tracking
- **Release OCI**: Uses triplet directly as the base identifier

This two-tier approach provides:
- **Separation of concerns**: Artifacts managed independently from releases
- **Reusability**: Same artifact can be referenced by multiple releases
- **Complete provenance**: Trace deployment back through release to artifacts to source
- **Audit trail**: Immutable record of what was built, published, and deployed
- **Global uniqueness**: Repository prefix ensures project names don't collide across repositories

### GitOps Repository Structure

The platform maintains a single GitOps repository with hierarchical organization for deployment management.

**Repository Pattern**:
```
gitops-repo/
├── environments/
│   ├── dev/
│   │   └── {repository}/
│   │       └── {project}/
│   │           └── release.yaml
│   ├── staging/
│   │   └── {repository}/
│   │       └── {project}/
│   │           └── release.yaml
│   └── production/
│       └── {repository}/
│           └── {project}/
│               └── release.yaml
```

**Pointer File Format**:

Pointer files use a Kubernetes-native format for consistency:

```yaml
apiVersion: forge.projectcatalyst.io/v1
kind: ReleasePointer
metadata:
  name: {project-name}
  environment: {environment}
spec:
  release: {commit-sha}  # Canonical identifier for Argo CD
  aliases:
    numeric: 42
    tag: v1.2.3
    branch: main-42
  deployedAt: "2024-01-01T00:00:00Z"
  deployedBy: "alice@company.com"
  approvedBy: "bob@company.com"
  approvedAt: "2023-12-31T22:00:00Z"
```

**Integration with Argo CD**:

Argo CD uses a Custom Management Plugin (CMP) to process Release pointers:

1. Plugin reads ReleasePointer from Git
2. Extracts `spec.release` commit SHA
3. Fetches Release OCI image using the SHA
4. Extracts resources layer (tar archive of YAML files)
5. Returns resources to Argo CD for application

**Design Rationale**:
- **Single repository**: Simplifies access control and audit logging
- **Environment-first hierarchy**: Clear separation of deployment targets
- **Pointer indirection**: Decouples Git commits from release versions
- **Kubernetes-native format**: Consistent with other platform resources
- **Canonical SHA**: Ensures deployment reproducibility

**GitOps Flow Diagram**:

```
┌─────────────────┐         ┌──────────────────┐         ┌─────────────────┐
│  GitOps Repo    │         │  Argo CD App     │         │ Custom Mgmt     │
│                 │         │                  │         │ Plugin (CMP)    │
│  environments/  │         │  Monitors repo   │         │                 │
│    dev/         │         │  for changes     │         │  Processes      │
│      repo/      │         │                  │         │  ReleasePointer │
│        project/ │         │                  │         │                 │
│    release.yaml │         │                  │         │                 │
└────────┬────────┘         └────────┬─────────┘         └────────┬────────┘
         │                           │                            │
         │  1. Git commit            │                            │
         │  (pointer update)         │                            │
         │                           │                            │
         │  ────────────────────────▶│                            │
         │                           │  2. Detects change         │
         │                           │     Triggers CMP           │
         │                           │  ─────────────────────────▶│
         │                           │                            │
         │                           │  3. Reads ReleasePointer   │
         │  ◀─────────────────────────────────────────────────────│
         │     (fetch from Git)      │                            │
         │                           │                            │
                                     │                            │  4. Extracts
                                     │                            │     commit SHA
                                     │                            │
                                     │                            ▼
                              ┌──────────────────────────────────────────────┐
                              │           OCI Registry                       │
                              │                                              │
                              │  Release OCI Images:                         │
                              │  [registry]/[repo]/[project]:[commit-sha]    │
                              └──────────────────┬───────────────────────────┘
                                                 │
                                                 │  5. Fetch Release OCI
                                                 │     by commit SHA
         ┌───────────────────────────────────────┘
         │  6. Returns image layers
         │     (YAML manifests)
         ▼
┌─────────────────┐
│ CMP Processing  │
│                 │
│ • Extracts YAML │
│ • Validates     │
│ • Returns to    │
│   Argo CD       │
└────────┬────────┘
         │
         │  7. Extracted Kubernetes YAML
         │
         ▼
┌─────────────────┐         ┌──────────────────┐         ┌─────────────────┐
│  Argo CD Core   │         │  Kubernetes      │         │  Crossplane     │
│                 │         │  Cluster         │         │                 │
│  Compares       │         │                  │         │  Handles XRs    │
│  desired vs     │  ──────▶│  Applies         │  ──────▶│  (infrastructure│
│  actual state   │         │  resources       │         │   provisioning) │
└─────────────────┘         └──────────────────┘         └─────────────────┘

Flow Summary:
  Pointer update → Argo CD detects → CMP reads pointer → Fetches OCI image →
  Extracts YAML → Argo applies → Kubernetes reconciles → Crossplane handles XRs

Key Properties:
  • Immutable reference ensures consistency (commit SHA)
  • CMP handles OCI extraction logic (standard GitOps sync after extraction)
  • Pointer update triggers reconciliation (Git as source of truth)
```

### Secret Management Patterns

The platform implements a universal secret reference pattern that prevents raw secrets from traversing component boundaries.

#### Universal Secret Reference Format

The platform uses a **ports and adapters** pattern for secret management. The `provider` field determines which secret provider implementation to use, and each provider defines its own configuration schema.

**Core Structure**:

```cue
credentials: {
    provider: string  // Selects the secret provider implementation
    // ... provider-specific fields determined by the provider's schema
}
```

**How It Works**:

1. Platform maintains a **registry of secret providers**
2. Each provider exposes its own configuration schema
3. The `provider` field selects which provider to use
4. Remaining fields are passed to that provider for interpretation
5. Provider returns the resolved secret value

**Example Provider Schemas**:

```cue
// AWS Secrets Manager Provider (Initial Release - Supported)
credentials: {
    provider: "aws"
    secretName: string      // AWS secret name
    region?: string         // Optional AWS region override
}

// Future Provider Examples (Architecture Support - Not Yet Implemented)

// HashiCorp Vault Provider (Potential)
credentials: {
    provider: "vault"
    path: string           // Vault secret path
    key?: string           // Optional key within secret
    version?: string       // Optional secret version
}

// GitHub OIDC Provider (Potential)
credentials: {
    provider: "github-oidc"
    repository: string     // Repository scope
}

// Kubernetes Secret Provider (Potential)
credentials: {
    provider: "kubernetes"
    namespace: string
    secretName: string
    key: string
}
```

**Design Rationale**:

This ports and adapters approach allows:
- **Easy integration** of new secret providers without platform changes
- **Provider-specific optimizations** (e.g., caching, rotation)
- **No forced universal schema** that doesn't fit all providers
- **Clear extension point** for custom secret sources

#### Resolution Pattern

**Principle**: Secrets are never resolved or passed as plain text between components. Each component resolves secrets at the point of use.

**Implementation**:
1. Configuration contains only secret **references**, never values
2. References are passed through component boundaries safely
3. Each component resolves references when needed
4. Secrets exist in memory only during active use
5. No secrets are persisted in databases or logs

**Example Flow**:

```
Project Config (secret reference)
    ↓
Pipeline discovers project (reference passes through)
    ↓
Publisher receives reference (not secret value)
    ↓
Publisher resolves secret at authentication time
    ↓
Secret used for registry authentication
    ↓
Secret discarded from memory
```

#### Secret Providers

The platform initially ships with **AWS Secrets Manager** as the sole secret provider. The architecture uses a provider registry pattern to support future extension.

**AWS Secrets Manager Provider** (Initial Release):
- Authentication: IRSA (IAM Roles for Service Accounts)
- Resolution: AWS SDK
- Features: Automatic rotation, audit logging via CloudWatch
- Schema: `{provider: "aws", secretName: string, region?: string}`

**Example Usage**:
```cue
credentials: {
    provider: "aws"
    secretName: "docker/ecr-credentials"
    region: "us-east-1"  // Optional, defaults to worker region
}
```

**Provider Interface** (Extension Point):

The architecture supports future providers through a common interface:
```go
type SecretProvider interface {
    // Name returns the provider identifier (e.g., "aws")
    Name() string

    // Schema returns the CUE schema for this provider's configuration
    Schema() string

    // Resolve retrieves the secret value given provider-specific config
    Resolve(ctx context.Context, config map[string]any) (string, error)
}
```

**Future Provider Support**:

While not included in the initial release, the architecture enables future integration of additional providers such as:
- HashiCorp Vault (dynamic secrets, versioning)
- Kubernetes Secrets (namespace-scoped)
- GitHub OIDC tokens (repository-scoped)
- Custom providers (organization-specific secret stores)

New providers can be added by implementing the `SecretProvider` interface and registering with the platform without requiring changes to existing configurations or components.

#### CI Secret Management

CI workers access secrets via IRSA policies:

```yaml
# Pipeline worker IRSA policy
Statement:
  - Effect: Allow
    Action:
      - secretsmanager:GetSecretValue
    Resource:
      - arn:aws:secretsmanager:*:*:secret:ci/*
```

**Security Properties**:
- Secrets never persisted in database
- GitHub OIDC tokens validated with configurable expiry
- Workers access secrets via IRSA policies (no long-lived credentials)
- Earthly handles secure transmission to build environment
- Secret references auditable without exposing values

#### Integration with External Secrets

For deployed applications, the platform integrates with External Secrets Operator:

- Crossplane XRs can declare external secret requirements
- External Secrets Operator syncs secrets into cluster
- Applications consume secrets as Kubernetes Secret resources
- Rotation handled automatically by External Secrets

### Authentication & Authorization Model

The platform implements layered authentication with clear boundaries for different types of access. **All authentication flows are managed through Keycloak**, which handles both user and machine authentication. External identity providers (including GitHub OIDC) exchange tokens for Keycloak identities.

#### Authentication Boundaries

**External API Calls (GitHub Actions)**:
- **Method**: GitHub OIDC tokens exchanged for Keycloak identity
- **Flow**: GitHub OIDC token → Keycloak token exchange → Platform authentication
- **Validation**: Repository membership validated through Keycloak
- **Scope**: Pipeline creation and triggering
- **Expiry**: Configurable token lifetime (default: 60 minutes)

**CLI Users (Developers)**:
- **Method**: Authentication via Keycloak (supports SSO, OAuth, etc.)
- **Validation**: Keycloak identity validation
- **Scope**: Read operations, manual deployments, approvals
- **Permissions**: Role-based access control (RBAC) managed in Keycloak

**Worker Pools (Internal)**:
- **Method**: IAM Roles via IRSA (IAM Roles for Service Accounts)
- **Validation**: Kubernetes service account tokens
- **Scope**: AWS service access (S3, DynamoDB, SQS, Secrets Manager)
- **Permissions**: Least-privilege IAM policies

**Argo Server**:
- **Method**: Bearer tokens for workflow submission
- **Validation**: Kubernetes service account authentication
- **Scope**: Workflow creation and management
- **Permissions**: Namespace-scoped workflow operations

**Internal Services**:
- **Method**: Keycloak service account tokens (machine-to-machine auth)
- **Validation**: Keycloak validates service identity
- **Scope**: Inter-component API calls
- **Permissions**: Service-specific RBAC policies managed in Keycloak

#### Authorization Model

**Repository Registration**:
- Repositories must be registered before pipeline creation
- Registration creates access control entries
- Per-repository quotas and limits enforced

**Environment-Specific Permissions**:
- Development: Broad access for developers
- Staging: Restricted deployment permissions
- Production: Approval-only deployments with named approvers

**Role-Based Access Control**:
```
Roles:
  - developer: Read pipeline status, manual deployments to dev
  - approver: Approve releases for promotion
  - operator: Platform administration
  - service-account: Automated system operations
```

#### Security Boundaries

**Per-Run Job Isolation**:
- Each pipeline run has isolated SQS jobs
- DynamoDB tracks status per job with TTL
- S3 results namespaced by run ID
- No cross-run data access

**Worker Pool Security**:
- Workers use IRSA for AWS service access (no long-lived credentials)
- Dispatcher pods have minimal permissions
- Network policies restrict inter-pod communication
- Resource quotas prevent resource exhaustion

**Secret Access Control**:
- Secrets scoped by team/repository
- IRSA policies enforce least-privilege access
- Audit logging for all secret access
- Secrets never persisted in database or logs

## Architecture

The core architecture section is distributed across component-specific documents. For detailed architectural information, see:

- [Execution & Orchestration](03-execution-orchestration.md) - Pipeline execution architecture
- [Build & Distribution](04-build-distribution.md) - Artifact management architecture
- [Release & Deployment](05-release-deployment.md) - Release and deployment architecture
- [Integration Contracts](07-integration-contracts.md) - Component interaction patterns

## Configuration

Configuration is managed through CUE at all levels of the platform. For detailed configuration specifications, see:

- [Configuration & Discovery](02-configuration-discovery.md) - Complete configuration reference
- [Build & Distribution](04-build-distribution.md) - Artifact and publisher configuration
- [Release & Deployment](05-release-deployment.md) - Release trigger configuration

## Operations

Operational aspects are covered in component-specific documents:

- [Execution & Orchestration: Performance & Scaling](03-execution-orchestration.md#performance--scaling)
- [Execution & Orchestration: Monitoring & Observability](03-execution-orchestration.md#monitoring--observability)

## Integration Points

This document defines shared concepts that are consumed by all components. For specific integration patterns, see:

- [Integration Contracts](07-integration-contracts.md) - Complete integration specifications
- [Domain Model & API Reference](06-domain-model-api-reference.md) - Entities and APIs

**How Components Use Shared Concepts**:

- **OCI Image Formats**: Consumed by Artifacts and Release components for creating tracking and release images
- **GitOps Repository Structure**: Used by Release component for deployment management
- **Secret Management**: Used by all components for credential management
- **Authentication Model**: Enforced across all API boundaries

**How Components Use Infrastructure Abstractions**:
- Projects declare XRs in deployment configuration using familiar abstractions (secrets, configs, DNS)
- Release component packages XRs alongside raw Kubernetes resources in Release OCI images
- Crossplane composition functions apply environment-specific configuration during deployment
- See [Infrastructure Abstractions](08-infrastructure-abstractions.md) for complete XRD catalog and patterns

## Reference

### Key Terms

**Repository**: A GitHub repository containing one or more projects
**Project**: A deliverable unit marked by a `.forge` folder (unique within repository, not globally)
**Canonical Triplet**: The identifier pattern `repository/project/commit-sha` used throughout the platform for global uniqueness
**Pipeline Run**: A single execution of the CI/CD pipeline for a commit
**Phase**: A named stage in the pipeline (e.g., test, build)
**Step**: An individual task within a phase
**Artifact**: A build output (container, binary, or archive)
**Release**: An immutable snapshot of a project at a specific commit
**Deployment**: The application of a release to an environment
**Environment**: A deployment target (dev, staging, production)
**KRD**: Kubernetes Resource Definition (YAML manifest)
**XR**: Crossplane Composite Resource (platform abstraction)

**Identifier Patterns**:
- **Artifact OCI**: `[tracking-registry]/[repository]/[project]/[artifact]:sha-[commit_sha]`
- **Release OCI**: `[release-registry]/[repository]/[project]:[commit-sha]`
- **GitOps Path**: `environments/[environment]/[repository]/[project]/release.yaml`

## Examples

Examples are provided in component-specific documents. See:

- [Configuration & Discovery: Complete Configuration Examples](02-configuration-discovery.md#complete-configuration-examples)
- [Build & Distribution: Examples](04-build-distribution.md#examples)
- [Release & Deployment: Examples](05-release-deployment.md#examples)

## Constraints & Limitations

### Platform Constraints

- **Single AWS Region**: Platform operates in one region per installation
- **Kubernetes Only**: No support for other orchestrators
- **Earthly Required**: Build steps must use Earthly
- **Monorepo Focus**: Optimized for monorepo workflows
- **GitHub Required**: No other SCM providers supported

### Scale Limits

- **Projects per Repository**: Recommended maximum 100
- **Concurrent Pipelines**: Recommended maximum 1000
- **Pipeline Duration**: Hard limit 24 hours
- **Discovery Output Size**: Maximum 1MB
- **Worker Pools**: Maximum 50 parallel workers per pool

### Technical Constraints

- **No Cross-Repository Dependencies**: Projects cannot depend on other repositories
- **Sequential Phase Execution**: Phases run sequentially, not in parallel
- **No Multi-Project Releases**: Each project releases independently
- **Environment Policies Global**: Cannot customize per-project

## Future Enhancements

### Planned Improvements

**Multi-Region Support**:
- Worker pools in multiple regions for disaster recovery
- Region-aware artifact publishing
- Cross-region release replication

**Enhanced Discovery**:
- Incremental discovery for monorepos
- Cached discovery results
- Predictive project loading

**Extended Build Support**:
- Alternative execution engines beyond Earthly
- GPU-enabled workers for ML workloads
- Custom worker pool configurations per project

**Deployment Enhancements**:
- Multi-project release groups
- Progressive deployment strategies (canary, blue-green)
- Dynamic environment creation for feature branches

**Performance Optimizations**:
- Distributed cache using Redis/Elasticache
- P2P cache sharing between workers
- Cost optimization through spot instances

---

**Component Owner**: Platform Core
**Last Updated**: 2025-10-02
