# Build & Distribution Specification

## Executive Summary
Defines artifact creation and publishing contracts, including producer implementations (Earthly), publisher types (Docker, PyPI, GitHub, etc.), and the artifact tracking OCI format.

## Core Concepts

### Core Philosophy

#### Separation of Concerns

The component enforces strict boundaries between artifact creation and distribution. Producers focus solely on building artifacts from source, while publishers handle registry-specific distribution logic. This separation enables the platform to support diverse build systems and registry types without coupling them together.

#### Idempotent Operations

Releases are designed to be fully idempotent. Running a release multiple times with the same inputs produces the same outputs without creating duplicates. Publishers check for existing artifacts before publishing, and the platform enforces uniqueness at the release level through commit-based identification.

#### Universal Provenance

Every artifact, regardless of type, receives an OCI tracking image that serves as its permanent provenance record. This tracking image contains metadata, build information, published locations, and security attestations, providing a complete audit trail that outlives the artifacts themselves.

### Artifact Types & Definitions

The platform supports three fundamental artifact types that cover all distribution needs:

**Container** - Docker/OCI container images that remain in the Docker daemon during processing to avoid unnecessary serialization. Publishers load images directly from the daemon for registry distribution.

**Binary** - Standalone executables for direct execution. Binaries include architecture and operating system metadata to ensure compatibility. This type is used for CLI tools, compiled applications, and other executable programs.

**Archive** - Compressed file collections in standard formats (tar.gz, zip). Archives serve as the universal type for any file-based distribution including libraries, packages, documentation, and arbitrary file sets. Publishers extract and process archive contents as needed for their specific registry requirements.

### Component Structure

The Artifacts Component consists of three primary subsystems working in concert:

**Artifact Producers** transform source code into built artifacts. The platform ships with an Earthly producer by default, with the architecture supporting additional producers in the future. Producers receive standardized requests and return structured responses, abstracting build system specifics from the platform.

**Artifact Publishers** distribute built artifacts to their destination registries. Each publisher handles the specifics of its target registry type, from Docker registries to PyPI repositories. Publishers operate idempotently, checking for existing artifacts before attempting publication.

**Platform Orchestrator** coordinates the entire lifecycle, from reading project configuration through tracking image creation. It manages staging directories, invokes producers and publishers, generates SBOMs, and creates the final OCI tracking images.

## Architecture

### Producer Contract & Implementations

#### Input Structure

The platform provides producers with complete context for artifact creation:

```
CreateArtifactRequest {
    project: string                     // Project name
    project_path: string                // Relative path in repository
    repository: string                  // <owner>/<repo> reference
    repository_path: string             // Local checkout path
    commit_sha: string                  // Git commit SHA
    release_id: string                  // Unique release identifier
    artifact: {                         // Full artifact config from project
        name: string
        type: string
        producer: {                     // Producer configuration
            [producer_type]: {          // e.g., "earthly"
                // Producer-specific fields
            }
        }
        publishers: {
            [string]: {...}             // Publisher configurations
        }
    }
    staging_path: string                // Output directory for artifacts
}
```

#### Output Structure

Producers return standardized responses with type-specific details:

```
CreateArtifactResponse {
    name: string                        // Artifact name
    project: string                     // Project name
    type: string                        // Artifact type
    output: {                           // Type-specific structure
        // Varies by artifact type
    }
    metadata: map[string]any            // Additional producer metadata
}
```

**Note**: The fields in the output structure become directly accessible in the artifact output schema, excluding internal fields like `location_type` and `path` which are used only for platform processing.

#### Type-Specific Output Structures

**Container:**
```
output: {
    location_type: "docker-daemon"
    image_ref: string                   // Full image reference in daemon
    image_id: string                    // Docker image ID
    digest: string                      // Image content digest
    size: int64                         // Image size in bytes
    base_image?: string                 // Base image used
    layers?: int                        // Number of layers
}
```

**Binary:**
```
output: {
    location_type: "file"
    path: string                        // Binary file location in staging
    executable: string                  // Binary filename
    arch: string                        // Target architecture (amd64, arm64, etc.)
    os: string                          // Target operating system (linux, darwin, windows)
    size: int64                         // Binary size in bytes
    digest: string                      // SHA256 of binary
    static: bool                        // Static vs dynamic linking
}
```

**Archive:**
```
output: {
    location_type: "file"
    path: string                        // Archive file location in staging
    format: string                      // Archive format (tar.gz, zip, tar.bz2)
    size: int64                         // Archive size in bytes
    digest: string                      // SHA256 of archive
    entries?: int                       // Number of files in archive
    content_type?: string               // Hint about contents (python-wheel, npm-package, etc.)
}
```

#### Earthly Producer

The platform ships with an Earthly producer as the default implementation. Earthly provides container-based builds with excellent caching and reproducibility.

Earthly-specific configuration in project CUE:
```cue
producer: {
    earthly: {
        target: string            // Required: Earthly target (e.g., "+build")
        args?: [...string]        // Optional: Additional arguments
    }
}
```

#### Future Producer Extensions

The producer interface is designed to support additional build systems as platform needs evolve. New producers will be added based on demonstrated requirements and business needs. The standardized contract ensures new producers integrate seamlessly without platform changes.

### Publisher Contract & Registry Types

#### Input Structure

Publishers receive complete context including Git information and credential references:

```
PublishArtifactRequest {
    project: string                     // Project name
    project_path: string                // Project path
    repository: string                  // Repository reference
    repository_path: string             // Local checkout
    git: {                              // Git context
        commit_sha: string
        tag?: string
        branch?: string
        ref: string
    }
    release_id: string                  // Release identifier
    artifact: CreateArtifactResponse   // Producer output
    repo_config: {                      // Repository-level publisher config
        name: string
        type: string
        // Type-specific fields (registry, bucket, etc.)
        credentials: {                  // Universal secret format
            provider: string
            path: string
            key?: string
        }
    }
    project_config: map[string]any     // Project-level publisher config
}
```

#### Output Structure

Publishers return the published artifact location and metadata:

```
PublishArtifactResponse {
    uri: string                         // Full URI to published artifact
    metadata: map[string]any            // Publisher-specific metadata
}
```

**Note**: The metadata fields returned by publishers become accessible in the artifact output schema under the `publishers.<publisher-name>` path. Publishers should return all relevant fields that users might need to reference in their deployment configurations.

#### Publisher Types

The platform ships with publishers for the following registry types:

**docker** - Publishes container images to Docker-compatible registries (Docker Hub, ECR, GCR, etc.)

**docs** - Publishes static files and documentation to S3 buckets for web hosting

**github** - Publishes binaries as GitHub release assets

**crate** - Publishes Rust libraries to crates.io

**pypi** - Publishes Python packages and tools to PyPI or compatible indices

**cue** - Publishes CUE modules to OCI registries for module distribution

Each publisher handles the specific protocols and requirements of its target registry type, including authentication methods, metadata formats, and validation rules.

### Artifact Tracking OCI Format

For complete OCI tracking image format specifications, see [Core Architecture: OCI Image Formats - Artifact OCI Tracking Images](01-core-architecture.md#artifact-oci-tracking-images).

**Summary:**

**Image Reference Format:**
```
[tracking-registry]/[repository]/[project]/[artifact]:sha-[commit_sha]
```

**Contains:**
- Metadata layer with artifact and build information
- SBOM layer with software bill of materials
- Attestation layer with signatures and provenance

**Security:**
- Tracking images are signed using Cosign for cryptographic attestation
- Platform generates SBOMs automatically for all artifacts
- Provides software composition analysis without producer involvement

### Producer/Publisher Interaction Model

The artifact lifecycle follows a clear progression from configuration to publication:

1. **Configuration Loading** - Platform reads artifact definitions from project configuration
2. **Staging Preparation** - Creates isolated staging directories for each project
3. **Artifact Production** - Invokes producers for each declared artifact
4. **Post-Processing** - Generates SBOMs and additional metadata
5. **Publishing** - Executes publishers in sequence, checking for existing artifacts
6. **Tracking Creation** - Builds and signs OCI tracking image with complete provenance
7. **Database Recording** - Persists artifact records with tracking references
8. **Cleanup** - Removes staging directories

#### Idempotency Enforcement

**Release Level** - The combination of project, artifact, and commit SHA uniquely identifies a release. Attempting to create a duplicate release reruns the existing one rather than creating a new entry.

**Publisher Level** - Each publisher checks whether its artifact already exists in the target registry. If found with the correct digest, the publisher skips publication and returns the existing URI.

**Retry Mechanism** - Failed releases can be retried using `forge release retry <release-id>`. The platform rebuilds artifacts fresh (no caching) and reattempts publication for all configured publishers.

## Configuration

### Project Configuration

For complete artifact configuration schema, see [Configuration & Discovery: Project Configuration Schema - Artifact Configuration](02-configuration-discovery.md#project-configuration-schema).

Projects declare artifacts as named entities with:
- Artifact type (container, binary, archive)
- Single producer configuration (Earthly initially)
- Multiple publisher configurations referencing repository-defined publishers

### Repository Configuration

For complete repository-level publisher configuration schema, see [Configuration & Discovery: Repository Configuration Schema - Publisher Configuration](02-configuration-discovery.md#repository-configuration-schema).

Repositories define available publishers that projects can reference. Each publisher type has its own schema for registry-specific settings and credentials.

### Artifact Output Schema

The platform assembles artifact outputs from producers and publishers into a structured format available during release rendering. **The format of available outputs depends on the artifact producer and publishers used.** Each producer and publisher declares its own output fields, which the platform combines into a unified structure.

#### How Output Schema Works

1. **Producers declare outputs**: Each producer (e.g., Earthly) returns specific fields based on artifact type
2. **Publishers declare outputs**: Each publisher (e.g., Docker, PyPI) returns fields specific to their registry type
3. **Platform combines outputs**: The platform merges these into a single structure available at render time
4. **CUE references outputs**: Use `@artifact("artifact-name.field.path")` to access any declared output

#### Output Structure

```
// Available at release time for @artifact() references
"<artifact-name>": {
    // Producer-specific outputs (varies by artifact type and producer)
    // Excludes internal-only fields like location_type and path
    <producer_output_fields>

    // Publisher outputs (keyed by publisher name)
    publishers: {
        "<publisher-name>": {
            // Publisher-specific outputs (varies by publisher type)
            <publisher_output_fields>
        }
    }
}
```

**Note**: Internal fields used only for platform processing (such as `location_type` and `path` in producer outputs) are not exposed in the artifact output schema.

#### Example Producer Output Schemas

The following are example outputs from the default Earthly producer:

**Container Producer Outputs:**
- `digest`: Content digest of the image
- `size`: Image size in bytes
- `image_id`: Docker image ID
- `image_ref`: Local daemon reference
- `layers`: Number of layers
- `base_image`: Base image used (if available)

**Binary Producer Outputs:**
- `digest`: SHA256 of the binary
- `size`: Binary size in bytes
- `executable`: Binary filename
- `arch`: Target architecture
- `os`: Target operating system
- `static`: Static vs dynamic linking

**Archive Producer Outputs:**
- `digest`: SHA256 of the archive
- `size`: Archive size in bytes
- `format`: Archive format (tar.gz, zip, etc.)
- `entries`: Number of files in archive
- `content_type`: Hint about contents (if available)

#### Example Publisher Output Schemas

The following are example outputs from platform-provided publishers:

**Docker Publisher Outputs:**
- `uri`: Full image URI in registry
- `registry`: Registry hostname
- `namespace`: Registry namespace
- `repository`: Repository name
- `tag`: Image tag
- `digest`: Registry digest (may differ from producer digest)

**GitHub Publisher Outputs:**
- `uri`: Asset download URL
- `release_url`: GitHub release page URL
- `asset_name`: Name of the uploaded asset
- `download_url`: Direct download link

**PyPI Publisher Outputs:**
- `uri`: Package URI in index
- `index_url`: PyPI index URL
- `package_name`: Python package name
- `version`: Package version

**S3/Docs Publisher Outputs:**
- `uri`: S3 object URL
- `bucket`: S3 bucket name
- `key`: Object key
- `public_url`: Public HTTP URL (if applicable)

## Operations

### Error Handling & Retry Logic

#### Producer Failures

When a producer fails to create an artifact, the entire release fails immediately. The platform logs detailed error information and cleans up any partial outputs. Developers can fix issues and retry the complete release.

#### Publisher Failures

Publisher failures result in partial release states. The platform tracks which publishers succeeded and which failed. Retrying a partial release rebuilds artifacts and attempts publication only for previously failed or unpublished destinations.

#### Network Failures

Transient network failures during publishing trigger automatic retries with exponential backoff. Persistent failures mark the publisher as failed, allowing manual retry later.

### Security Considerations

#### Credential Management

For complete secret management specifications, see [Core Architecture: Secret Management Patterns](01-core-architecture.md#secret-management-patterns).

Publisher credentials are never resolved or passed as plain text. The platform uses a universal secret reference format that publishers resolve at the moment of use. This prevents raw secrets from traversing the network or residing in memory across component boundaries. Each publisher receives only credential references and resolves them directly when authenticating with registries.

#### Artifact Integrity

All artifacts include checksums for integrity verification. Publishers verify checksums before publication. Tracking images provide tamper-evident records through cryptographic signatures.

#### Supply Chain Security

The platform generates SBOMs following SPDX standards. Cosign signatures provide non-repudiation. Tracking images create permanent audit trails that survive artifact deletion.

The platform reserves the right to enforce additional security measures before artifact publication. This may include vulnerability scanning, policy validation, or compliance checks. Such measures are transparent to producers and publishers, occurring as platform-level quality gates.

## Integration Points

For artifact tracking OCI format standards, see [Core Architecture: OCI Image Formats](01-core-architecture.md#oci-image-formats).

For pipeline integration during release phase, see [Execution & Orchestration: Release Creation](03-execution-orchestration.md#release-creation).

For cross-component integration patterns, see [Integration Contracts](07-integration-contracts.md).

### Database Entity

For complete domain entity specifications, see [Domain Model & API Reference: ReleaseArtifact](06-domain-model-api-reference.md#releaseartifact).

The platform maintains a minimal database entity for artifacts to enable efficient querying, while full metadata resides in OCI tracking images.

## Reference

For complete configuration examples including repository publisher configuration, project artifact configuration, and deployment usage, see [Configuration & Discovery: Complete Configuration Examples](02-configuration-discovery.md#complete-configuration-examples).

## Constraints & Limitations

### Current Limitations

- Single producer type (Earthly) in initial release
- No artifact caching between release attempts
- Maximum artifact size limited by staging disk space
- Manual retry required for partial publisher failures

### Design Boundaries

- Producers never publish directly to registries
- Publishers operate independently without coordination
- Artifacts are immutable once published
- Tracking images cannot be deleted or modified

## Future Enhancements

### Planned Capabilities

**Additional Producers** - The producer interface is designed to support additional build systems as platform needs evolve. New producers will be added based on demonstrated requirements and business needs.

**Vulnerability Scanning** - Integration with security scanning services to analyze artifacts before publication, providing an additional quality gate in the release process.

**Enhanced Publisher Features** - Support for advanced registry features like multi-architecture manifests, attestation bundles, and registry-specific metadata.

### Extension Points

The producer and publisher contracts are designed for extensibility. New artifact types can be added by extending the type enumeration and defining output structures. Additional publishers can be implemented following the defined contract. The platform's architecture ensures new capabilities can be added without disrupting existing functionality.

---

**Component Owner**: Artifacts & Distribution
**Shared Concepts**: See [Core Architecture: OCI Image Formats](01-core-architecture.md#oci-image-formats)
**Last Updated**: 2025-10-02
