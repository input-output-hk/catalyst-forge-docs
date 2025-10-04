# Catalyst Forge Architecture Documentation

## Overview

Welcome to the Catalyst Forge architecture documentation. This collection of documents provides comprehensive coverage of the platform's design, components, and integration patterns.

The documentation is organized to support both quick reference and deep dives, with clear ownership boundaries and consistent cross-referencing.

---

## Documentation Structure

### Getting Started Documents

#### [GLOSSARY.md](GLOSSARY.md)
Comprehensive glossary of all terms and acronyms used in the platform documentation.

**Read this when**: You need to look up a term or understand platform-specific terminology.

#### [00-ownership-boundaries.md](00-ownership-boundaries.md)
Defines which document owns which content, preventing duplication and ensuring single source of truth.

**Read this when**: You're unsure where content belongs or which document to consult.

#### [00-cross-reference-guide.md](00-cross-reference-guide.md)
Establishes conventions for referencing between documents.

**Read this when**: Creating cross-references or navigating between documents.

---

### Core Platform Documents

#### [01-core-architecture.md](01-core-architecture.md)
**Owner**: Platform Core

Single source of truth for system-wide concepts and shared patterns.

**Key Topics**:
- Architectural principles and constraints
- Component architecture and dependencies
- Unified data flow (commit → deployment)
- Shared concepts:
  - OCI Image Formats
  - GitOps Repository Structure
  - Secret Management Patterns
  - Authentication & Authorization Model

**Read this when**:
- Starting to learn the platform
- Need to understand cross-cutting concerns
- Working with shared infrastructure

#### [08-infrastructure-abstractions.md](08-infrastructure-abstractions.md)
**Owner**: Platform Core

Defines the Crossplane-based infrastructure abstractions for platform deployments.

**Key Topics**:
- XRD catalog and design philosophy
- Environment configuration model (two-tier EnvironmentConfig)
- Universal reference pattern for resources
- Namespace strategy and deployment resolution
- Integration with External Secrets, External DNS, and Envoy Gateway

**Read this when**:
- Configuring deployment resources for projects
- Understanding how XRDs abstract Kubernetes complexity
- Working with environment-specific configuration
- Designing new infrastructure abstractions

---

### Component Specifications

#### [02-configuration-discovery.md](02-configuration-discovery.md)
**Owner**: Configuration System

Complete reference for configuration and discovery mechanisms.

**Key Topics**:
- CUE configuration system
- Repository and project schemas
- Discovery mechanisms and rules
- KRD generation process

**Read this when**:
- Configuring repositories or projects
- Understanding how projects are discovered
- Working with CUE schemas

#### [03-execution-orchestration.md](03-execution-orchestration.md)
**Owner**: Pipeline & Execution

Defines how work gets executed in the platform.

**Key Topics**:
- Execution philosophy and model
- Pipeline architecture
- Argo Workflows integration
- Worker pool architecture
- AWS infrastructure (SQS, DynamoDB, S3)
- Job distribution and status tracking

**Read this when**:
- Understanding pipeline execution
- Working with Argo Workflows
- Debugging execution issues
- Optimizing performance

#### [04-build-distribution.md](04-build-distribution.md)
**Owner**: Artifacts & Distribution

Defines artifact creation and publishing contracts.

**Key Topics**:
- Artifact types and definitions
- Producer contract (Earthly implementation)
- Publisher contract and registry types
- Artifact Tracking OCI Format
- Producer/publisher interaction model

**Read this when**:
- Working with artifacts
- Implementing or configuring publishers
- Understanding build processes
- Integrating new registry types

#### [05-release-deployment.md](05-release-deployment.md)
**Owner**: Release & Deployment

Defines release creation and environment progression.

**Key Topics**:
- Release philosophy and concepts
- Release identification and aliasing
- Release creation workflow
- Release OCI Format
- GitOps integration (pointer files, Argo CD plugin)
- Environment promotion and rollback

**Read this when**:
- Understanding release processes
- Working with deployments
- Configuring GitOps integration
- Managing environment promotion

---

### Reference Documents

#### [06-domain-model-api-reference.md](06-domain-model-api-reference.md)
**Owner**: Platform Core (Coordinating)

Single reference for all domain entities and APIs.

**Key Topics**:
- Domain entity catalog
- Entity ownership assignments
- Entity relationship diagrams
- API specifications (Pipeline, Release, Deployment, Internal)
- Event schemas
- Error response formats

**Read this when**:
- Looking up entity definitions
- Understanding entity relationships
- Finding API specifications
- Working with events or errors

#### [07-integration-contracts.md](07-integration-contracts.md)
**Owner**: Platform Architecture

Defines all component interactions and integration patterns.

**Key Topics**:
- Component communication patterns
- Inter-component contracts:
  - Pipeline → Artifacts
  - Pipeline → Release
  - Release → GitOps
  - Artifacts → Publishers
- Shared infrastructure usage
- Error propagation patterns
- Event flow diagrams

**Read this when**:
- Understanding how components interact
- Debugging integration issues
- Implementing new integrations
- Understanding data flow between components

---

## Navigation Guide

### By Role

#### New Developer
**Recommended Reading Order**:
1. [Core Architecture](01-core-architecture.md) - Understand the platform
2. [Configuration & Discovery](02-configuration-discovery.md) - Learn how projects work
3. [Execution & Orchestration](03-execution-orchestration.md) - Understand pipeline execution
4. [Infrastructure Abstractions](08-infrastructure-abstractions.md) - Learn deployment abstractions
5. [Domain Model & API Reference](06-domain-model-api-reference.md) - Learn the domain model

#### Platform Operator
**Recommended Reading Order**:
1. [Core Architecture](01-core-architecture.md) - Understand shared infrastructure
2. [Execution & Orchestration](03-execution-orchestration.md) - Operational aspects
3. [Integration Contracts](07-integration-contracts.md) - Component interactions
4. [Release & Deployment](05-release-deployment.md) - Deployment workflows

#### Component Developer
**Recommended Reading Order**:
1. [Core Architecture](01-core-architecture.md) - Shared concepts
2. Your component's specification (02, 03, 04, or 05)
3. [Integration Contracts](07-integration-contracts.md) - How to integrate
4. [Domain Model & API Reference](06-domain-model-api-reference.md) - Entities and APIs

### By Task

#### Configuring a New Project
1. [Configuration & Discovery: Project Configuration Schema](02-configuration-discovery.md#project-configuration-schema)
2. [Core Architecture: GitOps Repository Structure](01-core-architecture.md#gitops-repository-structure)
3. [Configuration & Discovery: Complete Configuration Examples](02-configuration-discovery.md#complete-configuration-examples)

#### Understanding Pipeline Execution
1. [Execution & Orchestration: Execution Philosophy](03-execution-orchestration.md#execution-philosophy--model)
2. [Execution & Orchestration: Pipeline Architecture](03-execution-orchestration.md#pipeline-architecture)
3. [Integration Contracts: Pipeline → Artifacts](07-integration-contracts.md#pipeline--artifacts)
4. [Integration Contracts: Pipeline → Release](07-integration-contracts.md#pipeline--release)

#### Setting Up Artifact Publishing
1. [Build & Distribution: Publisher Contract](04-build-distribution.md#publisher-contract--registry-types)
2. [Configuration & Discovery: Publisher Config](02-configuration-discovery.md#publisher-configuration)
3. [Build & Distribution: Examples](04-build-distribution.md#examples)

#### Deploying a Release
1. [Release & Deployment: Release Creation Workflow](05-release-deployment.md#release-creation-workflow)
2. [Release & Deployment: GitOps Integration](05-release-deployment.md#gitops-integration)
3. [Release & Deployment: Environment Promotion](05-release-deployment.md#environment-promotion)

#### Configuring Deployment Resources
1. [Infrastructure Abstractions: XRD Catalog](08-infrastructure-abstractions.md#xrd-catalog)
2. [Infrastructure Abstractions: Configuration Example](08-infrastructure-abstractions.md#configuration-example)
3. [Configuration & Discovery: Deployment Configuration](02-configuration-discovery.md#project-configuration-schema)

#### Looking Up an API
1. [Domain Model & API Reference: API Specifications by Domain](06-domain-model-api-reference.md#api-specifications-by-domain)
2. Navigate to specific API section (Pipeline, Release, Deployment, etc.)

#### Understanding Component Integration
1. [Integration Contracts](07-integration-contracts.md) - Find your component interaction
2. [Core Architecture: Component Architecture](01-core-architecture.md#component-architecture--dependencies)
3. [Domain Model & API Reference](06-domain-model-api-reference.md) - Relevant entities

---

## Quick Reference

### Common Concepts

| Concept | Location |
|---------|----------|
| OCI Image Formats | [Core Architecture: OCI Image Formats](01-core-architecture.md#oci-image-formats) |
| GitOps Structure | [Core Architecture: GitOps Repository Structure](01-core-architecture.md#gitops-repository-structure) |
| Secret Management | [Core Architecture: Secret Management Patterns](01-core-architecture.md#secret-management-patterns) |
| Authentication | [Core Architecture: Authentication & Authorization Model](01-core-architecture.md#authentication--authorization-model) |
| Configuration Schema | [Configuration & Discovery: CUE Configuration System](02-configuration-discovery.md#cue-configuration-system) |
| Pipeline Execution | [Execution & Orchestration: Pipeline Architecture](03-execution-orchestration.md#pipeline-architecture) |
| Artifact Publishing | [Build & Distribution: Publisher Contract](04-build-distribution.md#publisher-contract--registry-types) |
| Release Process | [Release & Deployment: Release Creation Workflow](05-release-deployment.md#release-creation-workflow) |
| Infrastructure Abstractions | [Infrastructure Abstractions: XRD Catalog](08-infrastructure-abstractions.md#xrd-catalog) |
| Domain Entities | [Domain Model & API Reference: Domain Entity Catalog](06-domain-model-api-reference.md#domain-entity-catalog) |
| Component Interactions | [Integration Contracts](07-integration-contracts.md) |

---

## Documentation Principles

### Single Source of Truth
Every concept has exactly one authoritative location. Other documents reference but do not redefine.

### Clear Ownership
Each document has a clear owner and defined scope. See [Ownership Boundaries](00-ownership-boundaries.md).

### Progressive Disclosure
Documents support both quick reference and deep dives with consistent structure.

### Explicit Boundaries
Component interactions happen through defined contracts. Internal details stay within component specs.

### Practical Examples
Major concepts include real-world examples from actual usage patterns.

---

## Contributing to Documentation

### Before Adding Content

1. **Check ownership**: See [Ownership Boundaries](00-ownership-boundaries.md) to determine which document should own the content
2. **Check for duplication**: Search existing documents to ensure concept doesn't already exist
3. **Follow conventions**: Use the cross-reference format from [Cross-Reference Guide](00-cross-reference-guide.md)

### When Updating Content

1. **Update "Last Updated" date** in ownership block
2. **Update bidirectional references** if changing section names
3. **Validate links** after moving or renaming sections
4. **Maintain standard structure** per document template

### Document Template

See [MIGRATE.md: Appendix - Document Templates](MIGRATE.md#appendix-document-templates) for the standard section structure.

---

## Migration Status

This documentation structure is currently under migration from the original organic structure.

**Migration Plan**: See [MIGRATE.md](MIGRATE.md)
**Progress Tracking**: See [TODO.md](TODO.md)

**Current Phase**: Phase 6 - Review & Polish (In Progress)
**Completed**: Phases 1-5 complete; Technical review and editorial review complete
**Next Phase**: Final Validation

---

## Feedback and Questions

If you find:
- Broken links or references
- Duplicate content
- Missing information
- Unclear ownership

Please consult [Ownership Boundaries](00-ownership-boundaries.md) or reach out to the Platform Architecture team.

---

**Last Updated**: 2025-10-02
