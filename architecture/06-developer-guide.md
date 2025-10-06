# Developer Guide

## Table of Contents

- [Executive Summary](#executive-summary)
- [Core Concepts](#core-concepts)
  - [Repository and Project Structure](#repository-and-project-structure)
  - [Configuration with CUE](#configuration-with-cue)
  - [Pipeline Workflow](#pipeline-workflow)
  - [Deployment Model](#deployment-model)
- [Repository Setup](#repository-setup)
  - [File Organization](#file-organization)
  - [Repository Configuration](#repository-configuration)
  - [Project Configuration](#project-configuration)
- [Pipeline Configuration](#pipeline-configuration)
  - [Phase Participation](#phase-participation)
  - [Earthly Integration](#earthly-integration)
- [Artifact Configuration](#artifact-configuration)
  - [Artifact Types](#artifact-types)
  - [Publisher Configuration](#publisher-configuration)
  - [Artifact References](#artifact-references)
- [Deployment Configuration](#deployment-configuration)
  - [Crossplane XRDs](#crossplane-xrds)
  - [Reference Resolution](#reference-resolution)
  - [Environment-Agnostic Configuration](#environment-agnostic-configuration)
- [Release Configuration](#release-configuration)
  - [Release Triggers](#release-triggers)
  - [Release Identification](#release-identification)
- [Common Workflows](#common-workflows)
  - [New Project Setup](#new-project-setup)
  - [Adding New Artifact](#adding-new-artifact)
  - [Updating Deployment Configuration](#updating-deployment-configuration)
  - [Environment Promotion](#environment-promotion)
  - [Troubleshooting Pipeline Failures](#troubleshooting-pipeline-failures)
- [Integration Points](#integration-points)
  - [GitHub Actions](#github-actions)
  - [Platform CLI](#platform-cli)
  - [Platform API](#platform-api)
- [Constraints](#constraints)
  - [Configuration Constraints](#configuration-constraints)
  - [Pipeline Constraints](#pipeline-constraints)
  - [Artifact Constraints](#artifact-constraints)
  - [Deployment Constraints](#deployment-constraints)
- [Future Considerations](#future-considerations)
  - [Planned Features](#planned-features)
  - [Extension Points](#extension-points)

## Executive Summary

This document explains how developers use the Catalyst Forge platform to build, release, and deploy applications. The platform provides a configuration-driven workflow where developers define pipeline behavior, artifact publishing, and infrastructure requirements through CUE configuration files.

Projects are discovered automatically through repository conventions, executed through configurable pipelines, packaged as immutable releases, and deployed via GitOps. The guide covers repository setup, configuration patterns, artifact management, deployment specifications, and common workflows.

## Core Concepts

### Repository and Project Structure

The platform distinguishes between repository-level and project-level configuration:

**Repository:** A GitHub repository containing one or more projects. Repository configuration defines platform-wide settings, phase definitions, and available publishers.

**Project:** A deliverable unit within a repository, identified by a `.forge` directory containing project-specific configuration. Projects define pipeline participation, artifacts, and deployment resources. Projects are identified globally by `repository/project-name`.

**Release/Artifact Identifier:** Releases and artifacts follow the `repository/project/commit-sha` pattern for global uniqueness without centralized coordination. This canonical triplet identifies immutable snapshots at specific commits.

### Configuration with CUE

All platform configuration uses CUE (Configure, Unify, Execute) for type-safe, composable configuration with built-in validation. CUE provides:
- Strong typing with schema validation
- Configuration composition and merging
- Built-in functions for transformation
- Native Kubernetes resource generation

The platform provides CUE schemas for all configuration structures, enabling IDE completion and compile-time validation.

### Pipeline Workflow

Pipelines execute through sequential phase groups with fail-fast behavior:

1. **Discovery:** Platform discovers projects and builds execution DAG
2. **Phase Execution:** Phases execute sequentially by group number, tasks execute in parallel within phase
3. **Release Creation:** Conditional release creation based on trigger evaluation
4. **Deployment:** GitOps-based deployment to environments

Any failure immediately terminates the pipeline, ensuring continuous integration remains green.

### Deployment Model

Applications deploy through Crossplane XRDs (Composite Resource Definitions) that abstract Kubernetes complexity:

**Infrastructure Abstractions:** Platform provides XRDs for common patterns (Deployment, Stateful, Network, Secrets, ConfigMaps, Storage, Database)

**Environment Configuration:** EnvironmentConfigs provide environment-specific values at deployment time, enabling environment-agnostic releases

**Reference Resolution:** Universal reference pattern for accessing values from other XRDs (outputs, connections, secrets, configs)

See Implementation Guide: Infrastructure Abstractions for complete XRD catalog and EnvironmentConfig specifications.

## Repository Setup

### File Organization

```
repository/
├── .forge/                    # Repository configuration (required)
│   ├── repo.cue              # Repository settings
│   └── cue.mod/              # CUE module definition
├── services/
│   ├── auth/
│   │   └── .forge/           # Project configuration
│   │       ├── project.cue   # Project settings
│   │       └── cue.mod/      # CUE module definition
│   └── api/
│       └── .forge/
│           ├── project.cue
│           └── cue.mod/
└── libraries/
    └── common/
        └── .forge/
            ├── project.cue
            └── cue.mod/
```

### Repository Configuration

Repository configuration in `.forge/repo.cue` defines platform-wide settings:

```cue
package repo

// Schema version (required)
forgeVersion: "v1.0.0"

// Tagging strategy
tagging: {
    strategy: "monorepo"  // or "tag-all"
}

// Phase definitions
phases: {
    validate: {
        group: 1
        description: "Linting and formatting"
        timeout: "10m"
    }
    test: {
        group: 1
        description: "Unit and integration tests"
        required: true
    }
    build: {
        group: 2
        description: "Compile and package"
        timeout: "20m"
    }
}

// Publisher configuration
publishers: {
    "docker-hub": {
        type: "docker"
        registry: "docker.io"
        namespace: "mycompany"
        credentials: {
            provider: "aws"
            secretName: "docker-hub-credentials"
        }
    }
}
```

**Tagging Strategies:**

`monorepo`: Tags follow pattern `<project-name>/v*` for individual project releases

`tag-all`: Bare version tags (`v*`) trigger releases for all projects

### Project Configuration

Project configuration in `<project>/.forge/project.cue` defines project-specific behavior:

```cue
package project

// Phase participation
phases: {
    validate: {
        steps: [
            {
                name: "lint"
                action: "earthly"
                target: "+lint"
            }
        ]
    }
    test: {
        steps: [
            {
                name: "unit-tests"
                action: "earthly"
                target: "+test"
            }
        ]
    }
    build: {
        steps: [
            {
                name: "compile"
                action: "earthly"
                target: "+build"
            }
        ]
    }
}

// Artifact configuration
artifacts: {
    "api-server": {
        type: "container"
        producer: {
            earthly: {
                target: "+docker"
                imageRef: "api-server:latest"
            }
        }
        publishers: ["docker-hub"]
    }
}

// Release configuration
release: {
    on: [
        {branch: "main"},
        {tag: true}
    ]
}

// Deployment configuration
deploy: {
    resources: [
        {
            apiVersion: "forge.projectcatalyst.io/v1alpha1"
            kind: "Deployment"
            metadata: {
                name: "api-server"
            }
            spec: {
                image: @artifact("api-server").image
                replicas: 3
                port: 8080
            }
        },
        {
            apiVersion: "forge.projectcatalyst.io/v1alpha1"
            kind: "Network"
            metadata: {
                name: "api-server"
            }
            spec: {
                hostname: "api"
                port: 8080
                targetRef: {
                    name: "api-server"
                }
            }
        }
    ]
}
```

## Pipeline Configuration

### Phase Participation

Projects participate in phases by defining steps:

```cue
phases: {
    <phase-name>: {
        steps: [
            {
                name: string          // Step identifier
                action: "earthly"     // Execution action
                target: string        // Earthly target (e.g., "+test")
                timeout?: string      // Optional timeout override
            }
        ]
    }
}
```

**Phase Execution:**
- Phases execute sequentially by group number
- Steps within phase execute in parallel
- Any step failure fails the entire phase
- Any phase failure terminates the pipeline

### Earthly Integration

All build and test execution uses Earthly targets:

```cue
steps: [
    {
        name: "lint"
        action: "earthly"
        target: "+lint"
    },
    {
        name: "unit-tests"
        action: "earthly"
        target: "+test"
    },
    {
        name: "build-binary"
        action: "earthly"
        target: "+build"
    }
]
```

The platform expects corresponding Earthly targets in the project's `Earthfile`. Workers maintain persistent Git caches and warm Earthly connections for performance.

## Artifact Configuration

### Artifact Types

The platform supports three artifact types:

**Container:** Docker/OCI container images
```cue
artifacts: {
    "web-app": {
        type: "container"
        producer: {
            earthly: {
                target: "+docker"
                imageRef: "web-app:latest"
            }
        }
        publishers: ["docker-hub", "ecr"]
    }
}
```

**Binary:** Standalone executables
```cue
artifacts: {
    "cli-tool": {
        type: "binary"
        producer: {
            earthly: {
                target: "+build"
                binaryPath: "dist/cli"
                os: "linux"
                arch: "amd64"
            }
        }
        publishers: ["github-releases"]
    }
}
```

**Archive:** Compressed file collections
```cue
artifacts: {
    "documentation": {
        type: "archive"
        producer: {
            earthly: {
                target: "+docs"
                archivePath: "dist/docs.tar.gz"
                format: "tar.gz"
            }
        }
        publishers: ["s3"]
    }
}
```

### Publisher Configuration

Publishers distribute artifacts to registries:

```cue
publishers: {
    "docker-hub": {
        type: "docker"
        registry: "docker.io"
        namespace: "mycompany"
        credentials: {
            provider: "aws"
            secretName: "docker-hub-credentials"
        }
    },
    "github-releases": {
        type: "github"
        repository: "mycompany/myrepo"
        credentials: {
            provider: "aws"
            secretName: "github-token"
        }
    },
    "pypi": {
        type: "pypi"
        repository: "https://upload.pypi.org/legacy/"
        credentials: {
            provider: "aws"
            secretName: "pypi-credentials"
        }
    }
}
```

**Available Publisher Types:**
- `docker`: Docker/OCI registries
- `github`: GitHub Releases
- `pypi`: Python Package Index
- `s3`: S3-compatible object storage
- `crate`: Rust crate registry
- `npm`: NPM registry

### Artifact References

Artifacts are referenced in deployment configurations using `@artifact()` function:

```cue
deploy: {
    resources: [
        {
            apiVersion: "forge.projectcatalyst.io/v1alpha1"
            kind: "Deployment"
            spec: {
                image: @artifact("web-app").image
                // Resolves to tracking OCI image URI
            }
        }
    ]
}
```

The platform resolves `@artifact()` references during release creation, substituting them with actual artifact locations from the build phase.

## Deployment Configuration

### Crossplane XRDs

Applications declare infrastructure through XRDs:

**Deployment XRD (Stateless Services):**
```cue
{
    apiVersion: "forge.projectcatalyst.io/v1alpha1"
    kind: "Deployment"
    metadata: {
        name: "api-server"
    }
    spec: {
        image: @artifact("api-server").image
        replicas: 3
        port: 8080
        env: [
            {
                name: "DATABASE_URL"
                value: "connections/postgres/url"
            },
            {
                name: "API_KEY"
                value: "secrets/app-secrets/api/key"
            }
        ]
        resources: {
            requests: {
                cpu: "100m"
                memory: "128Mi"
            }
        }
    }
}
```

**Network XRD (Service Exposure & DNS):**
```cue
{
    apiVersion: "forge.projectcatalyst.io/v1alpha1"
    kind: "Network"
    metadata: {
        name: "api-server"
    }
    spec: {
        hostname: "api"
        port: 8080
        targetRef: {
            name: "api-server"
        }
        tls: {
            enabled: true
        }
    }
}
```

**Secrets XRD (External Secrets Sync):**
```cue
{
    apiVersion: "forge.projectcatalyst.io/v1alpha1"
    kind: "Secrets"
    metadata: {
        name: "app-secrets"
    }
    spec: {
        secrets: [
            {
                name: "api"
                ref: {
                    provider: "aws"
                    secretName: "prod/api-server/api-key"
                }
            },
            {
                name: "database"
                ref: {
                    provider: "aws"
                    secretName: "prod/api-server/db-credentials"
                }
            }
        ]
    }
}
```

**Database XRD (Application Database):**
```cue
{
    apiVersion: "forge.projectcatalyst.io/v1alpha1"
    kind: "Database"
    metadata: {
        name: "postgres"
    }
    spec: {
        engine: "postgres"
        version: "15"
        storage: "20Gi"
    }
}
```

### Reference Resolution

XRDs can reference outputs from other XRDs:

**Output References (Literal Values):**
```cue
env: [
    {
        name: "BUCKET_ARN"
        value: "outputs/assets/arn"
    }
]
```

**Connection References (Kubernetes Secrets):**
```cue
env: [
    {
        name: "DATABASE_URL"
        value: "connections/postgres/url"
    }
]
```

**Secret References (From Secrets XRD):**
```cue
env: [
    {
        name: "API_KEY"
        value: "secrets/app-secrets/api/key"
    }
]
```

**Config References (From ConfigMaps XRD):**
```cue
env: [
    {
        name: "FEATURE_FLAGS"
        value: "configs/app-settings/features/enabled"
    }
]
```

**Cross-Namespace References (Outputs Only):**
```cue
env: [
    {
        name: "SHARED_BUCKET"
        value: "platform::outputs/shared-storage/bucket"
    }
]
```

### Environment-Agnostic Configuration

Releases contain no environment-specific values. EnvironmentConfigs provide environment-specific values at deployment time:

**Development Environment:**
```yaml
# dev/env.yml
replicas: 1
resources:
  requests:
    cpu: "50m"
    memory: "64Mi"
```

**Production Environment:**
```yaml
# production/env.yml
replicas: 10
resources:
  requests:
    cpu: "500m"
    memory: "512Mi"
```

The same release deploys to both environments with different resource allocations.

## Release Configuration

### Release Triggers

Releases are triggered based on conditions:

```cue
release: {
    on: [
        {branch: "main"},
        {branch: "release/.*"},
        {tag: true}
    ]
}
```

**Trigger Types:**

`branch`: Push to specified branch (supports regex)

`tag`: Git tag creation (respects repository tagging strategy)

`manual`: Explicit CLI trigger

### Release Identification

Releases have multiple identifiers:

**Canonical:** `repository/project/commit-sha` (immutable)

**Numeric:** Auto-incrementing per project (e.g., 42)

**Tag:** From Git tags (e.g., v1.2.3)

**Branch:** Optional branch-based (e.g., main-42)

API access supports any identifier:
```
GET /api/v1/releases/<repo>/<project>/42
GET /api/v1/releases/<repo>/<project>/v1.2.3
GET /api/v1/releases/<repo>/<project>/<commit-sha>
```

## Common Workflows

### New Project Setup

1. Create project directory with `.forge/project.cue`
2. Define phase participation with Earthly targets
3. Configure artifacts and publishers
4. Define deployment resources with XRDs
5. Configure release triggers
6. Push to GitHub to trigger first pipeline

### Adding New Artifact

1. Update `artifacts` section in project.cue
2. Add Earthly target for artifact production
3. Reference artifact in deployment with `@artifact()`
4. Commit and push to trigger build

### Updating Deployment Configuration

1. Modify XRD specifications in `deploy.resources`
2. Commit changes
3. Trigger release (based on release triggers)
4. Platform updates GitOps repository
5. Argo CD detects change and deploys

### Environment Promotion

Releases deploy to environments based on policies:

1. Release created from main branch
2. Automatically deploys to development
3. Manual approval for staging deployment
4. Manual approval for production deployment

Approvals managed through Platform API or CLI.

### Troubleshooting Pipeline Failures

**View Pipeline Status:**
```
GET /api/v1/runs/<run-id>
```

**Access Step Logs:**
Logs stored in object storage, referenced in step execution metadata

**Common Issues:**
- Earthly target failures: Check Earthfile syntax and target dependencies
- Artifact publication failures: Verify publisher credentials and registry access
- Deployment failures: Check XRD specifications and EnvironmentConfig values

## Integration Points

### GitHub Actions

Trigger pipelines from GitHub Actions workflows:

```yaml
name: Catalyst Forge CI
on: [push, pull_request]
jobs:
  trigger:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - name: Trigger Pipeline
        run: |
          forge run create \
            --repository ${{ github.repository }} \
            --branch ${{ github.ref_name }} \
            --commit ${{ github.sha }}
        env:
          FORGE_TOKEN: ${{ secrets.FORGE_TOKEN }}
```

### Platform CLI

The platform provides a CLI for manual operations:

**Create Pipeline Run:**
```bash
forge run create --repository mycompany/myrepo --branch main --commit abc123
```

**View Pipeline Status:**
```bash
forge run get <run-id>
```

**Trigger Release:**
```bash
forge release create --repository mycompany/myrepo --project api-server --commit abc123
```

**Approve Deployment:**
```bash
forge deployment approve --release-id <id> --environment production
```

### Platform API

All operations available via REST API with Keycloak authentication:

**Authentication:**
```
Authorization: Bearer <keycloak-token>
```

**API Base Path:**
```
https://api.catalyst-forge.example.com/v1
```

See API & Operations Reference for complete API specifications.

## Constraints

### Configuration Constraints

- CUE configuration only (no YAML/JSON for platform config)
- Repository must have `.forge/repo.cue` at root
- Projects must have `.forge/project.cue` in project directory
- Earthly required for all build and test execution

### Pipeline Constraints

- Sequential phase execution (phases cannot run in parallel)
- Fail-fast behavior (any failure terminates pipeline)
- Maximum pipeline duration: 24 hours
- Maximum discovery output: 1MB

### Artifact Constraints

- All artifacts tracked with OCI images
- Publishers operate idempotently
- Artifact names must be unique within project
- Maximum 50 artifacts per project

### Deployment Constraints

- Releases immutable once created
- Environment-agnostic releases required
- XRDs defined at platform level (cannot create custom XRDs)
- Reference resolution limited to same-namespace except `outputs/`

## Future Considerations

### Planned Features

**Enhanced Discovery:**
Incremental discovery for large monorepos, cached discovery results, and predictive project loading to reduce pipeline initialization time.

**Advanced Deployment Strategies:**
Progressive deployments (canary, blue-green), dynamic environment creation for feature branches, and multi-project release coordination.

**Developer Experience Improvements:**
Local development workflows with platform CLI, configuration validation tooling, and enhanced error messages with remediation guidance.

### Extension Points

The platform provides clear extension mechanisms for future capabilities:
- Custom Earthly targets for specialized build workflows
- Additional artifact types through producer/publisher pattern
- Custom XRD compositions for organization-specific patterns
- Webhook integrations for external system notifications

---

**Related Documents:**
- System Architecture - Component architecture, service model, and data flows
- Platform Contracts - Infrastructure contract specifications and adapters
- Implementation Guide - XRD catalog, EnvironmentConfig, and deployment profiles
- API & Operations Reference - REST API specifications and operational procedures

**Last Updated:** 2025-10-05
