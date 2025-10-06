# Catalyst Forge Architecture Documentation

## Table of Contents

- [Document Structure](#document-structure)
- [Supporting Documents](#supporting-documents)
- [Reading Paths](#reading-paths)
  - [For Architects and Technical Leadership](#for-architects-and-technical-leadership)
  - [For Platform Engineers](#for-platform-engineers)
  - [For Application Developers](#for-application-developers)
  - [For API Integration](#for-api-integration)
- [Core Architectural Principles](#core-architectural-principles)
- [Technology Stack](#technology-stack)
- [Architecture Characteristics](#architecture-characteristics)
- [Document Conventions](#document-conventions)

---

This directory contains the complete architecture documentation for the Catalyst Forge platform—an internal developer platform that manages the software development lifecycle from commit to deployment.

---

## Document Structure

The architecture is documented across seven focused documents with clear boundaries:

### [01. Architecture Overview](01-architecture-overview.md)
**Executive-level system understanding**

Provides high-level architecture, core principles, and system scope. Introduces the ports and adapters pattern that defines the platform's approach to infrastructure portability.

**Read this first** to understand what Catalyst Forge is and how it works at a conceptual level.

**Key Topics:**
- System scope and philosophy
- Ports and adapters architecture
- Component architecture
- Service architecture
- Architectural principles

---

### [02. Platform Contracts](02-platform-contracts.md)
**Infrastructure contract specifications**

Defines all 11 infrastructure contracts (ports) that the platform depends on, independent of any specific implementation. Each contract specifies required capabilities, interface specifications, and behavioral requirements.

**Read this** to understand the platform's infrastructure dependencies and how the ports and adapters pattern enables deployment across different environments.

**Key Topics:**
- Ports and adapters pattern
- Contract specifications (Object Storage, Container Registry, Secret Management, DNS, Message Queue, Cluster Provisioning, Observability, Authentication, Workload Scaling, Database, Network)
- Contract classification (Platform-Provided vs External Required)
- Adapter implementations by profile

---

### [03. System Architecture](03-system-architecture.md)
**Technical architecture and service design**

Documents the platform's service architecture, component interactions, data flows, and integration patterns. Describes how the three services (Platform API, Worker Service, Dispatcher) collaborate to execute pipelines and manage deployments.

**Read this** to understand how the platform is built internally and how components communicate.

**Key Topics:**
- Service architecture (API, Worker, Dispatcher)
- Component architecture
- Data flow patterns (pipeline execution, GitOps deployment)
- OCI image formats
- GitOps repository structure
- Integration points

---

### [04. Domain Model](04-domain-model.md)
**Business entities and relationships**

Defines all platform entities, their relationships, and database schemas. Organized by component ownership (Pipeline Component, Release Component, Artifacts Component).

**Read this** as a reference for understanding platform entities and their lifecycle.

**Key Topics:**
- Pipeline entities (PipelineRun, PhaseExecution, TaskExecution, StepExecution)
- Release entities (Release, ReleaseAlias, Deployment, ReleaseApproval)
- Artifact entities (ReleaseArtifact)
- Entity relationships
- Database schemas

---

### [05. Implementation Guide](05-implementation-guide.md)
**Deployment profiles and infrastructure abstractions**

Explains how the platform is implemented across different infrastructure environments through deployment profiles. Documents the self-hosting model, bootstrapping sequence, and infrastructure abstractions (Crossplane XRDs).

**Read this** to understand how to deploy the platform and how deployment profiles work.

**Key Topics:**
- Platform deployment architecture
- Bootstrapping sequence
- Production/AWS profile
- On-Premises profile
- Infrastructure abstractions (XRD catalog)
- Environment configuration model
- Reference resolution pattern

---

### [06. Developer Guide](06-developer-guide.md)
**How developers use the platform**

Explains how development teams use the platform to build, release, and deploy applications. Covers repository setup, configuration patterns, pipeline configuration, artifact management, and deployment specifications.

**Read this** if you're a developer using the platform or integrating a project.

**Key Topics:**
- Repository and project structure
- CUE configuration
- Pipeline configuration
- Artifact configuration
- Deployment configuration (XRDs)
- Release configuration
- Common workflows

---

### [07. API & Operations Reference](07-api-operations-reference.md)
**REST API specifications and operational procedures**

Complete API specifications for all platform endpoints (pipelines, releases, deployments, approvals) and operational procedures for result handling, log storage, and monitoring.

**Read this** as a reference for API integration and operational procedures.

**Key Topics:**
- API conventions
- Pipeline APIs
- Release APIs
- Deployment APIs
- Approval APIs
- Event schemas
- Operational procedures

---

## Supporting Documents

### [GLOSSARY.md](GLOSSARY.md)
Comprehensive glossary of terms and concepts used throughout the architecture documentation.

---

## Reading Paths

### For Architects and Technical Leadership
1. Architecture Overview (01)
2. Platform Contracts (02)
3. System Architecture (03)
4. Implementation Guide (05)

### For Platform Engineers
1. Architecture Overview (01)
2. Platform Contracts (02)
3. Implementation Guide (05)
4. System Architecture (03)
5. API & Operations Reference (07)

### For Application Developers
1. Architecture Overview (01)
2. Developer Guide (06)
3. Domain Model (04) - as reference
4. API & Operations Reference (07) - as reference

### For API Integration
1. Architecture Overview (01) - context
2. Domain Model (04)
3. API & Operations Reference (07)

---

## Core Architectural Principles

The platform is built on several foundational principles:

**Ports and Adapters Pattern**
Infrastructure dependencies are expressed as contracts (ports) with multiple implementations (adapters), enabling deployment on AWS-managed services or self-hosted alternatives without changing application code.

**Immutability**
Releases represent immutable snapshots at specific commits. Once created, they cannot be modified—only superseded or redeployed.

**Environment Agnosticism**
Releases contain no environment-specific values, enabling deployment to any environment through EnvironmentConfig.

**GitOps**
All deployments managed declaratively through Git as single source of truth.

**Fail-Fast**
Pipeline execution terminates immediately on any failure, maintaining continuous integration quality.

**Progressive Disclosure**
Projects start simple and adopt platform capabilities incrementally as requirements evolve.

---

## Technology Stack

**Configuration:** CUE for type-safe configuration

**Build Execution:** Earthly for reproducible builds

**Orchestration:** Argo Workflows for pipeline execution, Argo CD for GitOps delivery

**Infrastructure Abstraction:** Crossplane for Kubernetes resource composition

**Messaging:** NATS JetStream for ephemeral work distribution

**Persistence:** PostgreSQL for state management and audit trails

**Authentication:** Keycloak for identity and access management

**Service Mesh:** Istio Ambient Mode for zero-trust networking

**Observability:** Grafana Alloy for metrics and log collection

---

## Architecture Characteristics

**Target Scale:** 15-20 developers with capacity for growth

**Deployment Models:**
- Production/AWS: AWS-managed services with self-hosted platform components
- On-Premises: Self-hosted open-source implementations

**Service Architecture:** 3 core services (Platform API, Worker Service, Dispatcher)

**Cluster Topology:** Platform cluster (persistent control plane) managing multiple environment clusters (ephemeral data planes)

---

## Document Conventions

**Cross-References:** References between documents use the format "See [Document]: [Section]"

**Abstraction Level:** Documents focus on architecture and design patterns, not implementation details or operational procedures

**Code Examples:** Minimal illustrative examples only; complete configurations are in implementation repositories

---

**Last Updated:** 2025-10-05
