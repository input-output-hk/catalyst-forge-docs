# Implementation Strategy

## Table of Contents

- [Executive Summary](#executive-summary)
- [Core Concepts](#core-concepts)
  - [Implementation Philosophy](#implementation-philosophy)
  - [Development Methodology](#development-methodology)
  - [Risk Reduction Strategy](#risk-reduction-strategy)
- [Build Sequence](#build-sequence)
  - [Phase 0: Foundation Libraries](#phase-0-foundation-libraries)
  - [Phase 1: Infrastructure Platform](#phase-1-infrastructure-platform)
  - [Phase 2: Core Services MVP](#phase-2-core-services-mvp)
  - [Phase 3: Pipeline Execution](#phase-3-pipeline-execution)
  - [Phase 4: Release Management](#phase-4-release-management)
  - [Phase 5: Production Hardening](#phase-5-production-hardening)
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

This document defines the implementation strategy for building the Catalyst Forge platform from concept to initial production deployment. The strategy follows a production-first development approach, deploying real infrastructure early and developing services directly against it.

The implementation progresses through six phases, building from foundational libraries through to production-ready services. Each phase delivers working components that validate architectural decisions and reduce integration risk.

Total estimated timeline: 16-20 weeks from start to initial production readiness.

## Core Concepts

### Implementation Philosophy

**Production-First Development**
Deploy real infrastructure on OVH bare-metal servers from the beginning. Develop and test all services against this production-like environment rather than local mocks or simulations. This approach validates infrastructure assumptions immediately and prevents late-stage integration surprises.

**Walking Skeleton Pattern**
Build a minimal end-to-end system that exercises all architectural layers before adding features. The skeleton includes real NATS messaging, real PostgreSQL persistence, real Kubernetes workloads, and real GitOps deployments. Features are added to this working skeleton iteratively.

**Contract-Driven Development**
Define and implement infrastructure contracts (ports) and their adapters early. Services depend only on contracts, enabling parallel development once contracts stabilize. This aligns with the platform's ports and adapters architecture.

### Development Methodology

**Bottom-Up Construction**
Build from the lowest abstraction layers upward:
1. Libraries and contracts
2. Infrastructure and platform operators
3. Core services
4. Integration layers
5. User-facing features

**Incremental Validation**
Each phase must demonstrate working functionality before proceeding. No big-bang integration—every component integrates as soon as it's built.

**Single Environment Focus**
Initially target only the on-premises profile on OVH infrastructure. AWS profile implementation deferred to avoid complexity multiplication during initial development.

### Risk Reduction Strategy

Identify and validate high-risk architectural decisions early:
- StatefulSet Git caching performance
- NATS JetStream ephemeral messaging reliability
- Crossplane XRD composition complexity
- Worker job timeout and retry patterns

Fail fast on architectural issues while the system is still malleable.

## Build Sequence

### Phase 0: Foundation Libraries
**Duration: 2 weeks**

**Deliverables:**
- CUE schema definitions for all configuration structures
- Go SDK for platform domain models (matching Domain Model entities)
- Contract interfaces for all 11 platform contracts
- Message schemas for NATS communication
- Common error types and handling patterns

**Validation:**
- Unit tests for all libraries
- Schema validation test suite
- Contract compliance test harness

### Phase 1: Infrastructure Platform
**Duration: 3 weeks**

**Deliverables:**
- OVH bare-metal Kubernetes cluster (3 control, 3 worker nodes)
- On-premises adapter deployments:
  - MinIO for object storage
  - Harbor for container registry
  - CloudNativePG for PostgreSQL
  - NATS JetStream cluster
  - Vault for secrets (or Kubernetes secrets for MVP)
- Essential platform operators:
  - Argo CD (GitOps foundation)
  - Crossplane with providers
  - External DNS
  - cert-manager
  - Istio Ambient Mode (deferred to Phase 5 for simplicity)

**Validation:**
- Successful Argo CD app-of-apps deployment
- Working Crossplane composition (simple test XRD)
- NATS cluster forming and accepting connections
- Object storage read/write verification

**Note:** Istio Ambient Mode deployment deferred to Phase 5 to reduce initial complexity. Use simple Kubernetes Ingress or port-forwarding for development access until then.

### Phase 2: Core Services MVP
**Duration: 4 weeks**

**Platform API (Minimal):**
- PostgreSQL schema setup and migrations
- Keycloak integration for authentication
- Essential endpoints:
  - `POST /v1/runs` (create pipeline run)
  - `GET /v1/runs/{id}` (get status)
  - Health/readiness endpoints
- Domain entity management (PipelineRun only)

**Worker Service (Single Type):**
- Discovery worker implementation
- Git repository caching mechanism
- NATS consumer pattern implementation
- Result storage to S3/MinIO

**Dispatcher:**
- Basic job submission to NATS
- Request-reply correlation
- Timeout handling

**Validation:**
- Manual pipeline run creation via API
- Discovery job execution end-to-end
- Git cache persistence across pod restarts
- Result retrieval from object storage

### Phase 3: Pipeline Execution
**Duration: 4 weeks**

**Argo Workflows Integration:**
- Workflow template generation from discovery
- Dynamic task DAG construction
- Dispatcher pod specifications

**Worker Service Extensions:**
- CI worker implementation (Earthly execution)
- Artifact worker implementation
- Log streaming to object storage
- Worker autoscaling via KEDA

**Platform API Extensions:**
- GitHub webhook ingestion
- Pipeline phase/task/step status tracking
- GitHub commit status updates
- Internal service-to-service authentication

**Validation:**
- Complete pipeline execution from GitHub push
- Earthly target execution in workers
- Log accessibility via presigned URLs
- Worker scaling based on queue depth

### Phase 4: Release Management
**Duration: 3 weeks**

**Release Creation:**
- CUE to Kubernetes YAML rendering
- Release OCI image packaging
- Artifact OCI tracking image creation
- Release trigger evaluation

**Deployment Orchestration:**
- GitOps repository management
- ReleasePointer updates
- Argo CD Custom Management Plugin
- Environment-specific configurations

**Platform API Extensions:**
- Release endpoints
- Deployment endpoints
- Approval workflows

**Validation:**
- Release creation from successful pipeline
- Deployment to development environment
- Resource extraction from Release OCI
- EnvironmentConfig value injection

### Phase 5: Production Hardening
**Duration: 2 weeks**

**Observability:**
- Grafana Alloy configuration
- Platform dashboards
- Alert rules
- Distributed tracing setup

**Service Mesh & Ingress:**
- Istio Ambient Mode deployment
- Gateway API configuration for ingress
- Service authorization policies
- mTLS verification for all cluster traffic
- Unified traffic management (north-south and east-west)

**Reliability:**
- Circuit breakers
- Retry policies
- Rate limiting
- Resource quotas

**Validation:**
- Load testing (100 concurrent pipelines)
- Failure injection testing
- Recovery time verification
- Security scanning

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

**Direct Cluster Access:**
- Individual namespaces per developer
- RBAC for namespace isolation
- Shared platform services
- Individual GitOps application instances

**Development Workflow:**
1. Push service changes to feature branch
2. Update Argo CD Application to track feature branch
3. Test against real infrastructure
4. Merge to main for "production" deployment

### Testing Strategy

**Integration Testing:**
- Real NATS messaging
- Real PostgreSQL persistence
- Real object storage
- No mocks for platform contracts

**Load Testing:**
- Gradual load increase on real cluster
- Monitor resource utilization
- Identify bottlenecks early

**Chaos Testing (Phase 5):**
- Pod deletion
- Network partitioning
- Node failures
- Storage exhaustion

## Integration Points

### Critical Path Dependencies

**Phase 1 → Phase 2:**
- Working Kubernetes cluster
- Functional NATS JetStream
- Available PostgreSQL

**Phase 2 → Phase 3:**
- Platform API authentication
- Worker NATS consumption
- Object storage for logs

**Phase 3 → Phase 4:**
- Complete pipeline execution
- Artifact creation
- Earthly integration

### External Service Requirements

**Required from Day 1:**
- GitHub repository access
- DNS provider (Cloudflare/other)
- Container registry (Harbor)

**Required by Phase 3:**
- GitHub API access (commit status)
- GitHub OIDC provider

**Required by Phase 4:**
- GitOps repository
- Additional environments (can be namespaces initially)

## Success Criteria

### Phase Validation

**Phase 1 Success:**
- Argo CD can deploy applications
- Crossplane can create resources
- Platform operators healthy
- Base infrastructure stable for 48 hours

**Phase 2 Success:**
- API accepts authenticated requests
- Worker processes discovery job
- Results retrievable from storage
- No message loss under normal operation

**Phase 3 Success:**
- GitHub push triggers pipeline
- All phases execute in sequence
- Logs accessible via API
- Workers scale with load

**Phase 4 Success:**
- Releases created automatically
- Deployments successful to dev environment
- Resources correctly rendered
- GitOps loop closed

**Phase 5 Success:**
- 99% pipeline success rate under load
- Recovery from component failures < 2 minutes
- All traffic encrypted via mTLS (both ingress and service-to-service)
- Gateway API handling external traffic through Istio
- Observability data flowing

### Integration Milestones

1. **First Discovery:** Worker successfully discovers projects in repository
2. **First Pipeline:** Complete pipeline execution from trigger to completion
3. **First Release:** Release OCI created and registered
4. **First Deployment:** Application running from GitOps deployment
5. **First Production Pipeline:** 100 concurrent pipelines successfully completed

## Risk Mitigation

### Technical Risks

**Risk: StatefulSet Git caching doesn't perform as expected**
- **Mitigation:** Build cache layer abstraction early, test with large repositories
- **Fallback:** Switch to shared cache via Redis or distributed filesystem

**Risk: NATS ephemeral approach causes job loss**
- **Mitigation:** Implement robust timeout/retry patterns in Phase 2
- **Fallback:** Add persistence to NATS or switch to durable message queue

**Risk: Crossplane composition complexity**
- **Mitigation:** Start with simple XRDs, add complexity incrementally
- **Fallback:** Generate raw Kubernetes manifests if composition proves unwieldy

**Risk: OVH infrastructure constraints**
- **Mitigation:** Test infrastructure limits early in Phase 1
- **Fallback:** Design for hybrid deployment with some cloud services

**Risk: Worker job isolation issues**
- **Mitigation:** Implement strong isolation boundaries from start
- **Fallback:** One worker pod per job if necessary

### Mitigation Strategies

**Architectural Flexibility:**
Keep contract interfaces stable while allowing adapter implementations to evolve. This enables infrastructure changes without service modifications.

**Feature Flags:**
Build feature toggle system early to enable/disable functionality without deployments.

**Rollback Capability:**
Every change through GitOps is revertible. Test rollback procedures regularly.

**Monitoring First:**
Deploy observability before services. Never fly blind during development.

---

**Related Documents:**
- Architecture Overview - System design and principles
- Platform Contracts - Infrastructure requirements
- Implementation Guide - Deployment profiles and configuration
- Developer Guide - Configuration patterns and workflows

**Timeline:** 16-20 weeks from start to initial production readiness

**Last Updated:** 2025-10-05