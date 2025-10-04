# Infrastructure Abstractions Specification

## Executive Summary

This document defines the Crossplane-based infrastructure abstractions for the Catalyst Forge platform. It specifies the initial XRD catalog, environment configuration patterns, and deployment resolution processes that allow developers to deploy applications without Kubernetes expertise.

The platform provides a curated set of Crossplane Composite Resources (XRs) that abstract common infrastructure patterns while maintaining flexibility for diverse workloads. Environment-specific configuration is managed through GitOps using Crossplane's native EnvironmentConfig resources, providing a two-tier system of cluster-wide defaults and per-project overrides.

## Core Concepts

### Design Philosophy

**Familiar Abstractions Over Kubernetes Primitives** - Developers work with concepts like secrets, configs, DNS, and volumes rather than Services, Ingresses, or PersistentVolumeClaims.

**Composable Building Blocks** - Focused XRDs that handle one concern well, avoiding monolithic complexity.

**Progressive Disclosure with Smart Defaults** - Minimal configuration for standard cases with detailed customization available when needed. Production environments automatically receive appropriate resource limits and high availability settings.

### Technology Foundation

The platform operates on AWS EKS with these key operators:
- External Secrets Operator (AWS Secrets Manager sync)
- External DNS (Route53 management)
- Envoy Gateway (Gateway API ingress)
- EBS CSI Driver (persistent volumes)
- IRSA (pod-level AWS authentication)

### Reference Resolution Model

The platform provides a universal reference pattern that works consistently across environment variables and volume mounts. This model addresses a fundamental question in infrastructure abstraction: when should values be inlined versus referenced?

#### Design Principle

The reference system follows a single rule: **`outputs` returns literal values, everything else returns references**.

This design makes the security model explicit in the syntax itself. Literal values (via `outputs/`) are treated as public and safe to log. Referenced values (via `connections/`, `secrets/`, `configs/`) are treated as sensitive and must remain in Kubernetes Secret or ConfigMap resources.

#### Path Structure Taxonomy

The reference system uses two distinct structural patterns that reflect the underlying resource model:

**Two-part paths** (`outputs/`, `connections/`):
- Directly reference any XR's published outputs
- Structure: `<prefix>/<xr_name>/<key>`
- Examples: `outputs/assets/arn`, `connections/assets/accessKeyId`

**Three-part paths** (`secrets/`, `configs/`):
- Reference items within Secrets or ConfigMaps XRs
- Structure: `<prefix>/<xr_name>/<item_name>/[key]`
- Examples: `secrets/myapp-secrets/database/password`, `configs/app-settings/features/enabled`

The structural difference reflects intent: two-part paths access XR outputs directly, while three-part paths navigate into collections managed by specialized XRs.

#### Reference Type Resolution

The first path segment determines resolution behavior:

| Path Type      | Structure             | Example                                   | Returns         | Description                                     |
| -------------- | --------------------- | ----------------------------------------- | --------------- | ----------------------------------------------- |
| `outputs/`     | `<xr>/<key>`          | `outputs/assets/arn`                      | Literal value   | Inlined value from any XR's `status.outputs.*`  |
| `connections/` | `<xr>/<key>`          | `connections/assets/accessKeyId`          | secretKeyRef    | Reference to any XR's connection-details Secret |
| `secrets/`     | `<xr>/<secret>/[key]` | `secrets/myapp-secrets/database/password` | secretKeyRef    | Reference to secret from Secrets XRD            |
| `configs/`     | `<xr>/<config>/[key]` | `configs/app-settings/features/enabled`   | configMapKeyRef | Reference to ConfigMap from ConfigMaps XRD      |

#### Cross-Namespace References

References can span namespace boundaries using explicit notation:

**Cross-project references**: `<repository>-<project>::outputs/<xr>/<key>`
- Example: `backend-repo-storage::outputs/bucket/arn`
- Limited to `outputs/` only

**Platform references**: `platform::outputs/<xr>/<key>`
- Example: `platform::outputs/shared-config/api-endpoint`
- Limited to `outputs/` only

Platform resources are infrastructure components managed by the platform team. Cross-namespace references are restricted to `outputs/` because accessing secrets and configs across namespace boundaries requires additional RBAC configuration not provided by default.

#### Producer/Consumer Semantics

XRs act as both producers and consumers in a dependency graph:

**Public Outputs** (`outputs/<xr_name>/<key>`):
- Producer publishes via `status.outputs.<key>`
- Values are inlined directly into consuming resources
- Treated as non-secret and safe to log
- Supports cross-namespace references

**Connection Details** (`connections/<xr_name>/<key>`):
- Producer publishes via connection-details Secret
- Values are referenced via `secretKeyRef`, never inlined
- Treated as sensitive
- Same-namespace only

**Secrets and ConfigMaps** (`secrets/...`, `configs/...`):
- Managed by specialized Secrets and ConfigMaps XRs
- Same-namespace only
- Cross-namespace forms are invalid

## Architecture

### XRD Catalog

The platform provides separate XRDs for each workload type rather than a single polymorphic resource. This design choice provides cleaner schemas, better validation, and avoids discriminated union complexity.

| XRD            | Purpose                           | Key Features                                                                           | Example Use Case                               |
| -------------- | --------------------------------- | -------------------------------------------------------------------------------------- | ---------------------------------------------- |
| **Deployment** | Stateless services                | Horizontal scaling, rolling updates, health checks, resource limits, output publishing | Web servers, APIs, stateless workers           |
| **Stateful**   | Services with persistent identity | Ordered deployment, stable storage, persistent identifiers, output publishing          | Databases, message queues, distributed systems |
| **Job**        | One-time tasks                    | Completion tracking, retry policies, TTL cleanup, output publishing                    | Migrations, batch processing, initialization   |
| **CronJob**    | Scheduled tasks                   | Cron syntax, concurrency control, history limits, output publishing                    | Reports, cleanup, recurring operations         |
| **Network**    | Service exposure & DNS            | HTTP routing, DNS records, TLS termination, rate limiting, endpoint output publishing  | External access, service discovery             |
| **Secrets**    | External secrets sync             | AWS Secrets Manager integration, automatic rotation, environment prefixing             | Database credentials, API keys                 |
| **ConfigMaps** | Application configuration         | Key-value pairs, file-based configs, nested structures                                 | Application settings, feature flags            |
| **Storage**    | Persistent volumes                | EBS provisioning, backup scheduling, volume expansion, output publishing               | Database storage, file uploads                 |

All XRDs support output publishing through `status.outputs.*` for public values and connection-details Secrets for sensitive values. This enables XRs to act as both producers and consumers in a dependency graph.

#### AWS Provider Managed Resources

In addition to the core platform XRDs, the platform may provide additional XRDs that wrap AWS provider managed resources (e.g., S3 buckets, SQS queues, RDS databases). Teams create XRs from these XRDs to provision AWS infrastructure. These XRDs follow the same outputs pattern, publishing public values via `status.outputs.*` and sensitive values via connection-details Secrets.

### Environment Configuration Model

The platform uses a two-tier configuration system with Crossplane EnvironmentConfigs. This design provides clear hierarchy with predictable precedence while remaining GitOps-native.

#### Cluster-Wide Configuration
Stored in `<env>/env.yml` in the GitOps repository. Provides:
- Environment identification (name, domain, region)
- Platform conventions (secret prefixes, storage classes)
- Resource defaults per workload type
- Environment-wide feature flags

#### Per-Project Configuration
Stored in `<env>/<repo>-<project>/env.yml` in the GitOps repository. Provides:
- Workload-specific tuning (replicas, resources)
- Schedule modifications for CronJobs
- Feature toggles per project
- Temporary overrides (e.g., suspend jobs during incidents)

#### Resolution Precedence

Values are resolved with clear precedence to ensure predictable behavior:

1. **Per-project override** - Specific overrides for named XRD instances
2. **XRD spec value** - Developer-specified configuration
3. **Cluster-wide default** - Environment-level defaults
4. **Composition default** - Hardcoded fallbacks

Per-project EnvironmentConfigs override XRD inputs rather than patching final manifests. This maintains type safety and ensures overrides work across implementation changes.

### Namespace Strategy

Each project automatically receives a namespace: `<repository>-<project>`

Examples:
- Repository `backend-repo`, Project `api` → Namespace `backend-repo-api`
- Repository `monorepo`, Project `auth-service` → Namespace `monorepo-auth-service`

This maintains consistency with the platform's canonical triplet pattern while ensuring global uniqueness without API lookups.

## Configuration Example

A minimal example demonstrating XRD composition, reference patterns, and producer/consumer relationships:

```yaml
deployment:
  resources:
    # Producer: Infrastructure resource publishing outputs
    - apiVersion: storage.example.org/v1alpha1
      kind: Bucket
      metadata:
        name: assets
      spec: {}

    # Secrets from AWS Secrets Manager
    - apiVersion: forge.io/v1
      kind: Secrets
      metadata:
        name: myapp-secrets
      spec:
        secrets:
          - name: database
            path: myapp/database
          - name: tls-cert
            path: certificates/wildcard
            global: true

    # Consumer: Application deployment using references
    - apiVersion: forge.io/v1
      kind: Deployment
      metadata:
        name: myapp-web
      spec:
        containers:
          - name: web
            image: @artifact("web.uri")
            ports:
              - containerPort: 8080
            env:
              DB_PASSWORD:
                source: secrets/myapp-secrets/database/password
              BUCKET_ARN:
                source: outputs/assets/arn
              S3_ACCESS_KEY_ID:
                source: connections/assets/accessKeyId
              S3_SECRET_ACCESS_KEY:
                source: connections/assets/secretAccessKey
              SHARED_BUCKET:
                source: backend-repo-storage::outputs/shared-assets/arn
              AUTH_ENDPOINT:
                source: backend-repo-auth::outputs/service-endpoint/url
              PLATFORM_API:
                source: platform::outputs/shared-config/api-endpoint
            mounts:
              - mountPath: /etc/app
                source: configs/app-settings/main
        replicas: 3

    # Service networking
    - apiVersion: forge.io/v1
      kind: Network
      metadata:
        name: myapp-network
      spec:
        workloadRef: myapp-web
        subdomain: myapp
        port: 8080
```

## Operations

### Secret Path Management

Secrets XRDs contain no environment references. A `global` boolean distinguishes between environment-specific (default) and global secrets, allowing composition functions to handle prefixing automatically.

The platform automatically applies environment prefixes to AWS Secrets Manager paths:

| Secret Type          | Path in XRD                                   | Actual AWS Path                |
| -------------------- | --------------------------------------------- | ------------------------------ |
| Environment-specific | `myapp/database`                              | `{env}/myapp/database`         |
| Global               | `certificates/wildcard` (with `global: true`) | `global/certificates/wildcard` |

### Deployment Resolution Process

1. **CUE Evaluation** - Project configurations render with `@artifact()` references resolved
2. **XR Creation** - Argo CD applies XRs to the cluster
3. **EnvironmentConfig Loading** - Compositions load cluster-wide and per-project configs
4. **Value Resolution** - Applied using precedence rules
5. **Reference Resolution** - Compositions resolve `outputs/<xr>/<key>` and `connections/<xr>/<key>` patterns
6. **Resource Generation** - Compositions create Kubernetes resources
7. **Operator Reconciliation** - External Secrets, External DNS, and other operators reconcile

### Output Freshness & Updates

Consumer XRs re-resolve producer output references on their **next reconcile** (triggered by periodic polling or when the consumer XR changes). This means:

- If a producer's `status.outputs.*` value changes, consumers pick up the new value on their next reconcile
- Connection-details Secret updates are handled by Kubernetes' native `secretKeyRef` mechanism
- There is no real-time propagation of output changes
- Manual reconcile can be triggered by updating a consumer XR's metadata (e.g., adding/updating an annotation)

### GitOps Structure

EnvironmentConfigs are managed alongside release pointers:

```
gitops-repo/
├── production/
│   ├── env.yml                    # Cluster-wide config
│   └── backend-repo-api/
│       ├── release.yaml           # Release pointer
│       └── env.yml                # Project overrides
└── dev/
    ├── env.yml
    └── backend-repo-api/
        └── release.yaml
```

## Integration Points

### Platform Components
- **Configuration & Discovery**: Projects declare XRDs in deployment configuration
- **Release & Deployment**: XRDs packaged in Release OCI images
- **Artifacts**: `@artifact()` references resolved before XRD creation

### Kubernetes Operators
- **External Secrets Operator**: Secrets XRD generates ExternalSecret resources
- **External DNS**: Network XRD generates DNSEndpoint resources
- **Envoy Gateway**: Network XRD generates Gateway API resources
- **Crossplane**: Handles XR to resource transformation

### Composition Functions
All XRDs use Python-based composition functions that:
- Load and merge EnvironmentConfigs
- Apply resolution precedence
- Transform reference patterns to Kubernetes resources
- Handle environment-specific logic (e.g., PodDisruptionBudgets in production)
- **Publish outputs** via two mechanisms:
  - Set `status.outputs.*` on the composite for public values
  - Auto-configure `spec.writeConnectionSecretToRef` with deterministic naming (`<xr-name>-conn`)
- **Resolve consumer references**:
  - For `outputs/<xr>/<key>`: request producer XR as required resource, read `status.outputs.<key>`, inline value
  - For `connections/<xr>/<key>`: ensure same namespace, request producer XR, read its connection-details Secret name, set `secretKeyRef` pointing to that Secret
  - For `secrets/<xr>/<secret>/[key]`: set `secretKeyRef` to the appropriate key in Secret from Secrets XRD
  - For `configs/<xr>/<config>/[key]`: set `configMapKeyRef` to the appropriate key in ConfigMap from ConfigMaps XRD

## Security Model

### RBAC Requirements

Composition functions require specific permissions to resolve cross-XR references:

- **Cluster-wide read** (`get/list/watch`) on all XRD types to read `status.outputs.*` from producer XRs
- **Secret read** limited to the consumer's namespace only
- **No cross-namespace Secret reads** - connection-details cannot be accessed across namespace boundaries

### Data Classification

The platform enforces clear data classification for outputs:

- **`status.outputs.*`** - Treated as **public/non-secret** by policy. Safe to inline into environment variables and log. Suitable for ARNs, endpoints, IDs, and other non-sensitive identifiers.
- **Connection-details Secrets** - Treated as **secret/sensitive** by policy. Must be referenced via `secretKeyRef`. Suitable for credentials, access keys, passwords, and other sensitive values.

The reference pattern makes this classification explicit:
- `outputs/` paths return literal values (non-secret data)
- `connections/` paths return secretKeyRef (sensitive data)
- `secrets/` paths return secretKeyRef (sensitive data)
- `configs/` paths return configMapKeyRef (configuration data)

### Secret Naming Convention

All XRs use deterministic connection-details Secret naming: `<xr-name>-conn` in the XR's namespace. This enables:
- Easy discovery and debugging
- Predictable reference resolution
- Clear ownership and lifecycle management

## Constraints & Limitations

### Design Constraints
- AWS EKS specific (no multi-cloud abstraction)
- EnvironmentConfigs are cluster-scoped, cannot contain secrets
- No multi-region support in initial release
- Namespace pattern fixed to `{repository}-{project}`

### Technical Constraints
- Maximum 50 XRs per project
- Environment overrides apply only to named XRD instances
- Cross-namespace references require explicit `::` notation
- Per-project configs can only override XRD inputs, not patch resources
- **Connection references** (`connections/<xr>/<key>`) limited to same namespace only
- **Secret and ConfigMap references** (`secrets/...`, `configs/...`) limited to same namespace only
- **Cross-project and platform references** limited to `outputs/` only - cross-namespace Secret/ConfigMap access requires RBAC not provided by default
- Deterministic Secret naming required for all XRs publishing connection details

## Future Enhancements

### Planned XRDs
- **Messaging**: SQS/SNS queue abstractions
- **Cache**: Redis/ElastiCache patterns
- **Database**: RDS provisioning (if needed)
- **Monitoring**: Advanced observability configuration

### Potential Extensions
- Type-safe CUE imports generated from XRDs
- Multi-region aware XRDs
- Custom XRD contributions from teams
- Dynamic preview environment configuration

---

**Component Owner**: Platform Core
**Related Documents**: [Core Architecture](01-core-architecture.md), [Configuration & Discovery](02-configuration-discovery.md), [Release & Deployment](05-release-deployment.md)
**Last Updated**: 2025-10-04