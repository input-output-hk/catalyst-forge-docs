# Cross-Reference Conventions Guide

## Purpose
This document establishes consistent conventions for cross-referencing between architectural documents, ensuring clear navigation and preventing broken links.

---

## Cross-Reference Format

### Basic Reference Format
Use this format when referencing another document or section:

```markdown
For [concept name], see [Document Name: Section Name](relative/path/to/document.md#section-anchor)
```

**Examples**:
```markdown
For OCI image formats, see [Core Architecture: OCI Image Formats](01-core-architecture.md#oci-image-formats)

For pipeline APIs, see [Domain Model & API Reference: Pipeline APIs](06-domain-model-api-reference.md#pipeline-apis)

For publisher configuration, see [Configuration & Discovery: Publisher Config](02-configuration-discovery.md#publisher-configuration)
```

### Internal Section Reference
When referencing a section within the same document:

```markdown
See [Section Name](#section-anchor) below for details.
```

**Example**:
```markdown
See [Release Creation Workflow](#release-creation-workflow) below for details.
```

---

## Ownership Block Format

Every document should include an ownership block at the bottom using this format:

```markdown
---

**Component Owner**: [Component/Team Name]
**Shared Concept**: See [Document Name: Section](path.md#section)
**Integration Contract**: See [Integration Contracts: Section](07-integration-contracts.md#section)
**Last Updated**: YYYY-MM-DD
```

**Example**:
```markdown
---

**Component Owner**: Artifacts & Distribution
**Shared Concept**: See [Core Architecture: OCI Image Formats](01-core-architecture.md#oci-image-formats)
**Integration Contract**: See [Integration Contracts: Pipeline → Artifacts](07-integration-contracts.md#pipeline--artifacts)
**Last Updated**: 2025-10-02
```

### Ownership Block Fields

- **Component Owner**: The team/component responsible for this document
- **Shared Concept** (optional): Link to shared concepts this component uses from Core Architecture
- **Integration Contract** (optional): Link to relevant integration contracts
- **Last Updated**: Date of last significant update (YYYY-MM-DD format)

---

## Reference Types

### 1. Definition Reference
When pointing to the authoritative definition of a concept:

```markdown
**Defined in**: [Document Name: Section](path.md#section)
```

**Example**:
```markdown
**Defined in**: [Core Architecture: Secret Management Patterns](01-core-architecture.md#secret-management-patterns)
```

### 2. See Also Reference
When suggesting related content:

```markdown
**See also**: [Document Name: Section](path.md#section)
```

**Example**:
```markdown
**See also**: [Integration Contracts: Error Propagation Patterns](07-integration-contracts.md#error-propagation-patterns)
```

### 3. Detailed Reference
When directing to detailed implementation information:

```markdown
For implementation details, see [Document Name: Section](path.md#section)
```

**Example**:
```markdown
For implementation details, see [Build & Distribution: Earthly Producer](04-build-distribution.md#earthly-producer)
```

### 4. Multiple References
When referencing multiple related documents:

```markdown
Related documentation:
- [Document 1: Section](path1.md#section)
- [Document 2: Section](path2.md#section)
- [Document 3: Section](path3.md#section)
```

**Example**:
```markdown
Related documentation:
- [Core Architecture: OCI Image Formats](01-core-architecture.md#oci-image-formats)
- [Build & Distribution: Artifact Tracking OCI Format](04-build-distribution.md#artifact-tracking-oci-format)
- [Release & Deployment: Release OCI Format](05-release-deployment.md#release-oci-format)
```

---

## Section Anchor Conventions

### Creating Anchors
Markdown automatically creates anchors from headers using lowercase with hyphens:

- Header: `## OCI Image Formats`
- Anchor: `#oci-image-formats`

- Header: `### Producer/Publisher Interaction Model`
- Anchor: `#producerpublisher-interaction-model`

### Special Characters in Anchors
- Spaces → hyphens (`-`)
- Forward slashes → removed or replaced
- Parentheses → removed
- Apostrophes → removed

**Examples**:
- `## Pipeline → Artifacts` → `#pipeline--artifacts`
- `## Worker Pool's Architecture` → `#worker-pools-architecture`
- `## DAG (Directed Acyclic Graph)` → `#dag-directed-acyclic-graph`

---

## Bidirectional References

### Principle
References should be bidirectional where appropriate. If Document A references Document B, consider whether Document B should reference back to Document A.

**Example**:

In `01-core-architecture.md`:
```markdown
## OCI Image Formats

The platform uses two primary OCI formats:
1. **Artifact Tracking Format** - See [Build & Distribution: Artifact Tracking OCI Format](04-build-distribution.md#artifact-tracking-oci-format)
2. **Release Format** - See [Release & Deployment: Release OCI Format](05-release-deployment.md#release-oci-format)
```

In `04-build-distribution.md`:
```markdown
### Artifact Tracking OCI Format
**Format Owner**: Build & Distribution

This format implements the OCI standards defined in [Core Architecture: OCI Image Formats](01-core-architecture.md#oci-image-formats).
```

---

## Integration Points Section

Every document should include an "Integration Points" section that references:
1. Shared concepts from Core Architecture
2. Relevant integration contracts
3. Domain entities and APIs from Domain Model & API Reference

**Template**:
```markdown
## Integration Points

### Shared Infrastructure
See [Core Architecture: Section](01-core-architecture.md#section)

### Component Interactions
See [Integration Contracts: Section](07-integration-contracts.md#section)

### Domain Entities
See [Domain Model & API Reference: Section](06-domain-model-api-reference.md#section)
```

**Example**:
```markdown
## Integration Points

### Shared Infrastructure
For OCI format standards, see [Core Architecture: OCI Image Formats](01-core-architecture.md#oci-image-formats)

For GitOps structure, see [Core Architecture: GitOps Repository Structure](01-core-architecture.md#gitops-repository-structure)

### Component Interactions
For pipeline integration, see [Integration Contracts: Pipeline → Release](07-integration-contracts.md#pipeline--release)

For GitOps integration, see [Integration Contracts: Release → GitOps](07-integration-contracts.md#release--gitops)

### Domain Entities
For release entities and APIs, see [Domain Model & API Reference: Release APIs](06-domain-model-api-reference.md#release-apis)
```

---

## Avoiding Broken Links

### Best Practices
1. **Use relative paths** from the current document location
2. **Test anchors** by verifying the actual header text
3. **Update references** when moving or renaming sections
4. **Document renames**: Update all references when renaming sections

### Link Validation Checklist
- [ ] Path is relative to current document
- [ ] Anchor matches actual header (lowercase, hyphens)
- [ ] Document exists at referenced path
- [ ] Section exists in referenced document

---

## Reference Maintenance

### When Creating New Sections
1. Choose clear, stable section names
2. Avoid special characters in headers
3. Consider who will reference this section
4. Add bidirectional references where appropriate

### When Moving Content
1. Identify all incoming references to the content
2. Update all references to new location
3. Consider adding redirect note in old location temporarily
4. Update ownership blocks if ownership changes

### When Deprecating Content
1. Add deprecation notice with reference to replacement
2. Keep deprecated content temporarily with clear notice
3. Update all references to point to new content
4. Remove deprecated content in next major update

---

## Examples by Use Case

### Use Case 1: Component Referencing Shared Concept
In `03-execution-orchestration.md`:
```markdown
## AWS Infrastructure

The execution system uses AWS services following the patterns defined in
[Core Architecture: Shared Infrastructure Patterns](01-core-architecture.md#shared-infrastructure-patterns).

Specifically:
- **SQS**: For job queue management
- **DynamoDB**: For status tracking
- **S3**: For artifact storage

For authentication to AWS services, see [Core Architecture: Authentication & Authorization Model](01-core-architecture.md#authentication--authorization-model).
```

### Use Case 2: Referencing API Definition
In `05-release-deployment.md`:
```markdown
## Release Creation Workflow

The release creation process invokes several APIs:

- **Artifact API**: For fetching artifact metadata - see [Domain Model & API Reference: Artifact APIs](06-domain-model-api-reference.md#artifact-apis)
- **Rendering API**: For resource rendering - see [Domain Model & API Reference: Internal Service APIs](06-domain-model-api-reference.md#internal-service-apis)
```

### Use Case 3: Referencing Integration Contract
In `04-build-distribution.md`:
```markdown
## Producer/Publisher Interaction Model

Publishers are invoked by the pipeline following the contract defined in
[Integration Contracts: Pipeline → Artifacts](07-integration-contracts.md#pipeline--artifacts).
```

### Use Case 4: Multiple Related References
In `02-configuration-discovery.md`:
```markdown
## KRD Generation Process

KRD generation involves multiple system components:

**Related documentation**:
- [Core Architecture: GitOps Repository Structure](01-core-architecture.md#gitops-repository-structure) - Target repository structure
- [Integration Contracts: Configuration → Pipeline](07-integration-contracts.md#configuration--pipeline) - How KRDs are consumed
- [Domain Model & API Reference: Project Entity](06-domain-model-api-reference.md#project-entity) - Project entity definition
```

---

## Quick Reference

### Most Common References

| When you need... | Reference this... |
|-----------------|-------------------|
| OCI format standards | [Core Architecture: OCI Image Formats](01-core-architecture.md#oci-image-formats) |
| GitOps structure | [Core Architecture: GitOps Repository Structure](01-core-architecture.md#gitops-repository-structure) |
| Secret management | [Core Architecture: Secret Management Patterns](01-core-architecture.md#secret-management-patterns) |
| Authentication | [Core Architecture: Authentication & Authorization Model](01-core-architecture.md#authentication--authorization-model) |
| Infrastructure abstractions | [Infrastructure Abstractions: XRD Catalog](08-infrastructure-abstractions.md#xrd-catalog) |
| Environment configuration | [Infrastructure Abstractions: Environment Configuration Model](08-infrastructure-abstractions.md#environment-configuration-model) |
| Resource references | [Infrastructure Abstractions: Universal Reference Pattern](08-infrastructure-abstractions.md#universal-reference-pattern) |
| Component interactions | [Integration Contracts](07-integration-contracts.md) |
| Entity definitions | [Domain Model & API Reference: Domain Entity Catalog](06-domain-model-api-reference.md#domain-entity-catalog) |
| API specifications | [Domain Model & API Reference: API Specifications](06-domain-model-api-reference.md#api-specifications-by-domain) |

---

**Last Updated**: 2025-10-02
