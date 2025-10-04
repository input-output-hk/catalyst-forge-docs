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

### Universal Reference Pattern

All resource references follow a consistent pattern that works for both environment variables and volume mounts:

**Basic Pattern**: `<type>/<name>/[field]`
- Example: `secrets/database/password`

**Cross-Project Pattern**: `<repository>-<project>::<type>/<name>/[field]`
- Example: `auth-repo-identity::secrets/signing/key`

**Platform Pattern**: `platform::<type>/<name>/[field]`
- Example: `platform::configs/feature-flags/enabled`

Platform resources are infrastructure components managed by the platform team that multiple projects can reference (shared API keys, certificates, feature flags).

## Architecture

### XRD Catalog

| XRD | Purpose | Key Features | Example Use Case |
|-----|---------|--------------|------------------|
| **Deployment** | Stateless services | Horizontal scaling, rolling updates, health checks, resource limits | Web servers, APIs, stateless workers |
| **Stateful** | Services with persistent identity | Ordered deployment, stable storage, persistent identifiers | Databases, message queues, distributed systems |
| **Job** | One-time tasks | Completion tracking, retry policies, TTL cleanup | Migrations, batch processing, initialization |
| **CronJob** | Scheduled tasks | Cron syntax, concurrency control, history limits | Reports, cleanup, recurring operations |
| **Network** | Service exposure & DNS | HTTP routing, DNS records, TLS termination, rate limiting | External access, service discovery |
| **Secrets** | External secrets sync | AWS Secrets Manager integration, automatic rotation, environment prefixing | Database credentials, API keys |
| **ConfigMaps** | Application configuration | Key-value pairs, file-based configs, nested structures | Application settings, feature flags |
| **Storage** | Persistent volumes | EBS provisioning, backup scheduling, volume expansion | Database storage, file uploads |

### Environment Configuration Model

The platform uses a two-tier configuration system with Crossplane EnvironmentConfigs:

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

### Namespace Strategy

Each project automatically receives a namespace: `<repository>-<project>`

Examples:
- Repository `backend-repo`, Project `api` → Namespace `backend-repo-api`
- Repository `monorepo`, Project `auth-service` → Namespace `monorepo-auth-service`

This maintains consistency with the platform's canonical triplet pattern while ensuring global uniqueness without API lookups.

## Configuration Example

A minimal example demonstrating XRD composition and reference patterns:

```yaml
deployment:
  resources:
    # External secrets from AWS Secrets Manager
    - apiVersion: forge.io/v1
      kind: Secrets
      metadata:
        name: myapp-secrets
      spec:
        secrets:
          - name: database
            path: myapp/database  # Becomes {env}/myapp/database
          - name: tls-cert
            path: certificates/wildcard  # Becomes global/certificates/wildcard
            global: true

    # Main application deployment
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
                source: secrets/myapp-secrets/database  # Same project
              AUTH_SERVICE:
                source: backend-repo-auth::configs/service/url  # Cross-project
              FEATURE_FLAGS:
                source: platform::configs/features/production  # Platform resource
            mounts:
              - mountPath: /etc/app
                source: configs/app-settings
        replicas: 3

    # Service networking
    - apiVersion: forge.io/v1
      kind: Network
      metadata:
        name: myapp-network
      spec:
        workloadRef: myapp-web
        subdomain: myapp  # Creates myapp.{environment-domain}
        port: 8080
```

## Operations

### Secret Path Management

The platform automatically applies environment prefixes to AWS Secrets Manager paths:

| Secret Type | Path in XRD | Actual AWS Path |
|------------|-------------|-----------------|
| Environment-specific | `myapp/database` | `{env}/myapp/database` |
| Global | `certificates/wildcard` (with `global: true`) | `global/certificates/wildcard` |

### Deployment Resolution Process

1. **CUE Evaluation** - Project configurations render with `@artifact()` references resolved
2. **XR Creation** - Argo CD applies XRs to the cluster
3. **EnvironmentConfig Loading** - Compositions load cluster-wide and per-project configs
4. **Value Resolution** - Applied using precedence rules
5. **Resource Generation** - Compositions create Kubernetes resources
6. **Operator Reconciliation** - External Secrets, External DNS, and other operators reconcile

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

## Key Design Decisions

### Unified Reference Pattern
All resource references use the same `<type>/<name>/[field]` pattern, providing a single mental model. This eliminates the need to remember different patterns for different resource types.

### Repository-Prefixed Namespaces
Namespaces follow `{repository}-{project}` pattern to ensure global uniqueness without API lookups, maintaining consistency with the platform's canonical triplet pattern.

### Separate Workload XRDs
Each workload type has its own XRD rather than a single `Workload` with a type field. This provides cleaner schemas, better validation, and avoids discriminated union complexity.

### Environment-Agnostic Secrets
Secrets XRDs contain no environment references. A `global` boolean distinguishes between environment-specific (default) and global secrets. Composition functions handle prefixing automatically.

### Two-Tier EnvironmentConfig
Environment configuration uses native Crossplane EnvironmentConfigs in two tiers, providing clear hierarchy with predictable precedence while remaining GitOps-native.

### XRD Input Overrides
Per-project EnvironmentConfigs override XRD inputs rather than patching final manifests. This maintains type safety and ensures overrides work across implementation changes.

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
**Last Updated**: 2025-10-03