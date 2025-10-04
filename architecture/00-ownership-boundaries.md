# Document Ownership Boundaries

## Purpose
This document defines clear ownership boundaries for each architectural document, establishing what content belongs where and preventing duplication.

---

## Ownership Assignments

### 01-core-architecture.md
**Owner**: Platform Core Team

**Owns**:
- System-wide architectural principles and constraints
- Shared concepts used by multiple components:
  - OCI Image Formats (both Artifact Tracking and Release formats)
  - GitOps Repository Structure
  - Secret Management Patterns
  - Authentication & Authorization Model
- Component architecture overview and dependency graph
- Unified data flow (commit → deployment)
- Technology stack decisions
- Design decisions affecting multiple components

**Does NOT Own**:
- Component-specific implementation details
- Individual component APIs
- Component-specific domain entities
- Component-to-component interaction protocols

**References**:
- All component specifications reference shared concepts from this document
- Integration Contracts references architectural patterns

---

### 02-configuration-discovery.md
**Owner**: Configuration System Team

**Owns**:
- CUE configuration system design and schemas
- Repository configuration schema and validation
- Project configuration schema and validation
- Configuration schema versioning and evolution
- Discovery mechanisms and rules
- Configuration inheritance patterns
- KRD (Kubernetes Resource Definition) generation process
- Configuration examples and best practices

**Does NOT Own**:
- GitOps repository structure (owned by Core Architecture)
- Publisher configuration semantics (references Build & Distribution)
- Runtime configuration of pipeline execution
- Environment-specific deployment configuration

**References**:
- Core Architecture for GitOps structure and shared patterns
- Build & Distribution for publisher configuration details
- Integration Contracts for configuration consumption patterns

---

### 03-execution-orchestration.md
**Owner**: Pipeline & Execution Team

**Owns**:
- Execution philosophy and execution model
- Pipeline architecture and design
- Argo Workflows integration details
- Worker pool architecture and scaling
- AWS infrastructure usage (SQS, DynamoDB, S3) for execution
- Phase and step execution logic
- DAG construction and optimization
- Job distribution and status tracking
- Performance tuning and scaling strategies
- Execution monitoring and observability

**Does NOT Own**:
- Artifact creation or publishing (owned by Build & Distribution)
- Release creation or deployment (owned by Release & Deployment)
- Pipeline domain entities and APIs (owned by Domain Model & API Reference)
- Generic AWS infrastructure patterns (owned by Core Architecture)

**References**:
- Core Architecture for AWS service patterns and authentication
- Integration Contracts for pipeline-to-component interactions
- Domain Model & API Reference for pipeline entities and APIs

---

### 04-build-distribution.md
**Owner**: Artifacts & Distribution Team

**Owns**:
- Artifact types and definitions
- Producer contract and interface specification
- Earthly producer implementation details
- Future producer extension points
- Publisher contract and interface specification
- Registry type implementations (Docker, PyPI, GitHub, etc.)
- Artifact Tracking OCI Format specification
- Producer/publisher interaction model
- Error handling and retry logic for build/publish operations

**Does NOT Own**:
- OCI format standards and shared patterns (owned by Core Architecture)
- Pipeline execution of build steps (owned by Execution & Orchestration)
- Artifact domain entities (owned by Domain Model & API Reference)
- Publisher configuration schema (owned by Configuration & Discovery)

**References**:
- Core Architecture for OCI image format standards
- Configuration & Discovery for publisher configuration schema
- Integration Contracts for pipeline → artifacts interaction
- Domain Model & API Reference for artifact entities

---

### 05-release-deployment.md
**Owner**: Release & Deployment Team

**Owns**:
- Release philosophy and concepts
- Release identification and aliasing system
- Release creation workflow and trigger evaluation
- Artifact coordination for releases
- Resource rendering process
- Release OCI Format specification
- GitOps integration implementation details
- Pointer file format and usage
- Argo CD custom management plugin
- Environment promotion workflows
- Approval policies and processes
- Deployment workflows and rollback operations

**Does NOT Own**:
- GitOps repository structure standards (owned by Core Architecture)
- OCI format standards (owned by Core Architecture)
- Pipeline execution of release steps (owned by Execution & Orchestration)
- Release domain entities and APIs (owned by Domain Model & API Reference)
- Infrastructure abstractions and XRD specifications (owned by Infrastructure Abstractions)

**References**:
- Core Architecture for OCI formats and GitOps structure
- Integration Contracts for pipeline → release and release → GitOps interactions
- Domain Model & API Reference for release entities and APIs
- Infrastructure Abstractions for XRD catalog and environment configuration

---

### 08-infrastructure-abstractions.md
**Owner**: Platform Core Team

**Owns**:
- Crossplane XRD catalog and specifications
- Environment configuration model (EnvironmentConfig two-tier system)
- Universal reference pattern (`<type>/<name>/[field]`)
- Namespace strategy (`{repository}-{project}` pattern)
- Deployment resolution process and precedence rules
- XRD design philosophy and composability patterns
- Integration specifications for:
  - External Secrets Operator
  - External DNS
  - Envoy Gateway
  - EBS CSI Driver
- Crossplane composition function requirements

**Does NOT Own**:
- Project deployment configuration syntax (owned by Configuration & Discovery)
- Release packaging and OCI formats (owned by Core Architecture)
- Kubernetes resource rendering from CUE (owned by Release & Deployment)
- Platform operators' general patterns (owned by Core Architecture)

**References**:
- Core Architecture for shared infrastructure patterns
- Configuration & Discovery for project deployment syntax
- Release & Deployment for KRD rendering and packaging
- Integration Contracts for component interactions

---

### 06-domain-model-api-reference.md
**Owner**: Platform Core Team (Coordinating)

**Owns**:
- Complete domain entity catalog
- Single definition for each entity across the platform
- Entity ownership assignments (which component owns which entity)
- Entity relationship diagrams
- All API specifications organized by domain:
  - Pipeline APIs
  - Release APIs
  - Deployment APIs
  - Internal Service APIs
- Event schemas
- Error response formats
- API versioning and evolution

**Does NOT Own**:
- API implementation details (owned by respective component specs)
- Business logic and workflows (owned by respective component specs)
- Entity storage infrastructure (owned by Core Architecture)

**Contributing Components**:
- Each component contributes their entity definitions
- Each component contributes their API specifications
- This document consolidates and deduplicates

**References**:
- All component specifications reference this document for entity definitions
- Integration Contracts references this for entity relationships in integration

---

### 07-integration-contracts.md
**Owner**: Platform Architecture Team

**Owns**:
- Component communication patterns
- Inter-component contracts and interfaces:
  - Pipeline → Artifacts
  - Pipeline → Release
  - Release → GitOps
  - Artifacts → Publishers
- Shared infrastructure usage patterns specific to integration
- Error propagation between components
- Event flow diagrams across component boundaries
- Integration testing contracts
- Backward compatibility requirements

**Does NOT Own**:
- Internal component implementation (owned by respective specs)
- Shared infrastructure design (owned by Core Architecture)
- Domain entity definitions (owned by Domain Model & API Reference)
- Individual component APIs (owned by Domain Model & API Reference)

**References**:
- Core Architecture for infrastructure patterns
- Domain Model & API Reference for entities and APIs involved in integration
- All component specifications for integration points

---

## Ownership Principles

### Single Source of Truth
Every concept, entity, API, or pattern has exactly **one** authoritative location. Other documents may reference but must not redefine.

### Clear Boundaries
- **Core Architecture**: Owns what's shared across multiple components
- **Component Specs**: Own their internal architecture, workflows, and contracts
- **Domain Model**: Owns entity and API definitions (consolidated from all components)
- **Integration Contracts**: Owns the space between components

### Reference Pattern
When content is owned elsewhere:
```markdown
For [concept], see [Owning Document: Section](path/to/document.md#section)
```

### Contribution Pattern
When multiple components contribute to a consolidated document:
- Each component provides their portion
- Owning document ensures consistency and deduplication
- Example: Domain Model & API Reference consolidates entities from all components

---

## Conflict Resolution

If unclear which document should own content, ask:

1. **Is it shared by multiple components?** → Core Architecture
2. **Is it about component interaction?** → Integration Contracts
3. **Is it a domain entity or API?** → Domain Model & API Reference
4. **Is it about configuration?** → Configuration & Discovery
5. **Is it internal to one component?** → That component's specification

---

**Last Updated**: 2025-10-02
