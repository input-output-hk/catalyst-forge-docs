# Environment Configuration Design

## Executive Summary

This document details the technical design of our environment-specific value injection system built on Crossplane EnvironmentConfigs. The system enables environment-agnostic releases by deferring all environment-specific configuration to deployment time.

**Core Capabilities:**
- Bottom-up merging for value resolution (composition defaults → cluster defaults → XRD spec → per-project overrides)
- Universal reference pattern for cross-XR dependencies (`outputs/`, `connections/`, `secrets/`, `configs/`)
- Two-tier configuration model (cluster-wide defaults + per-project overrides)
- Deep merge for maps, replacement for arrays and scalars
- Cross-namespace references for public outputs only

**Key Design Decisions:**
- Bottom-up merging simplifies implementation (each layer progressively overrides previous)
- Cluster-scoped EnvironmentConfigs for shared access across namespaces
- Label-based selection decouples resource naming from selection logic
- Separate reference types enforce security model (public vs private data)
- Per-reconciliation caching for performance

**Integration Points:**
- Composition functions query EnvironmentConfig during XR reconciliation
- Reference resolution enables dependency graphs between XRs
- GitOps repository stores EnvironmentConfig source (`<env>/env.yml`)

## Core Concepts

### Environment-Agnostic Releases

**Why Releases Can't Contain Environment Values:**

Releases are OCI images built once and deployed to multiple environments. Environment-specific values (replica counts, resource limits, domain names) cannot be embedded because:

1. **Build-Once Principle**: OCI image built in CI contains only application configuration
2. **Environment Portability**: Same release must work in dev, staging, production
3. **Security**: Secrets and credentials differ per environment, cannot be in image
4. **Operational Flexibility**: Operators need to tune values per environment without rebuilding

**When/Where We Inject Values:**

Value injection occurs at **deployment time** during XR reconciliation:

```
Release Creation (CI)
→ CUE configuration → OCI image (environment-agnostic)
→ Git commit → ReleasePointer updated

Deployment (Argo CD + Crossplane)
→ Argo CD extracts XRs from OCI image
→ Crossplane reconciles XR
→ Composition fetches EnvironmentConfig
→ Values injected based on environment
→ Managed resources created with environment-specific values
```

**Our Deployment-Time Resolution Approach:**

1. **EnvironmentConfig Lookup**: Composition fetches EnvironmentConfig by environment name (from XR label)
2. **Precedence Resolution**: Apply 4-level precedence chain to determine final value for each field
3. **Reference Resolution**: Parse and resolve any references (`outputs/`, `connections/`, etc.)
4. **Patch Application**: Inject resolved values into managed resource templates
5. **Resource Creation**: Apply managed resources to cluster with environment-specific configuration

This approach separates **what to deploy** (in release) from **how to deploy it** (in EnvironmentConfig).

**Note**: For general Crossplane EnvironmentConfig concepts, see [Crossplane Environment Configuration](https://docs.crossplane.io/latest/concepts/environment-configs/)

## Our EnvironmentConfig Structure
### CRD Design

We use Crossplane EnvironmentConfig resources to store environment-specific values that are injected during XR reconciliation. The system uses **two separate EnvironmentConfig resources** per environment:

**1. Cluster-Wide EnvironmentConfig** (`<env>/env.yaml`)
```yaml
apiVersion: apiextensions.crossplane.io/v1alpha1
kind: EnvironmentConfig
metadata:
  name: Cluster
  labels:
    forge.projectcatalyst.io/type: cluster
spec:
  # Environment identification
  environment:
    name: <env-name>
    domain: <env-domain>
    region: <region>

  # Cluster-wide defaults per XRD kind
  defaults:
    Deployment:
      replicas: 3
      resources:
        requests:
          cpu: "100m"
          memory: "128Mi"
    Network:
      tlsEnabled: true
```

**2. Per-Project EnvironmentConfig** (`<env>/apps/<repository>/<project>/env.yaml`)
```yaml
apiVersion: apiextensions.crossplane.io/v1alpha1
kind: EnvironmentConfig
metadata:
  name: <repository>-<project>  # e.g., "acme-services-api"
  labels:
    forge.projectcatalyst.io/type: project
    forge.projectcatalyst.io/project: <project>  # e.g., "api"
spec:
  # Per-XR instance overrides
  overrides:
    <xr-name>:
      <field>: <value>

  # Example:
  # overrides:
  #   api-deployment:
  #     replicas: 10
  #     resources:
  #       requests:
  #         cpu: "500m"
```

**Type System:**
- All values stored as JSON-compatible types (strings, numbers, booleans, objects, arrays)
- Composition functions convert types during patching
- No schema validation at EnvironmentConfig level (validated in compositions)

### Storage Strategy

**Two EnvironmentConfigs Per Environment:**

1. **Cluster-Wide EnvironmentConfig:**
   - Name: `Cluster` (standardized name)
   - Scope: Cluster-scoped (shared across all namespaces)
   - Source: `<env>/env.yaml` in GitOps repository
   - Labels:
     - `forge.projectcatalyst.io/type: cluster`
   - Contains: Environment metadata + cluster-wide defaults

2. **Per-Project EnvironmentConfig:**
   - Name: `<repository>-<project>` (e.g., `acme-services-api`)
   - Scope: Cluster-scoped (but logically associated with project namespace)
   - Source: `<env>/apps/<repository>/<project>/env.yaml` in GitOps repository
   - Labels:
     - `forge.projectcatalyst.io/type: project`
     - `forge.projectcatalyst.io/project: <project>`
   - Contains: Per-XR instance overrides for this project

**Rationale for Cluster-Scoped:**
- Both EnvironmentConfigs are cluster-scoped (not namespaced)
- Allows composition functions to query without namespace context
- Centralized management through GitOps repository
- Single source of truth per environment/project combination

**Access Patterns (Label-Based Selection):**
- Composition functions select cluster-wide config by label: `forge.projectcatalyst.io/type=cluster`
- Composition functions select per-project config by labels: `forge.projectcatalyst.io/type=project,forge.projectcatalyst.io/project=<project>`
- Both lookups happen during XR reconciliation
- Read-only access from compositions (values never modified during reconciliation)

### Data Organization

**Cluster-Wide EnvironmentConfig** (`dev/env.yaml`):
```yaml
apiVersion: apiextensions.crossplane.io/v1alpha1
kind: EnvironmentConfig
metadata:
  name: Cluster
  labels:
    forge.projectcatalyst.io/type: cluster
spec:
  environment:
    name: dev
    domain: dev.us-east-1.projectcatalyst.io
    region: us-east-1
  defaults:
    Deployment:
      replicas: 1  # Lower for dev
    Network:
      tlsEnabled: false  # Disable TLS in dev
```

**Per-Project EnvironmentConfig** (`production/apps/acme-services/api/env.yaml`):
```yaml
apiVersion: apiextensions.crossplane.io/v1alpha1
kind: EnvironmentConfig
metadata:
  name: acme-services-api
  labels:
    forge.projectcatalyst.io/type: project
    forge.projectcatalyst.io/project: api
spec:
  overrides:
    api-deployment:  # Specific XR instance name
      replicas: 10   # Override for this deployment
      resources:
        requests:
          cpu: "500m"
```

**Composition Defaults:**
- Defined in composition functions as hardcoded fallbacks
- Used when no EnvironmentConfig value or XRD spec value exists
- Composition defaults are version-specific (change with composition updates)

## Our Value Resolution Algorithm

### Bottom-Up Merging Strategy

Our platform resolves configuration values using a **bottom-up merging approach** where each layer progressively overrides the previous:

**Layer 1: Composition Defaults** (Base)
- Source: Hardcoded in composition function code
- Purpose: Safe fallbacks that ensure system functions
- Example: `replicas: 1` as ultimate fallback

**Layer 2: Cluster-Wide Defaults** (Environment Policy)
- Source: Cluster-wide EnvironmentConfig (selected by label `type=cluster`)
- Path: `spec.defaults.<xrKind>.<field>`
- Purpose: Environment-level defaults for all XRs of a kind
- Example: All Deployments in production default to 3 replicas

**Layer 3: XRD Spec Values** (Developer Intent)
- Source: XR's `spec.<field>` value
- Purpose: Developer intent declared in CUE configuration
- Example: Developer sets `replicas: 5` in configuration

**Layer 4: Per-Project Overrides** (Operator Override - Highest Priority)
- Source: Per-project EnvironmentConfig (selected by labels `type=project`, `project=<name>`)
- Path: `spec.overrides.<xr-name>.<field>`
- Purpose: Environment-specific tuning for individual XR instances
- Example: Production API deployment needs 10 replicas

**Rationale for Bottom-Up Merging:**
- Simpler mental model: each layer progressively overrides the previous
- No conditional precedence logic: later steps naturally win
- Easier to extend: new layers can be inserted at any point
- Developer intent preserved unless environment requires override
- Safe fallbacks always present at the base

**How Operators Override Developer Intent:**
Platform operators override developer values by updating the per-project EnvironmentConfig:
1. Edit `<env>/apps/<repository>/<project>/env.yaml` in GitOps repository
2. Add/update override for specific XR instance under `spec.overrides.<xr-name>.<field>`
3. Commit and push to Git
4. Argo CD reconciles and applies updated EnvironmentConfig
5. Crossplane re-reconciles affected XRs with new override values

Example: Override replicas for production API despite developer setting:
```yaml
# production/apps/acme-services/api/env.yaml
apiVersion: apiextensions.crossplane.io/v1alpha1
kind: EnvironmentConfig
metadata:
  name: acme-services-api
  labels:
    forge.projectcatalyst.io/type: project
    forge.projectcatalyst.io/project: api
spec:
  overrides:
    api-deployment:
      replicas: 10  # Overrides developer's value in XRD spec
```

### Merging Implementation

**Resolution Algorithm (Bottom-Up Merging):**

The custom composition function receives pre-loaded EnvironmentConfigs from the pipeline and builds final values by progressive merging:

1. **Start with composition default** - Initialize value with hardcoded fallback
2. **Merge cluster-wide defaults** - Apply environment defaults from cluster EnvironmentConfig, overriding composition defaults
3. **Merge XRD spec values** - Apply developer intent from XR spec, overriding environment defaults
4. **Merge per-project overrides** - Apply operator overrides from per-project EnvironmentConfig (if loaded), overriding all previous values

Each merge operation uses the appropriate strategy for the value type:
- **Scalars** (strings, numbers, booleans): Simple replacement
- **Maps/Objects**: Deep merge (keys combined, nested maps recursively merged)
- **Arrays**: Complete replacement

**EnvironmentConfig Availability:**

EnvironmentConfigs are loaded by the `function-environment-configs` pipeline step before the custom function executes:

- **Cluster-wide config** - Always loaded (required - composition fails if not found)
- **Per-project config** - Loaded if exists (optional - nil if not found)

The custom function accesses these pre-loaded configs from the pipeline context without performing any Kubernetes API queries.

**Deep Path Merging:**

For nested fields like `resources.requests.cpu`, the algorithm merges at each level:

1. Start with composition default: `{requests: {cpu: "50m", memory: "64Mi"}}`
2. Merge cluster-wide default: `{requests: {cpu: "100m"}}` → Result: `{requests: {cpu: "100m", memory: "64Mi"}}`
3. Merge XRD spec: `{limits: {cpu: "500m"}}` → Result: `{requests: {cpu: "100m", memory: "64Mi"}, limits: {cpu: "500m"}}`
4. Merge per-project override: `{requests: {memory: "256Mi"}}` → Final: `{requests: {cpu: "100m", memory: "256Mi"}, limits: {cpu: "500m"}}`

This demonstrates how deep merge combines keys from all layers rather than replacing entire objects.

### Matching Strategy

**Project Name Matching:**
- Exact match on namespace name: `<repository>-<project>`
- Namespace format enforced during release creation
- No pattern matching or wildcards supported
- Case-sensitive matching

**Fallback Behavior:**
- Missing per-project override → Check next precedence level
- Missing XRD spec value → Check next precedence level
- Missing cluster default → Use composition default
- Missing composition default → Error (composition must define defaults for required fields)

**Examples:**

*Example 1: Full merge chain*
- Composition default: `replicas: 1`
- Cluster-wide default: `replicas: 3`
- Developer (XRD spec): `replicas: 5`
- Per-project override: Not set
- Result: `5` (developer value wins)

*Example 2: Operator override*
- Composition default: `replicas: 1`
- Cluster-wide default: `replicas: 3`
- Developer (XRD spec): `replicas: 5`
- Per-project override: `replicas: 10`
- Result: `10` (operator override wins)

*Example 3: No developer value*
- Composition default: `replicas: 1`
- Cluster-wide default: `replicas: 3`
- Developer (XRD spec): Not set
- Per-project override: Not set
- Result: `3` (cluster-wide default wins)

*Example 4: Only composition default*
- Composition default: `replicas: 1`
- Cluster-wide default: Not set
- Developer (XRD spec): Not set
- Per-project override: Not set
- Result: `1` (composition default fallback)

## Our Reference Resolution Design

The platform provides a universal reference pattern that allows XRs to access values from other XRs in the system. This enables dependency graphs where one XR consumes outputs from another.

### Reference Types We Support

**`outputs/<xr>/<key>` - Literal Value Injection**
- Returns literal value from target XR's `status.outputs.<key>`
- Value inlined directly into consuming resource
- Treated as public data (safe to log)
- Example: `outputs/api-deployment/serviceUrl` → `"https://api.example.com"`

**`connections/<xr>/<key>` - Connection Secret Reference**
- Returns `secretKeyRef` pointing to target XR's connection-details secret
- Secret created automatically by Crossplane
- Contains sensitive connection information (credentials, endpoints)
- Example: `connections/database/password` → `{secretKeyRef: {name: "db-xyz-conn", key: "password"}}`

**`secrets/<xr>/<secret>/[key]` - Secrets XRD Reference**
- Returns `secretKeyRef` pointing to secret managed by Secrets XRD
- `<xr>` is the Secrets XRD instance name
- `<secret>` is the secret name within that XRD
- Optional `[key]` for specific key in secret
- Example: `secrets/app-secrets/api-key` → `{secretKeyRef: {name: "app-secrets-api-key", key: "value"}}`

**`configs/<xr>/<config>/[key]` - ConfigMaps XRD Reference**
- Returns `configMapKeyRef` pointing to ConfigMap managed by ConfigMaps XRD
- `<xr>` is the ConfigMaps XRD instance name
- `<config>` is the ConfigMap name within that XRD
- Optional `[key]` for specific key in ConfigMap
- Example: `configs/app-config/settings/database.host` → `{configMapKeyRef: {name: "app-config-settings", key: "database.host"}}`

### Parsing Our Syntax

**Detection Algorithm:**

Compositions check if a value string starts with any known reference prefix (`outputs/`, `connections/`, `secrets/`, `configs/`). If match found, the value is treated as a reference and parsed accordingly.

**Parsing Logic:**

1. **Identify reference type** - Match prefix to determine reference type
2. **Check for cross-namespace syntax** - If value contains `::`, split on it to extract namespace prefix
3. **Parse reference components**:
   - For `outputs/` and `connections/`: Split remaining path to extract `<xr-name>/<key>`
   - For `secrets/` and `configs/`: Split remaining path to extract `<xr-name>/<resource-name>/[key]` (key is optional)

**Routing to Resolver:**

Each reference type routes to a specialized resolver function:
- `outputs/` → Output reference resolver (returns literal value)
- `connections/` → Connection reference resolver (returns secretKeyRef)
- `secrets/` → Secret reference resolver (returns secretKeyRef)
- `configs/` → ConfigMap reference resolver (returns configMapKeyRef)

### Cross-Namespace Support

**Cross-Namespace Syntax:**
```
<namespace>::<reference-path>
```

**Supported Reference Types:**
- ✅ `outputs/` - Supports cross-namespace (public data)
  - Example: `acme-web::outputs/api-deployment/serviceUrl`
  - Example: `platform::outputs/keycloak/issuerUrl`
- ❌ `connections/` - Same namespace only (secret security)
- ❌ `secrets/` - Same namespace only (secret security)
- ❌ `configs/` - Same namespace only (ConfigMap locality)

**Namespace Inference Rules:**
1. If reference contains `::`, use specified namespace
2. If reference is `outputs/` without `::`, search current namespace first
3. If not found in current namespace and type is `outputs/`, search `platform` namespace as fallback
4. For `connections/`, `secrets/`, `configs/`: always use current namespace only

**Permission Model:**
- No RBAC enforcement at reference resolution (Crossplane runs with cluster admin)
- Security enforced through namespace restrictions
- Cross-namespace `outputs/` allowed because values are public by design
- Secret and ConfigMap references restricted to prevent unauthorized access

### Error Handling

**Reference Not Found:**

When target XR doesn't exist, composition returns error with reference details (XR name, type, namespace). Composition sets XR status to error state, preventing resource creation until target exists.

**Missing Key in Target:**

When referenced key doesn't exist in target XR's outputs, composition returns error listing:
- Requested key
- Target XR name
- Available keys in target

Composition blocks reconciliation until target publishes the key (eventual consistency model).

**Cross-Namespace Violation:**

When reference attempts cross-namespace access for non-`outputs/` types, composition returns error specifying:
- Reference type attempted
- Full reference path
- Explanation that only `outputs/` supports cross-namespace

Composition fails XR reconciliation with clear error message.

**Error Message Design:**
- Include full reference path for debugging
- List available keys/resources when not found
- Suggest correct namespace for cross-namespace attempts
- All errors logged to XR status conditions for user visibility

## Integration with Our Platform

### Composition Pipeline Access

Our Crossplane compositions use a pipeline where EnvironmentConfig selection happens **before** custom function execution:

**Pipeline Configuration:**

Compositions define a two-step pipeline:

1. **`function-environment-configs`** (Crossplane built-in) - Selects and loads EnvironmentConfigs
2. **Custom composition function** - Applies patches using loaded configs

**EnvironmentConfig Selection (via function-environment-configs):**

The composition pipeline configuration specifies which EnvironmentConfigs to load using label selectors:

```yaml
# In Composition resource
spec:
  pipeline:
    - step: load-environment-configs
      functionRef:
        name: function-environment-configs
      input:
        apiVersion: meta.fn.crossplane.io/v1alpha1
        kind: Input
        environment:
          - type: Selector
            selector:
              matchLabels:
                - key: forge.projectcatalyst.io/type
                  value: cluster
          - type: Selector
            selector:
              matchLabels:
                - key: forge.projectcatalyst.io/type
                  value: project
                - key: forge.projectcatalyst.io/project
                  # Value extracted from XR namespace at runtime
```

The `function-environment-configs` function:
1. Extracts project name from XR namespace
2. Queries EnvironmentConfigs using label selectors
3. Loads matching configs into pipeline context
4. Makes them available to subsequent pipeline steps

**Custom Function Access:**

The custom composition function receives pre-loaded EnvironmentConfigs in its context and applies the bottom-up merging algorithm without needing to query for configs.

**Values Accessible to Compositions:**

From cluster-wide EnvironmentConfig:
- `clusterEnvConfig.Spec.Environment.*` - Environment metadata (name, domain, region)
- `clusterEnvConfig.Spec.Defaults.<xrKind>.*` - Cluster-wide defaults for XR kind

From per-project EnvironmentConfig (if exists):
- `projectEnvConfig.Spec.Overrides.<xr-name>.*` - Per-XR instance overrides

All values read-only (compositions never modify EnvironmentConfig)

**Context Propagation:**

Compositions pass both EnvironmentConfigs through patch context:
- Current XR being reconciled
- Cluster-wide EnvironmentConfig (always present)
- Per-project EnvironmentConfig (may be nil)
- Cache of target XRs for reference resolution

All patch functions receive this context and can access both configs for value resolution.

### Value Injection Timing

Value injection occurs during XR reconciliation in this order:

**1. XR Observation (Read Current State)**
```
Crossplane reads XR from API server
→ Includes developer-specified spec values
→ Includes existing status from previous reconciliation
```

**2. EnvironmentConfig Resolution (Fetch Context)**
```
Composition fetches cluster-wide EnvironmentConfig by environment name
→ Validates cluster-wide EnvironmentConfig exists (required)
→ Attempts to fetch per-project EnvironmentConfig (optional)
→ Loads all environment-specific values into memory
```

**3. Value Resolution (Apply Precedence)**
```
For each field referenced in patches:
→ Apply 4-level precedence algorithm
→ Resolve all references (outputs/, connections/, etc.)
→ Build final value set for patching
```

**4. Resource Patching (Generate Managed Resources)**
```
Composition creates/updates managed resources:
→ Apply resolved values to resource templates
→ Generate Deployments, Services, ConfigMaps, etc.
→ Set owner references to XR
```

**5. Status Update (Publish Outputs)**
```
Composition updates XR status:
→ Set status.outputs.* for values this XR publishes
→ Update connection-details secret for sensitive outputs
→ Record reconciliation status
```

**Patch Application Order:**
All patches applied in declaration order within composition:
1. Base resource patches (create skeleton resources)
2. Field value patches (inject resolved values)
3. Reference patches (inject secretKeyRef/configMapKeyRef)
4. Cross-cutting patches (labels, annotations, owner refs)

**Update Propagation:**
```
EnvironmentConfig change
→ Crossplane detects change (watch on EnvironmentConfigs)
→ Requeues all XRs in affected environment
→ Each XR reconciles with new values
→ Managed resources updated via server-side apply
→ Kubernetes controllers roll out changes (Deployments, etc.)
```

**Reconciliation Frequency:**
- Triggered by XR spec changes (developer updates)
- Triggered by EnvironmentConfig changes (platform operations)
- Triggered by managed resource changes (drift detection)
- Periodic reconciliation every 60 seconds (Crossplane default)

## Our Merge Strategies

### Scalar Values

Scalar values (strings, numbers, booleans) use **simple replacement** - no merging logic:

```yaml
# XRD spec
replicas: 3

# EnvironmentConfig override
projects.acme-api.deployment.replicas: 5

# Result: 5 (override replaces spec value completely)
```

### Maps/Objects

Maps use **deep merge** where keys from both sources are combined:

**Deep Merge Algorithm:**

1. Copy all keys from base map to result
2. For each key in override map:
   - If key exists in both and both values are maps → Recursively deep merge the nested maps
   - If key exists in both but values are not both maps → Override value wins
   - If key only in override → Add to result
3. Return merged result

**Example:**
```yaml
# XRD spec
resources:
  requests:
    cpu: "100m"
    memory: "128Mi"
  limits:
    cpu: "500m"

# EnvironmentConfig override
resources:
  requests:
    memory: "256Mi"  # Override this
  limits:
    memory: "1Gi"    # Add this

# Result (deep merged):
resources:
  requests:
    cpu: "100m"      # From XRD spec
    memory: "256Mi"  # From override
  limits:
    cpu: "500m"      # From XRD spec
    memory: "1Gi"    # From override
```

**Conflict Resolution:**
- Override value always wins when same key exists
- No type checking during merge (override can change type)
- Null/empty values in override remove keys from result

**Key Preservation:**
- All keys from base preserved unless explicitly overridden
- All keys from override added to result
- No automatic key removal (must use null to delete)

### Arrays

Arrays use **replacement strategy** - override completely replaces base, no merging:

```yaml
# XRD spec
env:
  - name: LOG_LEVEL
    value: info
  - name: PORT
    value: "8080"

# EnvironmentConfig override
env:
  - name: LOG_LEVEL
    value: debug

# Result (array replaced):
env:
  - name: LOG_LEVEL
    value: debug
# PORT variable is GONE (not merged)
```

**Rationale for Replacement:**
- Arrays often order-dependent (can't safely merge)
- Element identity unclear (which elements match?)
- Replacement provides predictable behavior
- Developers can use maps instead if merging needed

## Design Decisions

### Why Label-Based Selection

**Rationale for Label Selectors Over Name-Based Lookup:**

1. **Standardized Naming**
   - Cluster-wide config always named `Cluster` (no environment-specific names)
   - Simplifies resource naming across all environments
   - Eliminates name conflicts and ambiguity

2. **Flexible Selection**
   - Compositions select by intent (`type=cluster` vs `type=project`)
   - Project matching via `project` label, not name parsing
   - Easier to extend with additional label-based filters

3. **Decoupled from Naming**
   - Resource names don't need to encode selection logic
   - Labels carry semantic meaning for selection
   - Name changes don't break composition logic

4. **Project Isolation**
   - Per-project config selected by `project` label
   - No namespace inference from resource names
   - Clear separation between cluster and project configs

**Label Schema:**
```yaml
# Cluster-wide EnvironmentConfig
labels:
  forge.projectcatalyst.io/type: cluster

# Per-project EnvironmentConfig
labels:
  forge.projectcatalyst.io/type: project
  forge.projectcatalyst.io/project: <project-name>
```

### Why Cluster-Scoped

**Rationale for Cluster-Scoped EnvironmentConfigs:**

1. **Shared Across Namespaces**
   - Multiple projects (namespaces) in same environment need shared configuration
   - Environment metadata (domain, region) same for all projects
   - Single source of truth prevents configuration drift

2. **Centralized Management**
   - Platform operators manage one EnvironmentConfig per environment
   - GitOps repository has single file per environment (`<env>/env.yml`)
   - Changes propagate to all projects automatically
   - No per-namespace duplication of environment values

3. **Composition Function Simplicity**
   - Compositions fetch by environment name (no namespace scoping needed)
   - Always resolves to single EnvironmentConfig
   - No ambiguity about which config applies

**Trade-offs Accepted:**
- ❌ Projects cannot define their own EnvironmentConfigs (platform-managed only)
- ❌ Requires cluster-admin permissions to create/update
- ❌ Namespace-level RBAC cannot restrict access to EnvironmentConfigs
- ✅ But: Per-project overrides still possible via `projects.<namespace>` section
- ✅ Security enforced at composition level (compositions validate what projects can override)

### Why This Precedence Order

**Design Principle: Most Specific to Least Specific**

1. **Per-Project Override (Level 1)**
   - Most specific: targets exact XR instance in exact project
   - Use case: Production API needs 10 replicas, all others use default
   - Explicit operator decision to override for this specific case

2. **XRD Spec Value (Level 2)**
   - Developer's explicit intent in CUE configuration
   - Use case: Developer knows application needs 5 replicas
   - Honors developer knowledge of application requirements

3. **Cluster-Wide Default (Level 3)**
   - Environment-appropriate defaults for XR kind
   - Use case: All Deployments in prod default to 3 replicas
   - Platform policy for environment tier

4. **Composition Default (Level 4)**
   - Fallback when no other configuration provided
   - Use case: Prevent deployment failures with safe defaults
   - Last resort to ensure system functions

**Why Developer Intent at Level 2 (not Level 1):**
- Platform operators need ability to override for operational reasons
- Example: Incident response requires scaling down all deployments
- Example: Cost optimization requires resource limit enforcement
- Developer intent respected unless platform explicitly overrides

**Why Explicit Over Implicit:**
- Empty/null XRD spec value → Skip to next level (don't use null as value)
- Must explicitly set value to override
- No "undefined" confusion - clear whether value was set

### Why Separate Reference Types

**Design Principle: Intent and Security Through Type System**

**1. Security Model (Public vs Private)**

`outputs/` - Public Data:
- Returns literal values (inlined)
- Safe to log, expose in status, cross-namespace access
- Example: Service URL, public endpoints, version numbers

`connections/`, `secrets/`, `configs/` - Private Data:
- Returns Kubernetes references (secretKeyRef/configMapKeyRef)
- Never inlined (Kubernetes resolves at container runtime)
- Restricted to same namespace (security boundary)
- Example: Passwords, API keys, connection strings

**2. Type Safety Guarantees**

Each reference type has known return type:
- `outputs/` → Always returns JSON-compatible value (string, number, object)
- `connections/` → Always returns secretKeyRef to connection secret
- `secrets/` → Always returns secretKeyRef to named secret
- `configs/` → Always returns configMapKeyRef to named ConfigMap

Compositions can validate and transform based on type:
```go
switch ref.Type {
case "outputs/":
    // Safe to inline
    patch.Value = resolveOutput(ref)
case "secrets/":
    // Must use secretKeyRef
    patch.ValueFrom = &SecretKeyRef{...}
}
```

**3. Intent Clarity**

Reference syntax clearly communicates intent:
- `outputs/database/host` → "I want the public database host value"
- `connections/database/password` → "I want a secret reference to the database password"
- `secrets/app-secrets/api-key` → "I want a secret reference to a specific app secret"
- `configs/app-config/feature-flags` → "I want a ConfigMap reference to feature flags"

Developer reading configuration immediately understands:
- What data is being accessed
- Whether it's public or sensitive
- How it will be injected (literal vs reference)

## Platform-Specific Patterns

### Our Naming Conventions

**EnvironmentConfig Naming:**
- Cluster-wide: Always named `Cluster` (standardized across all environments)
- Per-project: Named `<repository>-<project>` (e.g., `acme-services-api`)
- Selected by labels, not by name

**Project Identifier Format:**
- Namespace name: `<repository>-<project>`
- Repository: GitHub repository name (e.g., `acme-services`)
- Project: CUE package name from repository (e.g., `api`)
- Example: `acme-services-api`
- Used in EnvironmentConfig: `spec.data.projects.<repository>-<project>`

**XR Name Expectations:**
- XR name specified in CUE: `name: "api-deployment"`
- Must be unique within namespace
- Used in per-project overrides: `projects.<ns>.<xr-name>.*`
- Lowercase with hyphens (Kubernetes naming rules)

**GitOps File Paths:**
- Cluster-wide config: `<env>/env.yaml`
- Per-project config: `<env>/apps/<repository>/<project>/env.yaml`
- EnvironmentConfig resources generated from these files by Argo CD

**Label Conventions:**
- Cluster-wide config labels:
  - `forge.projectcatalyst.io/type: cluster`
- Per-project config labels:
  - `forge.projectcatalyst.io/type: project`
  - `forge.projectcatalyst.io/project: <project-name>`

### Our Value Schema

**Standard Environment Fields:**
```yaml
spec:
  data:
    environment:
      name: string        # Environment identifier
      domain: string      # Base domain for environment
      region: string      # Cloud region or location
      tier: string        # Environment tier (dev/staging/prod)
```

**Standard Default Fields by XRD Kind:**

*Deployment Defaults:*
```yaml
defaults:
  Deployment:
    replicas: int
    resources:
      requests:
        cpu: string
        memory: string
      limits:
        cpu: string
        memory: string
    autoscaling:
      enabled: bool
      minReplicas: int
      maxReplicas: int
```

*Network Defaults:*
```yaml
defaults:
  Network:
    tlsEnabled: bool
    ingressClass: string
    certificateIssuer: string
```

*Database Defaults:*
```yaml
defaults:
  Database:
    engine: string
    version: string
    instanceClass: string
    storageGB: int
    backupRetentionDays: int
```

**Per-Project Override Structure:**
```yaml
# In per-project EnvironmentConfig: <env>-<repository>-<project>
spec:
  overrides:
    <xr-name>:
      # Any field from XRD spec can be overridden
      # Structure mirrors XRD spec fields
      <field>: <value>
```

**Global Value Structure:**
Platform-managed values accessible to all XRs:
```yaml
spec:
  data:
    platform:
      registry: string           # Container registry URL
      objectStorage:
        endpoint: string
        bucket: string
      secretBackend:
        type: string             # "vault" or "secrets-manager"
        endpoint: string
      observability:
        metricsEndpoint: string
        logsEndpoint: string
```

## Performance Considerations

### Our Optimization Strategy

**Caching Approach:**

Composition functions maintain per-reconciliation caches:

**EnvironmentConfig Cache:**
- Cached after first lookup during reconciliation
- Prevents multiple API calls for same EnvironmentConfig
- Cache cleared after reconciliation completes
- Ensures fresh data on each reconciliation cycle

**XR Reference Cache:**
- Target XRs cached by namespace/name key during reference resolution
- Avoids repeated lookups for same XR within single reconciliation
- Cache invalidated after reconciliation completes

### Lookup Complexity

**Value Resolution Complexity:**
- O(1) for each precedence level lookup (map access)
- O(4) total = O(1) for full precedence chain
- Deep path traversal: O(d) where d = path depth (typically 2-4 levels)
- Overall: **O(1) for practical purposes**

**Reference Resolution Complexity:**
- Parsing: O(p) where p = reference path length (constant, <50 chars)
- XR lookup: O(1) with caching, O(n) without (n = XRs in namespace)
- Cross-namespace lookup: +1 additional API call
- Overall: **O(1) with caching per reconciliation**

**EnvironmentConfig Size Impact:**
```yaml
# Typical size estimates:
environment: ~100 bytes
defaults:
  <10 XRD kinds>:
    <10 fields each>:
      ~50 bytes per field
  Total: ~5 KB

projects:
  <100 projects>:
    <5 XRs per project>:
      <5 overrides per XR>:
        ~100 bytes per override
  Total: ~250 KB

Total EnvironmentConfig: ~255 KB (well under 1 MB etcd limit)
```

### Scale Limits We Expect

**EnvironmentConfig Limits:**
- Max size: 1 MB (Kubernetes resource limit)
- Practical limit: ~100 projects with ~5 XRs each = 500 override sets
- Fields per override: ~20 fields average
- Total overrides: ~10,000 field values (well within limits)

**Reconciliation Performance:**
- EnvironmentConfig fetch: ~10ms (cached Kubernetes API call)
- Value resolution: ~1ms per field (in-memory map lookups)
- Reference resolution: ~20ms per reference (API call for target XR)
- Total reconciliation overhead: ~50-100ms for typical XR

**Scaling Factors:**
- Number of environments: Linear (each is separate EnvironmentConfig)
- Number of projects: Linear (map lookup remains O(1))
- Number of XRs per project: Linear (each reconciles independently)
- Reference depth: No impact (not recursive, single-level lookup)

**Expected Scale:**
- Environments: 5-10 per platform (dev, staging, prod, etc.)
- Projects per environment: 100-500
- XRs per project: 5-20
- Total XRs per environment: 500-10,000
- Performance: Sub-second reconciliation even at scale

## Future Considerations

**Dynamic Value Sources:**
- Integration with external configuration stores (Consul, etcd)
- Runtime value resolution from external APIs
- Feature flag service integration
- Challenge: Cache invalidation and consistency

**Secret Rotation Integration:**
- Automatic secret rotation triggers XR reconciliation
- External Secrets Operator webhook integration
- Reference resolution detects secret version changes
- Challenge: Coordinating rotation across dependent XRs

**Policy-Based Overrides:**
- OPA/Gatekeeper policies enforce override constraints
- Policy validates which projects can override which fields
- Automatic override application based on labels/annotations
- Challenge: Policy complexity and debugging

**Multi-Environment References:**
- References across environments (e.g., staging references prod database)
- Requires cross-cluster XR lookups
- Security model for cross-environment access
- Challenge: Network connectivity and authentication

**Performance Optimizations:**
- Persistent EnvironmentConfig cache (shared across reconciliations)
- Background XR indexing for faster reference resolution
- Batch reference resolution for multiple references
- Challenge: Cache invalidation and memory usage

---

**Related Documents:**
- [GitOps Deployment Design](03-gitops-deployment.md) - How releases are deployed and EnvironmentConfigs created
- [Composition Management Design](04-composition-management.md) - XRD versioning and evolution
- Architecture: Implementation Guide - High-level environment configuration concepts
- Architecture: Developer Guide - User-facing configuration patterns and reference syntax

**Last Updated:** 2025-10-07
