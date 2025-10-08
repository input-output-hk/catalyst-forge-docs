# Implementation Strategy

## Table of Contents

- [Executive Summary](#executive-summary)
- [Core Concepts](#core-concepts)
  - [Implementation Philosophy](#implementation-philosophy)
  - [Bootstrap Strategy](#bootstrap-strategy)
  - [Development Methodology](#development-methodology)
  - [Risk Reduction Strategy](#risk-reduction-strategy)
- [Build Sequence](#build-sequence)
  - [Phase 0: Foundation Libraries & Bootstrap CLI](#phase-0-foundation-libraries--bootstrap-cli)
  - [Phase 1: Infrastructure & XRD Bootstrap](#phase-1-infrastructure--xrd-bootstrap)
  - [Phase 2: Minimal Platform Core](#phase-2-minimal-platform-core)
  - [Phase 3: Platform Completion](#phase-3-platform-completion)
  - [Phase 4: Production Hardening](#phase-4-production-hardening)
- [Development Environment](#development-environment)
  - [Infrastructure Specifications](#infrastructure-specifications)
  - [Developer Access Model](#developer-access-model)
  - [Testing Strategy](#testing-strategy)
- [Integration Points](#integration-points)
  - [Critical Path Dependencies](#critical-path-dependencies)
  - [External Service Requirements](#external-service-requirements)
- [Success Criteria](#success-criteria)
  - [Phase Validation](#phase-validation)
  - [Integration Milestones](#integration-milestones)
- [Risk Mitigation](#risk-mitigation)
  - [Technical Risks](#technical-risks)
  - [Mitigation Strategies](#mitigation-strategies)

## Executive Summary

This document defines the implementation strategy for building the Catalyst Forge platform from concept to initial production deployment. The strategy employs a bootstrap-first approach using a temporary CLI to establish the platform's self-hosting capabilities before building the full system.

The implementation progresses through five phases, starting with a bootstrap CLI that validates the core release and GitOps patterns, then transitioning to the full platform once self-management is proven. This approach validates the most complex architectural patterns early while building reusable components.

Total estimated timeline: 14-16 weeks from start to initial production readiness.

## Core Concepts

### Implementation Philosophy

**Bootstrap-First Development**
Build a temporary CLI that can create releases and manage XRDs, enabling the platform to be deployed via its own GitOps patterns from the beginning. This validates the self-hosting model before committing to the full platform architecture.

**Production Infrastructure Early**
Deploy real infrastructure on OVH bare-metal servers from the beginning. All development and testing happens against production-like infrastructure to validate assumptions immediately.

**Walking Skeleton Pattern**
Build a minimal end-to-end system that exercises all architectural layers before adding features. The skeleton includes real NATS messaging, real PostgreSQL persistence, real Kubernetes workloads, and real GitOps deployments.

### Bootstrap Strategy

The platform faces a fundamental bootstrapping challenge: it needs to deploy itself using the same GitOps patterns it provides to applications, but those patterns are produced by the platform itself.

The solution is a temporary bootstrap CLI that implements just enough functionality to:
1. Create Crossplane packages (xpkg) for platform XRDs
2. Generate artifact OCI tracking images
3. Create Release OCI images containing Kubernetes resources
4. Publish all artifacts to registries

This CLI enables manual creation of the initial release pointer files, allowing Argo CD to deploy the platform using the same Custom Management Plugin that will later manage applications. Once the minimal platform is operational, it takes over these responsibilities and the CLI is retired.

### Development Methodology

**Incremental Validation**
Each phase must demonstrate working functionality before proceeding. The bootstrap CLI validates GitOps patterns, the minimal platform validates core workflows, then additional features are layered on the working foundation.

**Component Reuse**
Code written for the bootstrap CLI (artifact creation, release packaging, CUE evaluation) directly transfers to the Worker Service, making the temporary CLI a valuable investment rather than throwaway code.

**Single Environment Focus**
Initially target only the on-premises profile on OVH infrastructure. AWS profile implementation deferred to avoid complexity multiplication during initial development.

### Risk Reduction Strategy

Validate high-risk architectural decisions early through the bootstrap process:
- GitOps self-management via Release OCI images
- Argo CD Custom Management Plugin resource extraction
- Crossplane XRD composition complexity
- NATS request-reply messaging patterns
- StatefulSet Git caching performance

The bootstrap CLI allows testing the most complex integration (self-hosting via releases) before building supporting infrastructure.

## Build Sequence

### Phase 0: Foundation Libraries & Bootstrap CLI
**Duration: 3 weeks**

**Required Foundation Libraries (Full Implementation):**
- `domain` - All entity definitions (Release, Artifact, PipelineRun, etc.)
- `errors` - Error types and classification
- `schemas` - Generated Go types from CUE schemas
- `fs` - Local and S3/MinIO implementations for file operations
- `git` - Repository cloning, fetching, and worktree management
- `repository` - Repository configuration parsing (`repo.cue`)
- `project` - Project configuration parsing (`project.cue`)
- `discovery` - Project traversal and DAG construction
- `cue` - CUE evaluation and Kubernetes resource rendering
- `oci` - OCI image manipulation and registry operations
- `earthly` - Earthfile parsing for target dependency analysis

**Required Foundation Libraries (Minimal Implementation):**
- `secrets` - StaticProvider only (environment variables for registry credentials)
- `publishers` - OCI publisher only (for pushing Release and Artifact OCI images)
- `provenance` - Basic artifact tracking image creation (metadata layer, no SBOM initially)

**Deferred Foundation Libraries (Not Needed Yet):**
- `mq` - NATS messaging (Phase 2)
- `database` - PostgreSQL operations (Phase 2)
- `auth` - Authentication/authorization (Phase 3)
- `observability` - Logging/metrics/tracing (Phase 3)
- `execution` - Task execution abstraction (Phase 2)
- `gitops` - GitOps repository management (Phase 3)
- Additional publishers beyond OCI (Phase 3)

**Bootstrap CLI (`forge-bootstrap`):**
- Repository discovery using `discovery` library
- Configuration parsing using `repository` and `project` libraries
- CUE evaluation using `cue` library
- Crossplane package (xpkg) creation
- Artifact OCI tracking image generation using minimal `provenance`
- Release OCI image packaging with resource layer
- Registry push using `oci` and minimal `publishers`
- Static credential management via environment variables

**Validation:**
- Unit tests for all required libraries
- CLI can successfully create XRD packages
- CLI generates valid Release OCI images with artifact references
- Artifact tracking images include required metadata
- Manual test of Argo CD CMP extracting resources from Release OCI

### Phase 1: Infrastructure & XRD Bootstrap
**Duration: 2 weeks**

**Infrastructure Deployment:**
- OVH bare-metal Kubernetes cluster (3 control, 3 worker nodes)
- NATS JetStream cluster
- PostgreSQL (CloudNativePG)
- MinIO for object storage
- Harbor for container registry
- Argo CD with app-of-apps pattern
- Crossplane with providers

**XRD Bootstrap Process:**
1. Write platform XRDs in `catalyst-forge-xrds` repository
2. Use bootstrap CLI to create xpkg packages
3. Use bootstrap CLI to create Release OCI for XRDs
4. Manually create ReleasePointer files in GitOps repository
5. Deploy Argo CD Custom Management Plugin
6. Configure Argo CD applications to use CMP
7. Verify XRDs deploy successfully via GitOps

**Validation:**
- Argo CD successfully extracts resources from Release OCI
- Crossplane accepts and reconciles platform XRDs
- Basic XRD can create a test resource
- GitOps repository structure established

### Phase 2: Minimal Platform Core
**Duration: 4 weeks**

**Complete Deferred Foundation Libraries:**
- `mq` - NATS JetStream implementation for message queue abstraction
- `database` - PostgreSQL connection pooling and query building
- `execution` - Task execution abstraction with Earthly adapter
- `provenance` - Full implementation with SBOM generation

**Platform API (Minimal):**
- PostgreSQL schema setup and migrations
- Core endpoints (no authentication initially):
  - `POST /v1/runs` (create pipeline run)
  - `GET /v1/runs/{id}` (get status)
  - Internal status update endpoints for workers
- Entity management (PipelineRun, Release, Deployment)
- Argo Workflow submission

**Worker Service:**
- All job handlers in single StatefulSet initially:
  - Discovery handler (repository traversal, CUE parsing)
  - CI handler (Earthly execution using `execution` library)
  - Artifact handler (producer/publisher orchestration, full `provenance`)
  - Release handler (reusing bootstrap CLI logic)
- NATS pull consumer implementation using `mq` library
- Request-reply pattern
- Git repository caching using `git` library
- Object storage for results/logs using `fs` library

**Dispatcher:**
- Job submission to NATS using `mq` library
- Reply correlation
- Timeout handling
- Result return to Argo Workflows

**Argo Workflows:**
- Workflow engine deployment
- Basic workflow templates
- Integration with dispatcher pods

**Transition Point:**
Once this phase is complete, the platform can create its own releases through the normal pipeline flow. The bootstrap CLI is no longer needed.

**Validation:**
- Manual pipeline trigger creates a Release
- Release OCI image matches bootstrap CLI output
- Worker handlers successfully process all job types
- End-to-end flow from API to Release creation

### Phase 3: Platform Completion
**Duration: 4 weeks**

**Complete Remaining Foundation Libraries:**
- `auth` - Full Keycloak integration and authorization
- `observability` - Structured logging, metrics, and tracing
- `gitops` - GitOps repository management and ReleasePointer updates
- `secrets` - Additional providers (AWS Secrets Manager, Vault, Kubernetes)
- `publishers` - Complete suite (Docker, GitHub, PyPI, S3, etc.)

**Authentication & Authorization:**
- Keycloak deployment and configuration
- Platform API authentication middleware using `auth` library
- Service account creation for workers
- GitHub OIDC federation

**GitHub Integration:**
- Webhook ingestion
- Commit status updates
- OIDC token exchange

**Worker Service Separation:**
- Split into specialized StatefulSets:
  - `worker-discovery` (light resources)
  - `worker-ci` (heavy CPU/memory)
  - `worker-artifact` (network I/O focused)
  - `worker-release` (moderate resources)
- KEDA autoscaling per worker type
- Resource optimization per workload
- Integration with `observability` library for metrics/logging

**Enhanced Platform API:**
- Complete REST API surface
- Event emission
- Pipeline phase/task/step tracking
- Release management endpoints
- Deployment endpoints
- Integration with `gitops` library for automated pointer updates

**Additional Platform Operators:**
- External DNS
- External Secrets Operator (integrated with `secrets` library)
- cert-manager
- Observability stack (Grafana Alloy)

**Validation:**
- GitHub push triggers pipeline automatically
- Authentication required for all API access
- Workers scale based on load
- Complete platform self-management via GitOps
- All publisher types functional
- Secrets synchronized from external providers

### Phase 4: Production Hardening
**Duration: 2 weeks**

**Service Mesh:**
- Istio Ambient Mode deployment
- Gateway API configuration
- Service authorization policies
- mTLS verification

**Observability:**
- Metrics collection and dashboards
- Log aggregation
- Distributed tracing
- Alert rules

**Reliability:**
- Circuit breakers
- Retry policies
- Rate limiting
- Resource quotas
- Backup and restore procedures

**Performance:**
- Load testing (100 concurrent pipelines)
- Cache optimization
- Database query optimization
- Network policy tuning

**Validation:**
- Platform can manage its own upgrades
- Recovery from component failures < 2 minutes
- 99% pipeline success rate under load
- Security scanning passes

## Development Environment

### Infrastructure Specifications

**OVH Bare-Metal Configuration:**
```
Control Plane Nodes (3):
- CPU: 8 cores
- RAM: 32 GB
- Storage: 500 GB NVMe
- Network: 1 Gbps private network

Worker Nodes (3):
- CPU: 16 cores
- RAM: 64 GB
- Storage: 1 TB NVMe
- Network: 1 Gbps private network

Shared Storage:
- 10 TB object storage for artifacts/logs
- Distributed across worker nodes via MinIO
```

**Kubernetes Distribution:**
- RKE2 or k3s for simplified management
- Cilium CNI for network policies
- Local-path provisioner for development
- Longhorn for persistent volumes

### Developer Access Model

**Bootstrap Phase:**
- Direct kubectl access for manual operations
- Bootstrap CLI runs from developer workstations
- Manual GitOps repository management

**Post-Bootstrap:**
- Individual namespaces per developer
- RBAC for namespace isolation
- Shared platform services
- Individual GitOps application instances

### Testing Strategy

**Bootstrap Validation:**
- Manual testing of Release OCI format
- Argo CD CMP extraction verification
- XRD deployment confirmation

**Integration Testing:**
- Real infrastructure for all tests
- No mocks for platform contracts
- End-to-end pipeline execution
- GitOps deployment verification

**Load Testing:**
- Gradual load increase on real cluster
- Monitor resource utilization
- Identify bottlenecks early

## Integration Points

### Critical Path Dependencies

**Phase 0 → Phase 1:**
- Bootstrap CLI must successfully create releases
- Release OCI format must be valid

**Phase 1 → Phase 2:**
- Argo CD CMP must extract resources correctly
- XRDs must be deployed via GitOps
- Infrastructure must be stable

**Phase 2 → Phase 3:**
- Platform must create releases matching CLI output
- Core workflow must be validated
- Self-management must be proven

### External Service Requirements

**Required from Day 1:**
- GitHub repository access
- OVH infrastructure access
- DNS provider account
- Container registry (Harbor)

**Required by Phase 2:**
- NATS JetStream cluster
- PostgreSQL database
- MinIO object storage

**Required by Phase 3:**
- GitHub API access (webhooks, status)
- GitHub OIDC provider
- External identity provider (optional)

## Success Criteria

### Phase Validation

**Phase 0 Success:**
- Bootstrap CLI creates valid xpkg files
- Bootstrap CLI generates Release OCI images
- Libraries have comprehensive test coverage

**Phase 1 Success:**
- Platform XRDs deployed via GitOps
- Argo CD CMP extracts resources from Release OCI
- Infrastructure stable for 48 hours
- Manual ReleasePointer updates trigger deployments

**Phase 2 Success:**
- Pipeline creates Release without bootstrap CLI
- All job types execute successfully
- NATS request-reply pattern works reliably
- Platform can update itself via GitOps

**Phase 3 Success:**
- GitHub integration fully automated
- Authentication protects all endpoints
- Workers scale with load
- Complete API surface available

**Phase 4 Success:**
- 99% pipeline success rate under load
- Recovery from failures < 2 minutes
- All traffic encrypted via mTLS
- Observability data flowing

### Integration Milestones

1. **First Bootstrap Release:** CLI creates platform's own Release OCI
2. **First GitOps Deployment:** Platform deploys itself from Release
3. **First Pipeline Release:** Platform creates Release through pipeline
4. **First Self-Update:** Platform updates itself via its own pipeline
5. **First Application:** External application deployed via platform

## Risk Mitigation

### Technical Risks

**Risk: Release OCI format incompatibility with Argo CD CMP**
- **Mitigation:** Test CMP extraction with bootstrap CLI output early
- **Fallback:** Adjust OCI layer structure to match CMP requirements

**Risk: Bootstrap CLI logic doesn't transfer to Worker Service**
- **Mitigation:** Design CLI with clear separation of concerns
- **Fallback:** Refactor during Phase 2 if necessary

**Risk: Self-hosting creates circular dependencies**
- **Mitigation:** Bootstrap CLI breaks the cycle initially
- **Fallback:** Maintain manual deployment option for platform core

**Risk: NATS ephemeral approach causes job loss**
- **Mitigation:** Implement robust timeout/retry patterns early
- **Fallback:** Add persistence to NATS if required

**Risk: StatefulSet Git caching doesn't perform**
- **Mitigation:** Test with large repositories in Phase 2
- **Fallback:** Shared cache via Redis or distributed filesystem

### Mitigation Strategies

**Incremental Validation:**
Test each architectural decision as soon as possible. The bootstrap CLI validates GitOps patterns before building the full platform.

**Component Reuse:**
Bootstrap CLI code transfers directly to Worker Service, ensuring early work provides lasting value.

**Rollback Capability:**
GitOps enables easy rollback. Platform can always be redeployed from previous Release if updates fail.

**Monitoring First:**
Deploy observability before services. Never fly blind during development.

---

**Related Documents:**
- Architecture Overview - System design and principles
- Platform Contracts - Infrastructure requirements
- Foundation Libraries - Shared code structure
- Developer Guide - Configuration patterns

**Timeline:** 14-16 weeks from start to initial production readiness

**Last Updated:** 2025-10-07