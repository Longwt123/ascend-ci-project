# Ascend-Ci-Project — Meta Repository for Ascend CI Platform

## Overview

This is a **git superproject** (meta-repo) that aggregates the entire Ascend CI platform into a single cloneable unit. It contains **no application code** — it exists solely as a convenience layer to clone all platform components via `git clone --recurse-submodules`.

The platform provides **self-hosted GitHub Actions runners on Ascend NPU hardware** for open-source AI/ML projects. It enables projects like vLLM, SGLang, Triton, and PyTorch to run their CI workflows on Huawei Ascend 910B/C and 310P NPU chips.

**GitHub Organization:** `opensourceways`
**Contact:** `ascendinfra@huawei.com`

---

## Architecture (Bird's-Eye View)

```
┌──────────────────────────────────────────────────────────────────┐
│                    Ascend-Ci-Project (this repo)                  │
│                    Meta-repo, .gitmodules only                    │
├──────────────────────────────────────────────────────────────────┤
│                                                                  │
│  ┌─────────────────────┐  ┌──────────────────────────────────┐  │
│  │ ascend-ci-argocd    │  │ ascend-ci-deployment              │  │
│  │ ArgoCD Applications │  │ K8s Manifests + Helm Charts       │  │
│  │ (What + Where)      │  │ (The actual deployment config)    │  │
│  └────────┬────────────┘  └──────────────┬───────────────────┘  │
│           │                              │                       │
│           │  GitOps: ArgoCD watches      │                       │
│           │  these repos, syncs to ──────┤                       │
│           │  K8s clusters               │                       │
│           │                              │                       │
│  ┌────────┴──────────────────────────────┴───────────────────┐  │
│  │ ascend-runner-onboarding                                   │  │
│  │ Go service: GitHub App → auto-generates configs in both    │  │
│  │ argocd + deployment repos, creates PRs                     │  │
│  └────────────────────────────────────────────────────────────┘  │
│                                                                  │
└──────────────────────────────────────────────────────────────────┘
```

### Data Flow (Onboarding a New Project)

```
1. User installs GitHub App "ascend-runner-mgmt"
2. Webhook → ascend-runner-onboarding (Go service)
3. Service writes installation_id to HashiCorp Vault
4. Service clones ascend-ci-deployment, copies reference project,
   applies field replacements → git push + PR
5. Service generates ArgoCD Application manifests → git push + PR
6. PR Poller waits for merge
7. On merge: ArgoCD syncs → ARC creates RunnerScaleSet pods
8. User adds `runs-on: linux-aarch64-a3-2` to their workflow
```

---

## Submodule Reference

### 1. `ascend-ci-deployment` — K8s Infrastructure as Code

**Repo:** `https://github.com/opensourceways/ascend-ci-deployment.git`

**Purpose:** Contains all Kubernetes manifests, Helm chart values, and Kustomize overlays for deploying GitHub Actions Runner Controller (ARC) and supporting infrastructure across ~12 Ascend NPU clusters.

**Key Technologies:**
- **Helm v3** — `gha-runner-scale-set` chart (ARC v0.12.0)
- **Kustomize** — Kubernetes native config composition for per-cluster overlays
- **ArgoCD** (GitOps consumer) — watches this repo's main branch, auto-syncs to clusters
- **Vault Agent Injector** — dynamic secret/cert injection via `SecretDefinition` CRD
- **Prometheus + kube-prometheus-stack** — monitoring (remote-write to central Prometheus)

**Directory Convention:**
```
{ORG-OR-USER}/
  {REPOSITORY}/
    config/                          # Kustomize: namespace, secrets, PVC, RBAC, configmaps
      kustomization.yaml
      namespace.yaml
      github-token-secret.yaml       # Vault-backed SecretDefinition
      local-storage-pvc.yaml          # Shared storage (SFS Turbo or local hostPath)
      runner-pod-permission.yaml      # ServiceAccount + Role + RoleBinding
      pre-execute-script-check-npu-configmap.yaml
      container-job-pod-template-npu-{N}-configmap.yaml
      custom-runner-container-hook-pvc.yaml
    config-{REGION}/                  # Region-specific config overlays (e.g., config-hk001)
    linux-{ARCH}-{NPU_TYPE}-{COUNT}/  # Helm chart per runner scale-set spec
      Chart.yaml                      # References gha-runner-scale-set as dependency
      Chart.lock
      values.yaml                     # githubConfigUrl, secret, pod templates, metrics
```

**NPU Types Supported:**
| Label Prefix | Hardware | Sizes |
|---|---|---|
| `linux-arm64-npu` | Legacy NPU | 0, 1, 2, 4, 8 |
| `linux-aarch64-a2` | Ascend 910B (A2) | 0, 1, 2, 4, 8 |
| `linux-aarch64-a3` | Ascend 910C (A3) | 0, 2, 4, 8, 16 |
| `linux-aarch64-310p` | Ascend 310P | 1, 2, 4 |
| `linux-aarch64-910c` | Ascend 910C | 0, 2, 4, 8, 16 |
| `linux-arm64-cpu` | CPU-only | 1, 4, 8, 16, 24 |

**Clusters — Full Inventory:**

| Short Name | ArgoCD Destination | Region | ARC | Projects | ci-deployment config | ci-argocd dir |
|---|---|---|---|---|---|---|
| `gy-003` | `openmerlin-guiyang-003-cluster` | Guiyang | 0.14.2 | vllm-ascend, ascend-gha-runners | `config-for-guiyang-003/` | `gy-003/` |
| `gy-004` | `openmerlin-guiyang-004-cluster` | Guiyang | 0.14.2 | sglang, sgl-kernel-npu | `config-for-guiyang-004/` | `gy-004/` |
| `gy-005` | `openmerlin-guiyang-005-cluster` | Guiyang | 0.14.2 | vllm-ascend, vllm-omni, tile-ai, triton-lang, cosdt | `config-for-guiyang-005/` | `gy-005/` |
| `gy-006` | `openmerlin-guiyang-006-cluster` | Guiyang | test | vllm-ascend, vllm-omni | `config-for-guiyang-006/` | `gy-006/`, `openmerlin-guiyang-006/` |
| `gy-007` | `openmerlin-guiyang-007-cluster` | Guiyang | — | — | — | `gy-007/`, `openmerlin-guiyang-007/` |
| `hk-001` | `ascend-hk-001-cluster` | Hong Kong | 0.14.2 | vllm-ascend, sglang, verl, LLaMA-Factory, ms-swift, linkedin, modelscope | `config-hk001/`, `config-for-hk-001/` | `hk-001/` |
| `hk-ci` | `infra-hk-opensourceway-ci-cluster` | Hong Kong | 0.14.2 | opensourceways, agentic-develop-playground | (in org dirs) | `hk-ci/` |
| `cn12-001` | `ascend-cn12-001-cluster` | **华北 (Huabei)** | 0.14.2 | vllm-omni, triton-lang, tile-ai, alibaba/ROLL | `config-cn12-001/`, `config-for-cn12/` | `cn12-001/` |
| `hb-003` | `ascend-aiframework` | **华北 (Huabei-3)** | 0.13.0 | Ascend/pytorch | `config-for-hb003/` (volcengine/verl) | `hb-003/` |
| `hb-003-verl` | `ascend-mind-third-ci` | **华北 (Huabei-3)** | 0.13.0 | volcengine/verl, ascend-gha-runners/sync-tools | `config-for-hb003-verl/` | `hb-003-verl/` |
| `hd-001` | `ascend-ci-sglang-cluster-001` | Huadong (华东) | 0.14.2 | sglang, sgl-kernel-npu | `config-for-sglang01/` | `hd-001/`, `ascend-ci-sglang-huadong2-cluster/` |
| `hidevlab-k8s` | `hidevlab-k8s` | Dev/Lab | 0.14.2 | vllm-ascend, add-node-check | `config-for-hidevlab-k8s/` | `hidevlab-k8s/` |
| `verl-suzhou` | `in-cluster` | Suzhou | 0.14.2 | volcengine/verl | `config-for-suzhou/` | `verl-suzhou/` |
| `karmada-test` | `ascend-karmada-test-cluster` | Test | 0.14.2 | vllm-ascend | `config-for-karmada-test/` | `ascend-karmada-test/` |
| `infra-cn4-x86` | `infra-cn4-x86-common-cluster` | Infra | — | monitoring only | `monitoring/config-for-infra-cn4-x86-common-cluster/` | `infra-cn4-x86-common-cluster/` |

> **华北 (Huabei) clusters together:** `cn12-001`, `hb-003` (aiframework), `hb-003-verl` (mind-third-ci), and `ascend-triton-agent-ci` (planned, not yet in code).
> When modifying clusters, both `ascend-ci-deployment` and `ascend-ci-argocd` must be updated together — they are two sides of the same deployment.

**Projects Hosted (~20):** vllm-project/vllm-ascend, sgl-project/sglang, triton-lang/triton-ascend, tile-ai/tilelang-ascend, volcengine/verl, modelscope/ms-swift, hiyouga/LLaMA-Factory, Ascend/Ascend-CI, pytorch-fdn, linkedin/liger-kernel, alibaba/ROLL, and more.

**Storage Patterns:**
- **SFS Turbo** (Huawei cloud): `csi-sfsturbo` storage class, `ReadWriteMany`, shared across pods
- **Local:** `hostPath` for bare-metal physical disks
- **Ephemeral:** `csi-disk` for temporary runner work directories

**Claude Code Skills (within this submodule):**
- `arc-deploy` — generates namespace.yaml, secret, PVC, configmap, kustomization, Helm values from JSON config templates

**Branch Protection Rules:** See `AGENTS.md` in submodule — all changes must go through PRs, mandatory CI checks, CODEOWNERS review.

---

### 2. `ascend-ci-argocd` — ArgoCD Application Manifests

**Repo:** `https://github.com/opensourceways/ascend-ci-argocd.git`

**Purpose:** Stores ArgoCD `Application` CRDs that define GitOps deployment targets. This is the **"what deploys where"** layer — each Application YAML tells ArgoCD which source repo path to sync to which cluster namespace.

**Two-layer pattern:**

**Layer 1 — Infrastructure Apps** (point back to this repo):
```
applications/argocd/{component}.yaml          # Application CRD
applications/deploy/{component}/              # Actual Helm/Kustomize resources
```
Infrastructure components: `nginx-pypi-cache`, `vault`, `secret-manager`, `imagepullsecret-patch`, `scheduler-plugins`, `npu-exporter`, `prometheus`, `volcano-queue`, `buildkitd-server`, `smart-git-proxy`, `lws`, `argus`

**Layer 2 — CI Runner Apps** (point to ascend-ci-deployment):
```
applications/argocd/{cluster}/
  {org}-{repo}-config-for-{cluster}.yaml     # Kustomize Application (config/)
  {org}-{repo}-{runner-spec}.yaml            # Helm Application (runner scale-set)
```

**Standard Application YAML template:**
```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: {app-name}
  namespace: argocd
spec:
  destination:
    namespace: {k8s-namespace}
    name: {cluster-name}          # or 'in-cluster' for local clusters
  project: {project-name}
  source:
    path: {org}/{repo}/{subdir}
    repoURL: https://github.com/opensourceways/ascend-ci-deployment.git
    targetRevision: HEAD
    helm:                          # Only for runner type
      releaseName: {short-name}
  syncPolicy:
    automated:
      prune: true
    syncOptions:
      - CreateNamespace=true
```

**Naming Convention:** `{org-lower}-{repo-lower}-{descriptor}` (e.g., `vllm-project-vllm-ascend-linux-aarch64-a3-2`)

**ARC Controller applications:** Found in `applications/argocd-controller/` — deploy the ARC controller itself (v0.13.0, v0.13.1, v0.14.2) using `ServerSideApply=true`

**Clusters Managed:** See the [full cluster inventory](#1-ascend-ci-deployment--k8s-infrastructure-as-code) in the deployment section above. The same clusters appear as ArgoCD Application directories under `applications/argocd/{cluster-name}/` and ARC controller files under `applications/argocd-controller/arc-controller-{cluster-name}.yaml`.

**Claude Code Skills (within this submodule):**
- `app-argocd-onboarding` — generates new ArgoCD Application YAML files for onboarding projects
- `argocd-arc-app` — generates arc-controller, config, and runner Application manifests from JSON config

---

### 3. `ascend-runner-onboarding` — Provisioning Gateway (Go)

**Repo:** `https://github.com/opensourceways/ascend-runner-onboarding.git`

**Purpose:** A Go HTTP service that automates the end-to-end onboarding process when a user installs the `ascend-runner-mgmt` GitHub App. Transforms what was previously a manual ops ticket into a self-service, click-to-provision workflow.

**Key Technologies:**
- **Go 1.25** — all application code
- **go-chi/chi** — HTTP router
- **go-github/v66** — GitHub API client
- **Redis Stream** (optional) — multi-instance HA via consumer groups, XREADGROUP, XAUTOCLAIM
- **PostgreSQL / SQLite** — dual-driver installation store
- **HashiCorp Vault** — KV v2 store for GitHub App installation IDs
- **Docker** — distroless/static-debian12:nonroot runtime

**Architecture Invariants (critical for making changes):**
1. **Vault writes MUST happen before git pushes** — If reversed, ArgoCD could sync a `SecretDefinition` pointing to a non-existent Vault key
2. `DeriveNaming()` in `internal/arcdeploy/render.go` is the **single source of truth** for org/repo → namespace/dir/URL naming
3. `Provisioner` is an interface with two implementations: `DryRun` (local testing) and `GitOps` (real git push + PR)
4. Reference projects ARE the templates — copy + 4-5 string replacements, no template engine
5. Allowlist is YAML file with `mtime`-based hot reload; parse errors log ERROR but keep last-good memory snapshot

**Directory Layout:**
```
cmd/server/main.go              # Entry point: wiring, routing, graceful shutdown
cmd/deprovision/main.go         # CLI for manual deprovisioning
internal/
  allowlist/                    # YAML allowlist, mtime hot-reload, NPU type resolution
  arcdeploy/                    # Core provisioning engine
    render.go                   # DeriveNaming, BuildReplacements, DetectOldValues, GenerateAppSecret
    gitops.go                   # GitOps provisioner: clone, replace, commit, push, create PR
    argocd.go                   # ArgoCD Application YAML generation
    dryrun.go                   # Dry-run implementation for local testing
    migrate.go                  # PAT → GitHub App secret migration
    pr_creator.go / pat_pr_creator.go  # PR creation strategies
    conflict.go                 # Namespace/directory conflict detection
  installation/                 # Business logic + state machine
    model.go                    # Record struct, Status constants (8 states)
    service.go                  # HandleCreated/Deleted/Suspended/Unsuspended/ReposUpdated/RecoverProvisioning
    pr_poller.go                # PR status polling (every 10 min)
    store.go                    # PostgreSQL + SQLite dual-driver with auto-schema
  queue/                        # Job queue: in-memory channel + Redis Stream
  worker/                       # Queue consumer goroutine
  vaultclient/                  # Minimal Vault HTTP client (AppRole login, KV v2 PATCH)
  githubapp/                    # JWT auth, HMAC-SHA256 webhook verification
  handler/                      # HTTP handlers: webhook, pages, cluster_status, rate limiting
  clusterprobe/                 # Optional NPU usage monitoring via K8s API + Prometheus
  config/                       # Env var loading + validation (~40 config vars)
config/reference-projects.yaml  # NPU type definitions, 12 clusters, vault config
web/templates/                  # Landing + setup HTML pages
deploy/Dockerfile               # Multi-stage distroless build
```

**State Machine (8 states):**
```
new → pending (not on allowlist)
new → provisioning → pr_pending → provisioned
                   → failed
                   → suspended (app paused)
                   → deleted (uninstalled)
```

**Queue Modes:**
- **Single-instance:** In-memory channel (capacity 100), `sync.Mutex` for git push serialization
- **Multi-instance HA:** Redis Stream + Consumer Group, `XAUTOCLAIM` for dead consumer recovery (5min idle), at-least-once delivery
- Toggle: set `REDIS_URL` env var to enable Redis mode

**CI/CD:**
- `ci.yml`: lint (`golangci-lint`) + build + test (`-race -count=1`)
- `release.yml`: Build Docker image → push to `swr.cn-north-4.myhuaweicloud.com/opensourceways/ascend-runner-onboarding`
- Release managed via `opensourceways/release-mgmt` issue-based flow

---

## ARC Multi-Cluster HA & Load Balancing

### Custom ARC Controller

The platform uses a **customized Actions Runner Controller** based on upstream v0.14.2:

- **Helm chart:** Official `gha-runner-scale-set` v0.14.2
- **Controller & listener image:** `swr.cn-southwest-2.myhuaweicloud.com/modelfoundry/gha-runner-scale-set-controller:0.14.201`

This custom build adds multi-cluster HA and cluster-aware load balancing on top of the upstream ARC.

### Multi-Cluster HA via Scale Set Labels

ARC v0.14.2 and above supports runner scale set labels to enable multi-cluster HA. When runner scale sets in **different clusters** carry the same label, GitHub Actions can route a job matching that label to **any** of those clusters — achieving cross-cluster failover.

In `ascend-ci-deployment`, labels are configured in runner `values.yaml` under `gha-runner-scale-set.scaleSetLabels`:

```yaml
gha-runner-scale-set:
  scaleSetLabels:
    - "linux-aarch64-a3-8"
    - "cn12-001"
```

**Two labels, two purposes:**

| Label | Purpose |
|---|---|
| `linux-aarch64-a3-8` | Runner **capability label** — what the user writes in `runs-on`. GitHub Actions uses this to discover and match runners. If the same label exists on runner scale sets in multiple clusters, any of them can claim the job → multi-cluster HA. |
| `cn12-001` | **Cluster identity label** — marks which cluster this runner scale set belongs to. Used for cluster-level isolation and routing within the platform. |

> **Rule:** Every runner scale set on ARC >= 0.14.0 **must** carry at least these two labels: the capability label and the cluster name label.

### Cluster Load Balancing via Resource Annotations

ARC v0.14.2+ supports **resource-aware job scheduling** through annotations on the ephemeral runner set. When a job arrives, ARC checks the resource requirements and routes it to a cluster with sufficient capacity:

```yaml
gha-runner-scale-set:
  resourceMeta:
    ephemeralRunnerSet:
      annotations:
        actions.github.com/job-cpu: "128"
        actions.github.com/job-memory: "512Gi"
        actions.github.com/job-npu: "huawei.com/ascend-1980:8"
```

| Annotation | Meaning |
|---|---|
| `actions.github.com/job-cpu` | CPU cores the job pod requests |
| `actions.github.com/job-memory` | Memory the job pod requests |
| `actions.github.com/job-npu` | NPU vendor resource in K8s format (`vendor/device:count`) |

These annotations let ARC perform cluster-level load balancing — a job that needs 8 NPUs won't be sent to a cluster where all NPU nodes are full.

---

## Naming Conventions

### Runner Namespace Naming

Namespaces are derived from the GitHub `{org}/{repo}` pair. The rules differ for new vs. existing projects:

**New projects (post-onboarding service):**
- Convert the full `{org}/{repo}` to **lowercase with hyphen**: `{org-lower}-{repo-lower}`
- Example: `alibaba/ROLL` → namespace `alibaba-roll`
- This is done automatically by the onboarding service; the transformation is the responsibility of `DeriveNaming()` in `ascend-runner-onboarding/internal/arcdeploy/render.go`

**Existing projects (legacy, pre-onboarding service):**
- Keep the **existing namespace format as-is**. Do not rename.
- Example: `vllm-project/vllm-ascend` → namespace remains `vllm-project` (NOT `vllm-project-vllm-ascend`)

> **Rule of thumb:** When adding a new org/repo, lowercase both the org and repo names and join with `-`. When modifying an existing project, preserve whatever namespace it already has.

### Runner Scale Set Naming

The runner scale set name format depends on the **ARC controller version**:

**ARC < 0.14.0 (legacy):**
```
linux-{arch}-{npu_type}-{npu_count}
```
- Does NOT include the cluster name
- Example: `linux-aarch64-a3-2`
- User's workflow: `runs-on: linux-aarch64-a3-2`

**ARC >= 0.14.0 (current):**
```
linux-{arch}-{npu_type}-{npu_count}-{cluster_suffix}
```
- MUST include the cluster suffix
- The `{cluster_suffix}` is derived from the **Short Name** in the cluster inventory table, with hyphens removed for guiyang-style names:
  - `gy-003` → `gy003`, `gy-005` → `gy005`, `gy-006` → `gy006`
  - `cn12-001` → `cn12-001` (hyphens preserved)
  - `hk-001` → `hk001`
  - `hb-003` → `hb003`
- Example scale set directory: `linux-aarch64-a3-2-cn12-001`, `linux-aarch64-a3-2-gy006`
- **MUST also set TWO labels** in `gha-runner-scale-set.scaleSetLabels`:
  1. The **capability label**: `linux-aarch64-a3-2` (short form matching the legacy pattern) — this is what GitHub Actions uses for `runs-on` matching
  2. The **cluster name label**: `cn12-001` or `gy006` — identifies which cluster this scale set runs on
- User's workflow: `runs-on: linux-aarch64-a3-2` (the capability label, NOT the full scale set name)

> **Why:** GitHub Actions discovers runners by their labels. The capability label `linux-aarch64-a3-2` is what the user writes in their workflow. The cluster label `cn12-001` or `gy006` is for platform-internal routing. Without the capability label, GitHub cannot find the runner.
>
> **Note on cluster suffix format:** The suffix in directory names strips hyphens from the cluster short name for guiyang/hk/hb clusters (`gy003`, `hk001`, `hb003`) but preserves them for cn12 (`cn12-001`). Always match the existing convention in the codebase for the specific cluster you're working with.

**Naming components:**

| Component | Values |
|---|---|
| `{arch}` | `arm64` (legacy), `aarch64` (current) |
| `{npu_type}` | `npu` (legacy), `a2` (910B), `a2b3` (910B3), `a3` (910C), `310p` (310P), `910c` (910C), `cpu` (no NPU) |
| `{npu_count}` | 0, 1, 2, 4, 8, 16 (varies by NPU type) |
| `{cluster_name}` | **Must match the Short Name from the cluster inventory table** (e.g., `gy005`, `cn12-001`, `hb003`). This same value is used as the cluster label in `scaleSetLabels` and as the suffix in the scale set name. Note: `cn12-001` keeps its hyphens, while `hb003` and `hk001` are hyphen-free — **use exactly the short name as listed in the cluster table**. |

**Label requirements summary for ARC >= 0.14.0:**

Every runner scale set must carry **both** labels in `scaleSetLabels`:
```yaml
gha-runner-scale-set:
  scaleSetLabels:
    - "linux-aarch64-a3-2"    # capability label (without cluster suffix)
    - "cn12-001"              # cluster name label (= Short Name from cluster table)
```

- The **capability label** is how GitHub Actions discovers the runner (`runs-on: linux-aarch64-a3-2`)
- The **cluster name label** uses the cluster Short Name exactly as listed in the cluster inventory (e.g., `cn12-001`, `gy006`, `hb003`)
- The full scale set directory name (`linux-aarch64-a3-2-cn12-001`) is the ArgoCD/internal identifier — it is NOT what users write in workflows

---

## Development Conventions (All Repos)

### General
- **Bilingual:** Documentation and comments are in both English and Chinese
- **GitOps:** All deployments are via ArgoCD watching git repos; no `kubectl apply` or `helm install` directly to production clusters
- **PR-first:** All changes go through pull requests; direct pushes to main are blocked
- **Branch naming for onboarding:** `onboarding/{targetDir}` for auto-generated PRs

### YAML/K8s Conventions
- Kubernetes manifests use `kustomization.yaml` (not `kustomization.yml`)
- Helm charts use `values.yaml` for configuration overrides
- All namespaces are created by ArgoCD (`CreateNamespace=true`)
- ARC controllers always go in `arc-systems` namespace; runners in org-named namespaces
- Resource naming pattern: `{org-lower}-{repo-lower}-{resource-type}`

### Go Conventions (ascend-runner-onboarding)
- Go 1.25 with sumdb verification
- Linting via `.golangci.yml`: errcheck, govet, staticcheck, unused, ineffassign, gosimple
- Tests use table-driven patterns with mock interfaces
- Vault integration tests require `vault_live` build tag
- Error handling: always wrap errors with context, log at appropriate level

### Security
- GitHub tokens NEVER stored in plaintext — always via Vault-backed `SecretDefinition`
- Webhooks are HMAC-SHA256 verified (1 MiB body cap)
- Image pull secrets managed by `imagepullsecret-patcher` DaemonSet
- Container images pinned by SHA in production; use `latest` only in dev

---

## Key Inter-Repository Relationships

1. **ascend-runner-onboarding writes to both ascend-ci-deployment and ascend-ci-argocd** — it is the only component that creates new deployment configs programmatically
2. **ascend-ci-argocd points to ascend-ci-deployment** — all CI runner Applications reference `repoURL: https://github.com/opensourceways/ascend-ci-deployment.git`
3. **ArgoCD watches both repos** — changes merged to main in either repo trigger automatic sync
4. **allowlist in ascend-runner-onboarding must match** the org/repo directory structure in ascend-ci-deployment — naming is derived by `DeriveNaming()`, not manually configured

---

## Common Tasks Reference

### Adding a new project to the Ascend CI platform
1. Add to allowlist in `ascend-runner-onboarding/config/reference-projects.yaml`
2. Either let the GitHub App onboarding flow auto-generate configs, OR manually:
   - Create `{org}/{repo}/config/` directory in `ascend-ci-deployment`
   - Create ArgoCD Application YAMLs in `ascend-ci-argocd/applications/argocd/{cluster}/`

### Adding a new cluster
1. Add cluster definition to `ascend-runner-onboarding/config/reference-projects.yaml`
2. Add ArgoCD Application directory: `ascend-ci-argocd/applications/argocd/{cluster-name}/`
3. Add ARC controller Application: `ascend-ci-argocd/applications/argocd-controller/arc-controller-{cluster}.yaml`
4. Add monitoring config: `ascend-ci-deployment/monitoring/config-for-{cluster}/`

### Upgrading ARC version
1. Add new Helm chart: `ascend-ci-argocd/applications/deploy/arc-controller-{version}/`
2. Update ARC controller Applications to point to new path
3. Update runner scale-set Charts to reference new ARC version
4. See `ascend-ci-deployment/checklist/arc-upgrade/` for detailed checklist

### Debugging onboarding failures
1. Check installation status: query `installations` table in PostgreSQL/SQLite
2. Check Vault: verify `{org}_installation_id` key exists in KV v2
3. Check PR status: look at `note` field in installation record for PR URLs
4. Check queue: if using Redis, inspect Stream + DLQ; if memory, check service logs
