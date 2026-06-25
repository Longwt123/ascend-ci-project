# Ascend-Ci-Project — AI Coding Reference

## TL;DR

这是一个 **git 超级项目**，聚合 3 个子模块来管理 Ascend NPU 上的 GitHub Actions CI 平台：

| 子模块 | 职责 | CLAUDE.md |
|--------|------|-----------|
| `ascend-ci-deployment` | K8s manifests + Helm values | [链接](ascend-ci-deployment/CLAUDE.md) |
| `ascend-ci-argocd` | ArgoCD Application CRD | [链接](ascend-ci-argocd/CLAUDE.md) |
| `ascend-runner-onboarding` | Go 服务：webhook → 自动开通 | [链接](ascend-runner-onboarding/CLAUDE.md) |

**数据流**：GitHub App 安装 → webhook → onboarding 服务 → Vault + git push → ArgoCD 同步 → Runner 可用

---

## Quick Reference Card

| Question | Answer |
|---|---|
| K8s manifests 在哪？ | `ascend-ci-deployment/{org}/{repo}/` |
| ArgoCD Applications 在哪？ | `ascend-ci-argocd/applications/argocd/{cluster}/` |
| 开通逻辑在哪？ | `ascend-runner-onboarding/internal/arcdeploy/` |
| 新项目的 namespace？ | `{org-lower}-{repo-lower}` |
| Runner 目录 (ARC < 0.14.0)？ | `linux-{arch}-{npu}-{count}` |
| Runner 目录 (ARC >= 0.14.0)？ | `linux-{arch}-{npu}-{count}-{cluster_suffix}` |
| ARC >= 0.14.0 标签？ | 必须有 2 个：capability + cluster |
| Vault key 格式？ | `{org}_installation_id` |

---

## Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│ Ascend-Ci-Project (this repo) — .gitmodules only, no app code   │
├─────────────────────────────────────────────────────────────────┤
│  ┌──────────────────────┐  ┌───────────────────────────────┐   │
│  │ ascend-ci-argocd     │  │ ascend-ci-deployment           │   │
│  │ ArgoCD Applications  │  │ K8s Manifests + Helm Values    │   │
│  │ "what deploys where" │  │ "the actual deployment config" │   │
│  └────────┬─────────────┘  └────────────┬──────────────────┘   │
│           │  ArgoCD watches both repos  │                       │
│           │  auto-syncs to clusters ────┤                       │
│           │                             │                       │
│  ┌────────┴─────────────────────────────┴───────────────────┐   │
│  │ ascend-runner-onboarding (Go)                             │   │
│  │ GitHub App webhook → generates configs → git push + PR    │   │
│  └───────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────┘
```

**Data flow (onboarding):**
1. GitHub App `ascend-runner-mgmt` installed → webhook
2. `ascend-runner-onboarding` receives webhook, validates allowlist
3. **Vault first**: PATCH `{org}_installation_id` to KV v2 (BEFORE git push!)
4. Clone `ascend-ci-deployment`, copy reference project, string-replace 4-5 fields
5. Generate `SecretDefinition` for GitHub App auth
6. Git push → create PR to `ascend-ci-deployment`
7. Generate ArgoCD Application YAMLs → git push → create PR to `ascend-ci-argocd`
8. PR Poller waits for merge; on merge: ArgoCD syncs → ARC creates pods
9. User adds `runs-on: linux-aarch64-a3-2` to workflow

---

## Submodule Map

| Submodule | Repo | What It Contains |
|---|---|---|
| `ascend-ci-deployment` | `opensourceways/ascend-ci-deployment` | K8s manifests, Helm values, Kustomize overlays, monitoring |
| `ascend-ci-argocd` | `opensourceways/ascend-ci-argocd` | ArgoCD Application CRDs, infrastructure Helm charts |
| `ascend-runner-onboarding` | `opensourceways/ascend-runner-onboarding` | Go HTTP service: webhook → auto-provision |

---

## Cluster Inventory (CANONICAL — update BOTH deployment + argocd when modifying)

| Short Name | ArgoCD Destination | ARC | Projects (sample) | Config Dirs in deployment | ArgoCD App Dir |
|---|---|---|---|---|---|
| `gy-003` | `openmerlin-guiyang-003-cluster` | 0.13.0 | vllm-ascend, Ascend-CI, ascend-gha-runners, nv-action/vllm-benchmarks | `config/`, `config-for-gy003/` | `gy-003/` |
| `gy-004` | `openmerlin-guiyang-004-cluster` | 0.13.0 | sglang, sgl-kernel-npu, Ascend/sglang, ascend-gha-runners | `config/`, `config-gy004/`, `config-for-guiyang004/` | `gy-004/` |
| `gy-005` | `openmerlin-guiyang-005-cluster` | **0.14.2*** | vllm-ascend, vllm-omni, tile-ai, triton-lang, Ascend/ray-ascend, GDzhu01, nv-action, xuedinge233 | `config/`, `config-for-guiyang-005/` | `gy-005/` |
| `gy-006` | `openmerlin-guiyang-006-cluster` | **0.14.2*** | vllm-ascend, vllm-omni, ascend-gha-runners | `config-for-guiyang-006/` | `gy-006/`, `openmerlin-guiyang-006/` |
| `gy-007` | `openmerlin-guiyang-007-cluster` | — | argo-workflows, infra only | — | `gy-007/`, `openmerlin-guiyang-007/` |
| `hk-001` | `ascend-hk-001-cluster` | 0.13.0 | vllm-ascend, sglang, verl, LLaMA-Factory, ms-swift, linkedin, modelscope, Ascend-CI, pytorch-fdn, cosdt, GDzhu01, nv-action, xuedinge233, ascend-gha-runners | `config/`, `config-hk001/`, `config-for-hk001/` | `hk-001/` |
| `hk-ci` | `infra-hk-opensourceway-ci-cluster` | 0.13.0 | opensourceways, agentic-develop-playground | `config/` (in org dirs) | `hk-ci/` |
| `cn12-001` | `ascend-cn12-001-cluster` | **0.14.2*** | vllm-omni, triton-lang, tile-ai, alibaba/ROLL, ascend-gha-runners/vllm-ascend, vllm-project/vllm-ascend, UsernameFull/ROLL | `config-cn12-001/`, `config-for-cn12/` | `cn12-001/` |
| `hb-003` | `ascend-aiframework` | 0.13.0 | Ascend/pytorch | `config/` | `hb-003/` |
| `hb-003-verl` | `ascend-mind-third-ci` | 0.13.0 | volcengine/verl, ascend-gha-runners/sync-tools | `config-for-hb003/`, `config-for-hb003-verl/` | `hb-003-verl/` |
| `hd-001` | `ascend-ci-sglang-cluster-001` | custom† | sglang, sgl-kernel-npu | `config/`, `config-for-sglang01/` | `hd-001/`, `ascend-ci-sglang-huadong2-cluster/` |
| `hidevlab-k8s` | `hidevlab-k8s` | 0.13.0 | vllm-ascend, add-node-check | `config-for-hidevlab-k8s/` | `hidevlab-k8s/` |
| `verl-suzhou` | `in-cluster` | 0.13.0 | volcengine/verl | `config-for-suzhou/` | `verl-suzhou/` |
| `karmada-test` | `ascend-karmada-test-cluster` | 0.13.0 | vllm-ascend | `config-for-karmada-test/` | `ascend-karmada-test/` |
| `infra-cn4-x86` | `infra-cn4-x86-common-cluster` | — | monitoring only (Prometheus, pushgateway) | `monitoring/config-for-infra-cn4-x86-common-cluster/` | `infra-cn4-x86-common-cluster/` |

> **\* = 0.14.2 custom**: uses `arc-controller-0.14.2` chart + controller image `swr.cn-southwest-2.myhuaweicloud.com/modelfoundry/gha-runner-scale-set-controller:0.14.201`. These clusters support `scaleSetLabels`, `resourceMeta`, and multi-cluster HA.  
> **† = custom CPU-node**: hd-001 uses `arc-controller-for-cpu-node` (no NPU scheduling).  
> **verl-suzhou special**: uses **Gitee** (`gitee.com/tfhoo/`) instead of GitHub, destination `in-cluster`.

**Region groupings:**
- **华北 (Huabei)**: cn12-001, hb-003 (aiframework), hb-003-verl (mind-third-ci), + `ascend-triton-agent-ci` (planned)
- **贵阳 (Guiyang)**: gy-003, gy-004, gy-005, gy-006, gy-007
- **香港 (Hong Kong)**: hk-001, hk-ci
- **华东 (Huadong)**: hd-001
- **Dev/Test**: hidevlab-k8s, karmada-test, verl-suzhou

---

## Naming Conventions (THE source of truth)

### 1. Namespace Naming

```
NEW project:   {org-lower}-{repo-lower}
               alibaba/ROLL → alibaba-roll
               tile-ai/tilelang-ascend → tile-ai-tilelang-ascend

EXISTING:      keep whatever namespace already exists
               vllm-project/vllm-ascend → vllm-project (legacy, do NOT rename!)
               Ascend/Ascend-CI → ascend
               Ascend/pytorch → ascend
```

**Rule**: When you see an existing namespace in `namespace.yaml`, preserve it exactly.  
**Implementation**: `DeriveNaming()` in `ascend-runner-onboarding/internal/arcdeploy/render.go` handles this.

### 2. Runner Scale Set Directory Naming

```
ARC < 0.14.0:    linux-{arch}-{npu_type}-{count}
                 linux-aarch64-a3-2
                 linux-aarch64-a2b3-4
                 linux-arm64-npu-1

ARC >= 0.14.0:   linux-{arch}-{npu_type}-{count}-{cluster_suffix}
                 linux-aarch64-a3-2-cn12-001
                 linux-aarch64-a3-2-gy006
                 linux-aarch64-a3-0-karmada-test
```

**Cluster suffix derivation** (from Short Name):
- `gy-003` → `gy003`, `gy-006` → `gy006`
- `cn12-001` → `cn12-001` (hyphens PRESERVED)
- `hk-001` → `hk001`, `hb-003` → `hb003`
- `hidevlab-k8s` → `for-hidevlab-k8s` (special legacy form)
- `karmada-test` → `karmada-test` (unchanged)

**Naming components:**

| Component | Values |
|---|---|
| `{arch}` | `arm64` (legacy), `aarch64` (current), `amd64` |
| `{npu_type}` | `npu` (legacy), `a2` (910B), `a2b3` (910B3), `a2b4` (910B4), `a3` (910C), `310p` (310P), `910c` (910C), `cpu` (no NPU) |
| `{count}` | 0, 1, 2, 4, 8, 16 (varies by NPU type) |
| `{cluster_suffix}` | derived from Short Name per rule above |

### 3. Runner Scale Set Labels (ARC >= 0.14.0)

Every runner scale set MUST carry exactly TWO labels in `scaleSetLabels`:

```yaml
gha-runner-scale-set:
  scaleSetLabels:
    - "linux-aarch64-a3-2"   # capability label = what user writes in runs-on
    - "cn12-001"             # cluster label = Short Name from cluster inventory
```

| Label | Purpose |
|---|---|
| Capability label | `{arch}-{npu}-{count}` — GitHub Actions uses this for `runs-on` matching |
| Cluster label | Cluster Short Name — platform-internal routing/isolation |

**Why two labels**: The capability label is what GitHub discovers. If the same capability label exists on scale sets in multiple clusters → multi-cluster HA (any cluster can claim the job). The cluster label identifies which specific cluster.

### 4. Resource Annotations (ARC >= 0.14.0)

```yaml
gha-runner-scale-set:
  resourceMeta:
    ephemeralRunnerSet:
      annotations:
        actions.github.com/job-cpu: "32"
        actions.github.com/job-memory: "128Gi"
        actions.github.com/job-npu: "huawei.com/ascend-1980:2"
```

These enable cluster-level load balancing — jobs are routed to clusters with sufficient capacity.

### 5. ArgoCD Application Naming

```
Pattern: {namespace}-{repo}-{descriptor}

Config apps:   {namespace}-{repo}-config[-{cluster}]
               vllm-project-vllm-ascend-config
               vllm-project-vllm-ascend-hk001-config
               ascend-gha-runners-vllm-ascend-config-for-guiyang-005

Runner apps:   {namespace}-{repo}-{runner_spec}
               vllm-project-vllm-ascend-linux-aarch64-a3-2
               vllm-project-vllm-ascend-linux-aarch64-a3-2-cn12-001

Controllers:   arc-controller-{cluster}  (naming varies, check existing files)
               arc-controller-hk-001, arc-controller-cn12-001, arc-test-openmerlin-guiyang-006
```

### 6. Secret / PVC / ConfigMap Naming

```
GitHub Secret:         {namespace}-{repo}-secret
PVC (SFS Turbo):       {namespace}-{repo}-{safe_cluster}
ServiceAccount:        runner-service-account  (fixed)
Role:                  runner-role             (fixed)
RoleBinding:           runner-rolebinding      (fixed)
Pod Template ConfigMap: {runner_prefix}-{count}  or  {runner_prefix}-{cluster_suffix}
```

---

## File Patterns

> 详细目录结构见各子模块 CLAUDE.md：
> - [ascend-ci-deployment](ascend-ci-deployment/CLAUDE.md) — `{org}/{repo}/` 布局
> - [ascend-ci-argocd](ascend-ci-argocd/CLAUDE.md) — `applications/` 布局

**Monitoring** (`ascend-ci-deployment/monitoring/`):
- `base/` — 共享基础配置（所有集群引用）
- `config-for-{cluster}/` — 集群特定 patch
- 监控集群: gy-003, gy-004, gy-005, gy-006, hk-001, infra-cn4-x86
- 详见 `monitoring/README.md`

---

## ARC Controller Version Matrix

| Controller Path | Tag | Clusters | Features |
|---|---|---|---|
| `arc-controller-0.13.0` | (none) | gy-003, gy-004, hb-003, hb-003-verl, hk-001, hk-ci, hidevlab-k8s, karmada-test, verl-suzhou | Base ARC, no multi-cluster HA |
| `arc-controller-0.14.2` | `0.14.201` | cn12-001, gy-005, gy-006 | scaleSetLabels, resourceMeta, multi-cluster HA, cluster load balancing |
| `arc-controller-for-cpu-node` | (none) | hd-001 | CPU-only scheduling |

**Custom image**: `swr.cn-southwest-2.myhuaweicloud.com/modelfoundry/gha-runner-scale-set-controller:0.14.201` (used by all 0.14.2 clusters)

**Key difference**: Only 0.14.2 clusters support `scaleSetLabels` and `resourceMeta`. When adding a new runner to a 0.14.2 cluster, you MUST include both labels and resource annotations.

---

## Runner Type Reference

| Label Prefix | Hardware | Sizes | Example Dir |
|---|---|---|---|
| `linux-arm64-npu` | Legacy NPU | 1, 2, 4, 8 | `linux-arm64-npu-2` |
| `linux-aarch64-a2` | 910B (A2) | 0, 1, 2, 4, 8 | `linux-aarch64-a2-4` |
| `linux-aarch64-a2b3` | 910B3 (A2B3) | 0, 1, 2, 4, 8 | `linux-aarch64-a2b3-8` |
| `linux-aarch64-a2b4` | 910B4 (A2B4) | 1, 2, 4, 8 | `linux-aarch64-a2b4-4` |
| `linux-aarch64-a3` | 910C (A3) | 0, 2, 4, 8, 16 | `linux-aarch64-a3-8` |
| `linux-aarch64-310p` | 310P | 1, 2, 4 | `linux-aarch64-310p-2` |
| `linux-aarch64-910c` | 910C | 0, 2, 4, 8, 16 | `linux-aarch64-910c-4` |
| `linux-arm64-cpu` | ARM64 CPU | 1, 4, 8, 16, 24 | `linux-arm64-cpu-16` |
| `linux-amd64-cpu` | x86_64 CPU | 0, 1, 4, 8, 16, 24 | `linux-amd64-cpu-8` |
| `linux-aarch64-cpu` | AArch64 CPU | 1, 2, 8, 16 | `linux-aarch64-cpu-8` |

---

## Container Images

| Component | Image |
|---|---|
| Runner (legacy) | `swr.cn-southwest-2.myhuaweicloud.com/base_image/actions/actions-runner:2.330.0` |
| Runner (modern) | `swr.cn-southwest-2.myhuaweicloud.com/modelfoundry/runner-containers-hooks:release-no_volumes-9c3ea5` |
| ARC Controller | `swr.cn-southwest-2.myhuaweicloud.com/modelfoundry/gha-runner-scale-set-controller:0.14.201` |
| ARC Helm chart | `oci://ghcr.io/actions/actions-runner-controller-charts/gha-runner-scale-set` |
| ARC Helm mirror | `oci://ghcr.nju.edu.cn/actions/actions-runner-controller-charts/gha-runner-scale-set` |

**Container modes**:  
- **Legacy**: No `containerMode` specified (Ascend/, Ascend/Ascend-CI, older ascend-gha-runners)  
- **Modern**: `containerMode.type: "kubernetes-novolume"` (all other projects)

---

## Storage Patterns by Cluster

| Storage Class | Clusters | Notes |
|---|---|---|
| `csi-sfsturbo` | Most (shanghai-icsl-cluster) | SFS Turbo, `ReadWriteMany`, shared across pods |
| `sfsturbo-subpath-sc` | agentic-develop-playground, opensourceways, gy-006, cn12-001 | Subpath variant |
| `csi-disk` | Legacy runners | Ephemeral work volume |
| ephemeral (no SC) | Modern `kubernetes-novolume` runners | `ReadWriteMany` for work dir |

---

## `ascend-runner-onboarding` — Go Service Reference

> 详见 [ascend-runner-onboarding/CLAUDE.md](ascend-runner-onboarding/CLAUDE.md)

---

## Common Operations — Step-by-Step Recipes

### Adding a new project to existing cluster

1. Add to allowlist: `ascend-runner-onboarding/config/allowlist.yaml`
2. Either:
   - Let GitHub App auto-provision, OR manually:
   - Create `{org}/{repo}/config[-{cluster}]/` in `ascend-ci-deployment`
   - Create `kustomization.yaml` referencing: namespace, pvc, rbac, secret, configmaps
   - Create `namespace.yaml` with `metadata.name: {namespace}`
   - Create `github-app-secret.yaml` (Vault-backed SecretDefinition)
   - Create `runner-pod-permission.yaml` (SA + Role + RoleBinding)
   - Create `local-storage-pvc.yaml` with appropriate storage class
   - For each NPU type: create `linux-{arch}-{npu}-{count}[-{suffix}]/` with Chart.yaml + values.yaml
3. Create ArgoCD Applications in `ascend-ci-argocd/applications/argocd/{cluster}/`:
   - Config app: `{ns}-{repo}-config[-{cluster}].yaml` → kustomize (no helm)
   - Runner app per spec: `{ns}-{repo}-{runner_spec}.yaml` → helm with releaseName

### Adding a new cluster

1. Add cluster definition to `ascend-runner-onboarding/config/reference-projects.yaml`
2. Create ArgoCD app dir: `ascend-ci-argocd/applications/argocd/{cluster-name}/`
3. Create ARC controller: `ascend-ci-argocd/applications/argocd-controller/arc-controller-{cluster}.yaml`
   - Choose version: 0.13.0 (base) or 0.14.2 (multi-cluster HA)
4. Add monitoring: `ascend-ci-deployment/monitoring/config-for-{cluster}/` (thin overlay on `base/`)
5. Add infra: imagepullsecret-patch, secrets-manager, nginx-pypi-cache as needed

### Upgrading ARC version for a cluster

1. Edit `arc-controller-{cluster}.yaml` → change `source.path` to target version
2. For 0.14.2: add custom image values (repository + tag `0.14.201`)
3. Update runner `values.yaml` files: add `scaleSetLabels`, `resourceMeta`, update `listenerTemplate` image
4. Add cluster suffix to runner directory names
5. See `ascend-ci-deployment/checklist/arc-upgrade/` for detailed checklist

### Modifying an existing project

1. Check if namespace is legacy (`vllm-project` style) or new (`org-repo` style)
2. If legacy: **preserve existing namespace name** — do NOT rename to new convention
3. If new: use `{org-lower}-{repo-lower}` format
4. Match the cluster suffix convention already in use for that cluster

---

## Gotchas & Edge Cases

| Gotcha | Explanation |
|---|---|
| **Vault order matters** | Vault write MUST happen before git push. Reversed = SecretDefinition points to non-existent key. |
| **Legacy namespaces** | `vllm-project/vllm-ascend` uses namespace `vllm-project`, NOT `vllm-project-vllm-ascend`. Do not rename existing namespaces. |
| | `Ascend/pytorch` uses namespace `ascend`, NOT `ascend-pytorch`. |
| | `volcengine/verl` on hb-003-verl uses namespace `verl-project`, on verl-suzhou uses `verl`. |
| **Cluster suffix format** | `cn12-001` keeps hyphens (`linux-aarch64-a3-2-cn12-001`), but `gy003` strips them (`linux-aarch64-a3-2-gy006`). |
| **Config dir naming** | Three variants: `config/` (default), `config-hk001/` (no "for"), `config-for-guiyang-005/` (with "for"). Match existing convention per cluster. |
| **ARC 0.14.2 labels** | Must have TWO: capability label + cluster label. One label = job routing broken. |
| **verl-suzhou** | Uses **Gitee** (`gitee.com/tfhoo/`), not GitHub. Different repoURL in ALL Application YAMLs. |
| **vllm-omni** | Uses **Buildkite** (`oci://ghcr.io/buildkite/helm`), not ARC. Has `buildkite-token-secret.yaml` instead of `github-token-secret.yaml`. |
| **ASCEND-GHA-RUNNERS org** | `ascend-gha-runners` is both an org in GitHub AND a namespace. Most new cluster-suffixed runners are here. |
| **syncOptions missing** | Many config apps lack `CreateNamespace=true`. When adding new apps, always include it. |
| **Controller file naming** | gy-003 uses `arc-controller.yaml` (generic), gy-004 uses `arc-controller-sglang.yaml`, hb-003 uses `arc-controller-huabei-003.yaml`, hb-003-verl uses `ascend-controller-hb-003-verl.yaml`. No uniform pattern. |
| **Go version** | `ascend-runner-onboarding` uses Go 1.26 (check `go.mod`, not memory). |
| **Secret type** | Two kinds: `github-token-secret.yaml` (PAT) and `github-app-secret.yaml` (GitHub App). New projects should use App auth. |

---

## Cross-Repo Change Checklist

When modifying a cluster or project, these repos must be updated together:

| Change | ascend-ci-deployment | ascend-ci-argocd | ascend-runner-onboarding |
|---|---|---|---|
| New project | Create `{org}/{repo}/` tree | Create Application YAMLs | Add to allowlist |
| New cluster | Add `config-for-{cluster}/` | Create `argocd/{cluster}/` + controller | Add to `reference-projects.yaml` |
| ARC upgrade | Update runner values.yaml | Update controller path + image | — |
| Rename/delete | Update namespace, PVC refs | Update/delete Application YAMLs | Update allowlist |

---

## Skills Available

| Skill | Location | Purpose |
|---|---|---|
| `arc-deploy` | `ascend-ci-deployment/.claude/skills/` | Generate K8s manifests from JSON config |
| `app-argocd-onboarding` | `ascend-ci-argocd/.claude/skills/` | Generate ArgoCD Application YAMLs for new projects |
| `argocd-arc-app` | `ascend-ci-argocd/.claude/skills/` | Generate ARC controller + config + runner Application YAMLs |

---

## Repo URLs

| Repo | URL |
|---|---|
| Superproject | `https://github.com/opensourceways/Ascend-Ci-Project` |
| Deployment | `https://github.com/opensourceways/ascend-ci-deployment.git` |
| ArgoCD | `https://github.com/opensourceways/ascend-ci-argocd.git` |
| Onboarding | `https://github.com/opensourceways/ascend-runner-onboarding.git` |
| verl-suzhou (Gitee) | `https://gitee.com/tfhoo/ascend-ci-deployment.git` |
| verl-suzhou ArgoCD | `https://gitee.com/tfhoo/ascend-ci-argocd.git` |
