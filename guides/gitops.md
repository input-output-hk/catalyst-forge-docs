# GitOps Design

## Table of Contents

- [Executive Summary](#executive-summary)
- [Repository Structure](#repository-structure)
  - [Canonical Directory Layout](#canonical-directory-layout)
  - [Environment Variations](#environment-variations)
  - [Project Organization](#project-organization)
  - [File Restrictions](#file-restrictions)
- [EnvironmentConfig Management](#environmentconfig-management)
  - [Storage and Reconciliation](#storage-and-reconciliation)
  - [Namespace Placement](#namespace-placement)
  - [Per-Project Overrides](#per-project-overrides)
- [Argo CD Application Architecture](#argo-cd-application-architecture)
  - [Application Hierarchy](#application-hierarchy)
  - [App-of-Apps Implementation](#app-of-apps-implementation)
  - [Application Naming Conventions](#application-naming-conventions)
  - [Sync Strategy](#sync-strategy)
- [Custom Management Plugin](#custom-management-plugin)
  - [Deployment Configuration](#deployment-configuration)
  - [Processing Algorithm](#processing-algorithm)
  - [Error Handling](#error-handling)
- [Self-Hosting Implementation](#self-hosting-implementation)
  - [Bootstrap to Self-Managed Transition](#bootstrap-to-self-managed-transition)
  - [Platform Service Management](#platform-service-management)
  - [Platform XRD Releases](#platform-xrd-releases)
  - [Updates and Rollbacks](#updates-and-rollbacks)
- [Multi-Cluster Management](#multi-cluster-management)
  - [Environment Cluster Creation](#environment-cluster-creation)
  - [Complete Environment Provisioning Flow](#complete-environment-provisioning-flow)
- [Open Questions](#open-questions)
- [Constraints](#constraints)
- [Future Considerations](#future-considerations)

## Executive Summary

This document defines the complete GitOps implementation for Catalyst Forge, including repository structure, Argo CD Application hierarchy, Custom Management Plugin behavior, self-hosting patterns, and multi-cluster management.

The platform uses a single GitOps repository with hierarchical organization. Each environment (including the platform cluster itself) is managed as a first-class environment with standardized structure. All deployments flow through ReleasePointer files that reference Release OCI images by commit SHA.

**Key Design Decisions:**

- **Platform cluster as environment**: The platform cluster has its own `platform/` directory managed identically to dev/staging/production
- **Unified release pattern**: Platform services, infrastructure, and applications all follow the same release-based deployment pattern
- **Infrastructure as versioned releases**: Core infrastructure (operators, service mesh, etc.) deployed via versioned releases of XR bundles
- **Self-contained environment clusters**: Each cluster has its own Crossplane managing its own resources with automatically registered Argo CD access
- **App-of-apps pattern**: Root ApplicationSet discovers all environments, each environment defines its own Applications/ApplicationSets
- **Bootstrap exception**: Only Argo CD in the platform cluster requires manual Helm installation; Argo CD then manages itself and all other components via GitOps

**Status:** Complete - All design questions resolved

---

## Repository Structure

### Canonical Directory Layout

The GitOps repository follows a standardized structure across all environments:

```
gitops-repo/
├── app.yaml                           # Root ApplicationSet (manually applied once)
├── platform/
│   ├── app.yaml                       # Platform Applications + ApplicationSets
│   ├── env.yaml                       # EnvironmentConfig for platform cluster
│   ├── apps/                          # All releases (services + infrastructure)
│   │   ├── catalyst-forge/            # Platform services repository
│   │   │   ├── api-server/
│   │   │   │   ├── release.yaml       # ReleasePointer
│   │   │   │   └── env.yaml (optional)
│   │   │   ├── worker-service/
│   │   │   │   ├── release.yaml
│   │   │   │   └── env.yaml (optional)
│   │   │   └── dispatcher/
│   │   │       ├── release.yaml
│   │   │       └── env.yaml (optional)
│   │   └── catalyst-forge-xrds/       # Platform XRDs and infrastructure repository
│   │       ├── database-v1/           # XRD definitions
│   │       │   └── release.yaml
│   │       ├── network-v1/            # XRD definitions
│   │       │   └── release.yaml
│   │       ├── deployment-v1/         # XRD definitions
│   │       │   └── release.yaml
│   │       └── infrastructure/        # XR bundle (operators, service mesh, etc.)
│   │           └── release.yaml
│   └── clusters/
│       ├── dev/
│       │   └── cluster.yaml           # Cluster XRD (creates cluster + registers with Argo CD)
│       ├── staging/
│       │   └── cluster.yaml
│       ├── production/
│       │   └── cluster.yaml
│       └── platform/
│           └── cluster.yaml           # PlatformCluster XRD (infrastructure only)
├── dev/
│   ├── app.yaml                       # Dev Applications + ApplicationSets
│   ├── env.yaml                       # EnvironmentConfig for dev
│   └── apps/                          # Application deployments
│       ├── catalyst-forge-xrds/       # Platform infrastructure (deployed to dev)
│       │   ├── database-v1/
│       │   │   └── release.yaml
│       │   ├── network-v1/
│       │   │   └── release.yaml
│       │   ├── deployment-v1/
│       │   │   └── release.yaml
│       │   └── infrastructure/
│       │       └── release.yaml       # Core infrastructure bundle for dev
│       └── <user_repos>/              # User applications
│           └── <projects>/
│               ├── release.yaml
│               └── env.yaml (optional)
├── staging/
│   ├── app.yaml
│   ├── env.yaml
│   └── apps/
│       ├── catalyst-forge-xrds/       # Same XRDs + infrastructure
│       └── <user_repos>/              # User applications
└── production/
    ├── app.yaml
    ├── env.yaml
    └── apps/
        ├── catalyst-forge-xrds/       # Same XRDs + infrastructure
        └── <user_repos>/              # User applications
```

**Key Principles:**

- **Consistent `/apps` directory**: All environments use `/apps` for releases (applications and infrastructure)
- **ReleasePointer pattern**: All deployments reference Release OCI images via `release.yaml` files
- **EnvironmentConfig location**: Each environment has `<env>/env.yaml` at root, optional per-project overrides at `<env>/apps/<repo>/<project>/env.yaml`
- **Cluster separation**: Only platform environment has `/clusters` directory; environment clusters are provisioned from platform via Crossplane XRDs
- **Infrastructure as releases**: XRDs and infrastructure operators deployed via standard release pattern

### Environment Variations

**Platform Environment Specifics:**

The platform environment differs from application environments:
- Contains `/clusters` directory with XRDs for provisioning environment clusters
- Contains platform services (API, Worker, Dispatcher) in `/apps/catalyst-forge`
- Contains platform XRDs and infrastructure in `/apps/catalyst-forge-xrds`

**Application Environments (dev/staging/production):**

Application environments follow a hybrid structure:
- NO `/clusters` directory (clusters provisioned from platform)
- Platform infrastructure deployed via releases in `/apps/catalyst-forge-xrds/*`
- User applications in `/apps/<user_repos>/*`

Each environment cluster requires:
1. XRD definitions deployed first (database-v1, network-v1, deployment-v1, etc.)
2. Infrastructure bundle deployed after XRDs (operators, service mesh, DNS, secrets management)
3. User applications deployed last

### Project Organization

All projects follow the canonical `<repo_name>/<project_name>` identifier pattern:

```
<env>/apps/<repo_name>/
└── <project_name>/
    ├── release.yaml
    └── env.yaml (optional)
```

This structure applies universally to single-project and multi-project repositories. Each project has independent release pointer and optional environment override.

**Namespace Creation:**

Namespaces are created automatically by Argo CD Applications using naming pattern: `<repo_name>-<project_name>`.

### File Restrictions

Only two files permitted in project directories:
- `release.yaml` (required) - ReleasePointer resource
- `env.yaml` (optional) - Per-project EnvironmentConfig override

Any other YAML files are illegal and ignored by the Custom Management Plugin.

---

## EnvironmentConfig Management

### Storage and Reconciliation

Each environment has an `env.yaml` file at the environment root containing a single EnvironmentConfig resource.

**Reconciliation Flow:**

1. Root ApplicationSet discovers `<env>/app.yaml` and creates `<env>-bootstrap` Application
2. Bootstrap Application reconciles `<env>/app.yaml`, which contains a Config Application
3. Config Application (`<env>-config`) reconciles `<env>/env.yaml`
4. Argo CD applies EnvironmentConfig as standard Kubernetes resource

### Namespace Placement

- **Cluster-wide EnvironmentConfig**: `<env>/env.yaml` → applied to `platform` namespace (cluster-scoped)
- **Per-project EnvironmentConfig**: `<env>/apps/<repo>/<project>/env.yaml` → applied to `<repo>-<project>` namespace (project-scoped)

### Per-Project Overrides

When the Custom Management Plugin processes a `release.yaml`, it checks for an adjacent `env.yaml`:
1. Fetches Release OCI image (extracts application resources)
2. Checks for `env.yaml` in same directory
3. Returns both release resources AND per-project EnvironmentConfig to Argo CD
4. Argo CD applies both sets of resources to project namespace

**File Structure:**

Each `env.yaml` contains a single EnvironmentConfig resource:

```yaml
apiVersion: apiextensions.crossplane.io/v1alpha1
kind: EnvironmentConfig
metadata:
  name: dev  # Environment name for cluster-wide, or project-specific name for overrides
data:
  replicas: "3"
  domain: "dev.projectcatalyst.io"
  # Additional key-value pairs
```

---

## Argo CD Application Architecture

### Application Hierarchy

The platform implements app-of-apps pattern with a root ApplicationSet that discovers all environments:

```
Root ApplicationSet (app.yaml at repo root)
└── Per-Environment Bootstrap Applications
    ├── Environment Config Application (reconciles env.yaml)
    └── Apps ApplicationSet (creates Applications for releases in /apps)

Platform Environment Additional:
└── Clusters ApplicationSet (discovers /clusters/*)
    └── Cluster Applications (one per environment)
```

**Application Types:**

- **Root ApplicationSet**: Discovers all `*/app.yaml` files
- **Environment Bootstrap Applications**: Reconciles environment's `app.yaml`
- **Environment Config Applications**: Reconciles environment-root `env.yaml`
- **Apps ApplicationSets**: Discovers `apps/*/*` and creates Applications
- **Workload Applications**: One per project per environment (created dynamically)
- **Clusters ApplicationSet**: Platform environment only, discovers `platform/clusters/*`

### App-of-Apps Implementation

**Root ApplicationSet:**

```yaml
# Root app.yaml (manually applied once)
kind: ApplicationSet
metadata:
  name: environments
  namespace: argocd
spec:
  generators:
    - git:
        repoURL: https://github.com/org/gitops-repo
        revision: main
        files:
          - path: "*/app.yaml"
  template:
    metadata:
      name: '{{path[0]}}-bootstrap'
    spec:
      source:
        path: '{{path[0]}}'
        directory:
          include: 'app.yaml'
      destination:
        server: https://kubernetes.default.svc
        namespace: argocd
```

This creates Applications: `platform-bootstrap`, `dev-bootstrap`, `staging-bootstrap`, `production-bootstrap`.

**Standard Environment app.yaml:**

Each application environment defines two resources:

```yaml
---
# Application for environment config
kind: Application
metadata:
  name: dev-config
  namespace: argocd
spec:
  source:
    path: dev
    directory:
      include: 'env.yaml'
  destination:
    server: https://kubernetes.default.svc
---
# ApplicationSet for apps
kind: ApplicationSet
metadata:
  name: dev-apps
  namespace: argocd
spec:
  generators:
    - git:
        repoURL: https://github.com/org/gitops-repo
        revision: main
        directories:
          - path: dev/apps/*/*
  template:
    metadata:
      name: 'dev-{{path[2]}}-{{path[3]}}'
    spec:
      source:
        path: '{{path}}'
        plugin:
          name: catalyst-forge-cmp
      destination:
        server: https://kubernetes.default.svc
        namespace: '{{path[2]}}-{{path[3]}}'  # repo-project
      syncPolicy:
        syncOptions:
          - CreateNamespace=true
```

**Platform Environment app.yaml:**

Platform environment adds a third resource for cluster management:

```yaml
---
# (config and apps ApplicationSets same as above)
---
# ApplicationSet for clusters (platform only)
kind: ApplicationSet
metadata:
  name: clusters
  namespace: argocd
spec:
  generators:
    - git:
        directories:
          - path: platform/clusters/*
  template:
    metadata:
      name: '{{path[2]}}-cluster'
    spec:
      source:
        path: '{{path}}'
      syncPolicy:
        syncOptions:
          - SyncWave=0  # Create clusters before apps
```

This creates: `dev-cluster`, `staging-cluster`, `production-cluster`, `platform-cluster`.

### Application Naming Conventions

- Bootstrap Applications: `<env>-bootstrap`
- Config Applications: `<env>-config`
- Workload Applications: `<env>-<repo>-<project>`
- Cluster Applications: `<env>-cluster`

### Sync Strategy

**Config Applications:**
```yaml
syncPolicy:
  automated: {}
```

**Apps ApplicationSets:**
```yaml
syncPolicy:
  automated: {}
  syncOptions:
    - CreateNamespace=true
```

**Clusters ApplicationSets:**
```yaml
syncPolicy:
  automated: {}
  syncOptions:
    - SyncWave=0  # Clusters before apps
```

All Applications use default pruning and self-healing.

---

## Custom Management Plugin

The Custom Management Plugin (CMP) processes ReleasePointer files and extracts Release OCI images. It runs as a sidecar container in the Argo CD repo-server deployment.

### Deployment Configuration

**Sidecar Registration:**

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: cmp-plugin
  namespace: argocd
data:
  plugin.yaml: |
    kind: ConfigManagementPlugin
    metadata:
      name: catalyst-forge-cmp
    spec:
      discover:
        fileName: "release.yaml"
      generate:
        command: ["/usr/local/bin/catalyst-forge-cmp"]
        args: ["generate"]
```

**Credentials:**

Registry and Platform API credentials are mounted from Kubernetes Secrets created during CMP deployment. Credential management during CMP deployment is covered in Implementation Strategy (Phase 4: Platform Services).

### Processing Algorithm

When Argo CD detects a `release.yaml` file, the CMP binary executes:

1. **Read ReleasePointer**: Parse `release.yaml` from Git working directory
2. **Extract commit SHA**: Read `spec.release` field
3. **Fetch Release OCI**: Pull image using format `[registry]/[repo]/[project]:[commit-sha]`
4. **Extract resources layer**: Unpack tar archive containing Kubernetes YAML manifests
5. **Check for per-project EnvironmentConfig**: If `env.yaml` exists adjacent to `release.yaml`, include it
6. **Return manifests**: Stream all YAML documents to stdout for Argo CD

**Output**: Multi-document YAML stream containing application resources and optional EnvironmentConfig override.

**Implementation:** Custom Go binary using platform OCI client library.

### Error Handling

**Missing Release OCI Image:**
- CMP logs error with release details
- Returns error to Argo CD
- Application enters degraded state with error message visible in UI

**Corrupt Release OCI Image:**
- CMP logs error with image details
- Returns error to Argo CD
- Application enters degraded state

**Invalid ReleasePointer:**
- CMP logs parsing error
- Returns error to Argo CD
- Application enters degraded state

All errors include sufficient context for platform engineers to diagnose and resolve issues.

---

## Self-Hosting Implementation

### Bootstrap to Self-Managed Transition

The platform uses a simplified bootstrap process where only Argo CD requires manual installation.

**Step 1: Manual Argo CD Installation**

```bash
helm install argocd argo/argo-cd \
  --namespace argocd \
  --create-namespace
```

**Step 2: Apply Root Configuration**

```bash
kubectl apply -f app.yaml
```

**Root app.yaml Structure:**

```yaml
---
# Argo CD self-management
kind: Application
metadata:
  name: argocd
  namespace: argocd
spec:
  source:
    repoURL: https://argoproj.github.io/argo-helm
    chart: argo-cd
    targetRevision: 5.51.0
    helm:
      values: |
        # Platform-specific configuration maintained in Git
  destination:
    namespace: argocd
---
# Crossplane deployment
kind: Application
metadata:
  name: crossplane
  namespace: argocd
spec:
  source:
    repoURL: https://charts.crossplane.io/stable
    chart: crossplane
    targetRevision: 1.14.0
    helm:
      values: |
        # Platform-specific configuration maintained in Git
  destination:
    namespace: crossplane-system
  syncPolicy:
    syncOptions:
      - CreateNamespace=true
---
# Environment discovery
kind: ApplicationSet
metadata:
  name: environments
  namespace: argocd
spec:
  generators:
    - git:
        files:
          - path: "*/app.yaml"
  template:
    metadata:
      name: '{{path[0]}}-bootstrap'
    spec:
      source:
        path: '{{path[0]}}'
        directory:
          include: 'app.yaml'
```

Full Helm values configuration maintained in Git for version tracking and reproducibility.

**Step 3: Automatic Transition**

Argo CD processes the root `app.yaml`:
1. **Self-adoption**: Argo CD Application reconciles and adopts the existing Helm release
2. **Crossplane deployment**: Argo CD deploys Crossplane via Helm chart
3. **Environment discovery**: ApplicationSet discovers all environment folders and creates bootstrap Applications

**Benefits:**
- Only one manual step (Argo CD Helm install)
- Immediate self-management
- Crossplane via GitOps from day one
- Both Argo CD and Crossplane versions tracked in Git

### Platform Service Management

Platform services follow the same release-based deployment pattern as applications.

**Deployment:**

Platform engineers manually deploy releases using CLI:

```bash
forge deployment create \
  --release <release-id> \
  --project api-server \
  --environment platform
```

This updates `platform/apps/catalyst-forge/api-server/release.yaml`:

```yaml
apiVersion: forge.projectcatalyst.io/v1alpha1
kind: ReleasePointer
metadata:
  name: api-server
spec:
  release: abc123def456  # commit SHA
```

**Argo CD Reconciliation:**
1. Detects pointer file change in Git
2. CMP extracts Release OCI image
3. Returns Kubernetes manifests
4. Argo CD applies resources to platform cluster
5. Platform service rolls out new version

**Sync Policy:**

Platform service Applications use manual sync policy instead of automated sync to prevent automatic deployments when pointer files change, requiring explicit engineer approval for platform updates.

### Platform XRD Releases

Platform infrastructure is managed through two types of releases in the `catalyst-forge-xrds` repository.

**XRD Definition Releases:**

Individual XRD families (database-v1, network-v1, deployment-v1, etc.) are released as separate projects containing Crossplane Configuration and Function packages. The CI pipeline builds xpkg artifacts and creates Release OCI images containing Configuration and Function manifests.

**Infrastructure Bundle Release:**

The `infrastructure` project packages XR instances for all supporting services into a single versioned release.

```
catalyst-forge-xrds/infrastructure/
├── .forge/
│   └── project.cue      # Bundles XR instances into release
└── README.md            # Documents infrastructure bundle
```

**Release Content:**

The Release OCI image contains XR instances (not XRD definitions):

```cue
// .forge/project.cue
deploy: {
  resources: [
    {
      apiVersion: "forge.projectcatalyst.io/v1alpha1"
      kind: "ExternalDNS"
      metadata: name: "external-dns"
      spec: { /* configuration */ }
    },
    {
      apiVersion: "forge.projectcatalyst.io/v1alpha1"
      kind: "ExternalSecretsOperator"
      metadata: name: "eso"
      spec: { /* configuration */ }
    },
    {
      apiVersion: "forge.projectcatalyst.io/v1alpha1"
      kind: "Istio"
      metadata: name: "istio"
      spec: { /* configuration */ }
    },
    // cert-manager, KEDA, monitoring, etc.
  ]
}
```

**Versioning:**

Infrastructure releases follow semantic versioning (v1.0.0, v1.1.0, v2.0.0). Each environment can run different infrastructure versions for staged rollouts.

**Deployment:**

Platform engineers deploy infrastructure releases manually:

```bash
forge deployment create \
  --release <infrastructure-release-id> \
  --project infrastructure \
  --environment dev
```

This updates `dev/apps/catalyst-forge-xrds/infrastructure/release.yaml`. Argo CD reconciles the pointer, CMP extracts the Release OCI, and Argo CD applies all XR instances to the environment cluster. Crossplane reconciles each XR, deploying the supporting services.

**Dependencies:**

Infrastructure bundle depends on XRD definitions being installed first. The manual deployment sequence ensures correct ordering. For future automation, Argo CD sync waves could enforce dependencies declaratively.

See [Crossplane Versioning Strategy](guides/crossplane/versioning.md) for complete details on version progression and CompositionRevision management.

**Bootstrap Constraint:**

During initial bootstrap, the platform cluster has no releases yet. The FIRST installation of platform XRDs and infrastructure must be manual (`kubectl apply -f`) since the release mechanism requires working platform services. After the platform is operational, all subsequent updates use pointer files.

### Updates and Rollbacks

**Update Process:**

Platform updates follow standard deployment workflow:
1. CI pipeline creates release for platform component
2. Engineer updates pointer file via CLI (or direct Git commit)
3. Argo CD detects change and reconciles
4. Crossplane reconciles for infrastructure updates

**Rollback:**

Rollback performed by reverting pointer file to previous release:

```bash
# CLI rollback
forge deployment rollback \
  --project api-server \
  --environment platform \
  --to-release <previous-release-id>

# Or manual Git revert
git revert <commit-that-updated-pointer>
git push
```

**Database Migrations:**

Platform API manages database schema. Migrations run as part of API deployment. Schemas must be forward-compatible to support rollback.

---

## Multi-Cluster Management

### Environment Cluster Creation

Environment clusters are provisioned from the platform cluster via Cluster XRD resources in `platform/clusters/<env>/cluster.yaml`.

**Cluster XRD Responsibilities:**

1. **Cluster Provisioning**: Creates the Kubernetes cluster (EKS on AWS, vCluster on bare metal)
2. **Crossplane Bootstrap**: Deploys Crossplane to the new cluster via Helm provider
3. **Argo CD Registration**: Creates cluster Secret in platform cluster for Argo CD access

**Implementation:**

```yaml
# platform/clusters/dev/cluster.yaml
apiVersion: forge.projectcatalyst.io/v1alpha1
kind: Cluster
metadata:
  name: dev
spec:
  provider: aws  # or vcluster for bare metal
  region: us-west-2
  nodeGroups:
    - name: general
      instanceType: t3.large
      minSize: 3
      maxSize: 10
```

**Composition Steps:**

1. Create Cluster (EKS or vCluster)
2. Write ClusterAuth Secret (kubeconfig)
3. Create Helm ProviderConfig pointing at new cluster
4. Deploy Crossplane Helm chart to new cluster
5. Configure Crossplane Providers in new cluster for self-management
6. Register with Argo CD (create cluster Secret with label `argocd.argoproj.io/secret-type: cluster`)

**Argo CD Cluster Secret:**

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: dev-cluster
  namespace: argocd
  labels:
    argocd.argoproj.io/secret-type: cluster
type: Opaque
stringData:
  name: dev
  server: https://dev-eks-cluster.us-west-2.eks.amazonaws.com
  config: |
    {
      "tlsClientConfig": {
        "insecure": false,
        "caData": "...",
        "certData": "...",
        "keyData": "..."
      }
    }
```

This Secret automatically registers the dev cluster with Argo CD.

**Platform Cluster XRD:**

The platform cluster uses a PlatformCluster XRD that deploys infrastructure without creating a cluster:

```yaml
# platform/clusters/platform/cluster.yaml
apiVersion: forge.projectcatalyst.io/v1alpha1
kind: PlatformCluster
metadata:
  name: platform
spec:
  # No cluster creation - platform cluster already exists
  # Deploys base infrastructure only
```

### Complete Environment Provisioning Flow

The complete process for provisioning a new environment from cluster creation through application readiness.

**Step 1: Create Cluster**

Platform engineer creates Cluster XRD:

```bash
cat <<EOF | kubectl apply -f -
apiVersion: forge.projectcatalyst.io/v1alpha1
kind: Cluster
metadata:
  name: dev
spec:
  provider: aws
  region: us-west-2
  # ... cluster configuration
EOF
```

Crossplane provisions cluster, bootstraps Crossplane into it, and registers with Argo CD.

**Step 2: Create Environment Folder**

Platform engineer creates environment structure in GitOps repository:

```bash
mkdir -p dev/apps
# Commit app.yaml, env.yaml to Git
```

Root ApplicationSet detects new environment and creates `dev-bootstrap` Application. Bootstrap Application creates `dev-config` Application (for env.yaml) and `dev-apps` ApplicationSet (for apps).

**Step 3: Deploy XRD Definitions**

Platform engineer deploys XRD releases:

```bash
# For each XRD family
forge deployment create \
  --release <xrd-release-id> \
  --project database-v1 \
  --environment dev

forge deployment create \
  --release <xrd-release-id> \
  --project network-v1 \
  --environment dev

# ... repeat for all XRD families
```

Pointer files created in `dev/apps/catalyst-forge-xrds/*/release.yaml`. Argo CD reconciles, installing XRDs to dev cluster.

**Step 4: Deploy Infrastructure Bundle**

Platform engineer deploys infrastructure:

```bash
forge deployment create \
  --release <infrastructure-release-id> \
  --project infrastructure \
  --environment dev
```

Pointer file created in `dev/apps/catalyst-forge-xrds/infrastructure/release.yaml`. Argo CD reconciles, applying XR instances. Crossplane deploys supporting services.

**Dependencies:**

The infrastructure bundle XRs depend on XRD definitions being available. The deployment sequence ensures XRDs are installed before infrastructure is applied. If infrastructure is deployed before XRDs, Argo CD will report sync failure until XRDs become available.

**Step 5: Verify Environment**

Platform engineer verifies infrastructure health:

```bash
# Check Argo CD Applications
kubectl get applications -n argocd | grep dev

# Check Crossplane XR status in dev cluster
kubectl get externaldn,externalsecrets,istio -n default

# Verify operator pods
kubectl get pods -n external-secrets-system
kubectl get pods -n istio-system
```

**Step 6: Enable User Deployments**

Environment is now ready for user application deployments. Users deploy via standard pointer file pattern:

```bash
forge deployment create \
  --release <app-release-id> \
  --project my-app \
  --environment dev
```

**Environment Updates:**

To upgrade infrastructure in an environment:
1. Create new infrastructure release with updated XR specs
2. Deploy new release to environment
3. Argo CD detects pointer change and applies updated XRs
4. Crossplane reconciles changes to supporting services

Rollback is achieved by reverting the pointer file to previous release.

---

## Open Questions

No unresolved questions remain. The GitOps design is complete.

---

## Constraints

**GitOps Repository:**
- Single GitOps repository per installation
- Git provider must be accessible from platform cluster
- Repository size limits may apply for large installations

**Argo CD:**
- Application count limits per cluster (typically 1000+)
- ApplicationSet generator performance with large directory counts
- Custom Management Plugin must be registered before Applications can use it

**Custom Management Plugin:**
- Release OCI image must be accessible from platform cluster
- Plugin execution timeout may affect large releases
- Memory constraints for extracting large Release OCI images

**Platform Bootstrap:**
- Argo CD must be deployed manually before GitOps activation
- Initial platform infrastructure installation requires manual `kubectl apply`
- Cannot bootstrap from empty cluster without these prerequisites

---

## Future Considerations

**Multi-Repository GitOps:**
Support for multiple GitOps repositories with coordination between them for large installations.

**Progressive Delivery:**
Integration with Argo Rollouts for canary and blue/green deployments of platform services.

**Dependency Ordering Automation:**
Argo CD sync waves to enforce XRD → Infrastructure → Application ordering declaratively.

**Multi-Region Deployments:**
GitOps patterns for platform deployments spanning multiple AWS regions or geographic locations.

**Disaster Recovery:**
Automated backup and restoration of GitOps repository state, Application configurations, and cluster registrations.

**Audit and Compliance:**
Enhanced audit logging for Git commits tied to deployments, approval workflows for production changes.

---

**Related Documents:**
- Architecture Overview - Ports and adapters pattern, deployment model
- System Architecture - GitOps deployment flow, OCI image formats
- Implementation Guide - Bootstrapping sequence, self-hosting model
- Developer Guide - ReleasePointer usage, deployment workflows
- Crossplane Versioning Strategy - Infrastructure release versioning (guides/crossplane/versioning.md)

**Status:** Complete - All design questions resolved

**Last Updated:** 2025-10-07