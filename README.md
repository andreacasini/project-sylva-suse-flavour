# Dummy CAPI Unit for Sylva-Core (Rancher 2.13.1 + Rancher-Bundled Turtles)

## Goal

Replace the CAPI core operator with a no-op dummy **only on the management cluster** (after pivot), so that Rancher's built-in turtles (v0.25.1, bundled in Rancher 2.13.1) takes over CAPI operations. The bootstrap cluster keeps using the real CAPI controller as normal (Metal3/CAPM3 needs it to provision BareMetalHosts).

## Architecture overview

| Context | CAPI unit | Turtles | CAPI CRDs installed by |
|---------|-----------|---------|------------------------|
| Bootstrap cluster | Real CAPI (sylva-core `kustomize-units/capi/`) | N/A | Real CAPI controller |
| Management cluster | Dummy (ConfigMap only) | Rancher-bundled v0.25.1 (chart `108.0.1+up0.25.1`) | Turtles via CAPIProvider CR |

### Key discovery

Rancher 2.13.1 bundles a complete turtles chart at `/var/lib/rancher-data/local-catalogs/v2/rancher-charts/.../assets/rancher-turtles/rancher-turtles-108.0.1+up0.25.1.tgz`. This chart is self-contained: it includes `operator-crds.yaml` (CAPI Operator CRDs), `core-provider.yaml` (creates a `CAPIProvider` of type `core` that installs CAPI core CRDs + controllers), and the turtles deployment itself. When installed, it creates the `cattle-capi-system` namespace and installs CAPI v1.10.6 core CRDs automatically.

This is fundamentally different from Sylva's built-in `rancher-turtles` unit (v0.24.4), which has `cluster-api-operator.enabled: false` and `cluster-api.enabled: false` — meaning it does NOT install CAPI core CRDs.

### How `_internal.mgmt_cluster` enables conditional behavior

Sylva's `_internal.mgmt_cluster` variable is `false` on the bootstrap cluster and `true` on the management cluster. We use goTpl ternary expressions in the `capi` unit override to switch between the real CAPI (bootstrap) and the dummy (management).

## What you need

### Part 1: Create the dummy CAPI git repo

Create a public git repo (e.g., `https://github.com/youruser/sylva-capi-dummy.git`) with this structure:

```
dummy-capi/
├── kustomization.yaml
├── namespace.yaml
├── configmap.yaml
└── components/
    └── ha/
        └── kustomization.yaml
```

#### `dummy-capi/kustomization.yaml`

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - namespace.yaml
  - configmap.yaml
```

#### `dummy-capi/namespace.yaml`

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: capi-system
```

#### `dummy-capi/configmap.yaml`

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: capi-dummy-placeholder
  namespace: capi-system
data:
  info: "CAPI core operator replaced by rancher-turtles from Rancher 2.13.1"
```

#### `dummy-capi/components/ha/kustomization.yaml`

```yaml
apiVersion: kustomize.config.k8s.io/v1alpha1
kind: Component
```

This empty Component is required because the chart's default `_components` reference `components/ha` conditionally. Even though your non-HA cluster won't use it, the path must exist to prevent kustomize errors.

### Part 2: Modify your `values.yaml`

Add the following to your environment's `values.yaml` (e.g., `environment-values/vanilla-rke2-capm3/values.yaml`):

```yaml
# -----------------------------------------------------------
# 1. New Flux GitRepository source for the dummy capi content
# -----------------------------------------------------------
source_templates:
  sylva-capi-dummy:
    kind: GitRepository
    spec:
      url: https://github.com/youruser/sylva-capi-dummy.git  # <-- CHANGE THIS
      ref:
        branch: main

# -----------------------------------------------------------
# 2. Override capi unit: real on bootstrap, dummy on management
# -----------------------------------------------------------
units:
  capi:
    repo: '{{ .Values._internal.mgmt_cluster | ternary "sylva-capi-dummy" "sylva-core" }}'
    unit_templates:
      - base-deps
    depends_on:
      cert-manager: true
    kustomization_spec:
      path: '{{ .Values._internal.mgmt_cluster | ternary "./dummy-capi" "./kustomize-units/capi" }}'
      postBuild:
        substitute:
          force_var_substitution_enabled: "true"
      wait: true
      _components:
        - '{{ tuple "components/ha" (.Values._internal.ha_cluster.is_ha) | include "set-only-if" }}'

# -----------------------------------------------------------
# 3. Disable Sylva's standalone rancher-turtles unit
#    (Rancher's built-in turtles will be used instead)
# -----------------------------------------------------------
  rancher-turtles:
    enabled: false

# -----------------------------------------------------------
# 4. Pin Rancher chart version to 2.13.1 (SUSE Edge v3.5)
# -----------------------------------------------------------
  rancher:
    helmrelease_spec:
      chart:
        spec:
          version: "2.13.1"
```

### Critical: preserve the full unit structure

When overriding the `capi` unit, you **must** include all fields from the chart defaults (`unit_templates`, `depends_on`, `kustomization_spec.postBuild`, `kustomization_spec._components`). Helm's values merge replaces entire nested objects, so a partial override will wipe out required fields and cause schema validation errors.

## Deploying

Run the bootstrap as usual:

```bash
./bootstrap.sh environment-values/vanilla-rke2-capm3
```

### What happens during deployment

1. **Bootstrap cluster** — `_internal.mgmt_cluster` is `false`. The `capi` unit resolves to `repo: sylva-core`, `path: ./kustomize-units/capi` — the real CAPI core operator installs. Metal3/CAPM3 works, BareMetalHosts get provisioned, management cluster is created.

2. **Management cluster (after pivot)** — `_internal.mgmt_cluster` is `true`. The `capi` unit resolves to `repo: sylva-capi-dummy`, `path: ./dummy-capi` — only a ConfigMap is applied. The real CAPI controller does not run.

3. **Rancher-bundled turtles** — After Rancher starts on the management cluster, install the `rancher-turtles` chart (v108.0.1+up0.25.1) from Rancher's built-in catalog (Apps & Marketplace in the Rancher UI, or via Rancher's App CR). This chart:
   - Installs the CAPI Operator CRDs
   - Creates a `CAPIProvider` resource of type `core` in `cattle-capi-system`
   - This triggers installation of CAPI core CRDs (clusters.cluster.x-k8s.io, machinedeployments.cluster.x-k8s.io, etc.) at version v1.10.6
   - Deploys the turtles controller (`rancher/turtles:v0.25.1`)

4. The Flux Kustomization for the `capi` unit reports **Ready** (it successfully applied the ConfigMap).

## Known issue: `sylvactl` per-unit timeout

The `kyverno-policy-prevent-mgmt-cluster-delete` Kustomization depends on CAPI CRDs being present (its Kyverno policy watches `cluster.x-k8s.io/*/Cluster` resources). Since the Rancher-bundled turtles installs CRDs asynchronously after Rancher is up, there's a timing gap.

The `sylvactl` watch loop has a hardcoded 5-minute per-unit timeout that cannot be changed via environment variables. The script will error with:

```
✗ Unit timeout exceeded: unit Kustomization/kyverno-policy-prevent-mgmt-cluster-delete
  did not became ready after 5m0s
```

**This is cosmetic.** Flux continues reconciling in the background and the kustomization becomes Ready once the CRDs are installed. You can verify by checking after the script exits:

```bash
kubectl get kustomization kyverno-policy-prevent-mgmt-cluster-delete -n sylva-system
# Should show READY=True
```

Or simply re-run the bootstrap script — on the second run everything is already reconciled and it passes immediately.

### Timeout-related environment variables

These control the overall script timeout, NOT the per-unit timeout:

| Variable | Default | Used by |
|----------|---------|---------|
| `BOOTSTRAP_WATCH_TIMEOUT_MIN` | 30 | bootstrap.sh (bootstrap cluster units) |
| `MGMT_WATCH_TIMEOUT_MIN` | 45 | bootstrap.sh (management cluster units) |
| `APPLY_WC_WATCH_TIMEOUT_MIN` | N/A | apply-workload-cluster.sh only |

For slow homelab hardware, increasing `MGMT_WATCH_TIMEOUT_MIN` helps the overall script timeout but does not affect the 5-minute per-unit timeout in `sylvactl`.

## Verification

After deployment, on the management cluster:

```bash
# 1. Check dummy capi kustomization is Ready
kubectl get kustomization capi -n sylva-system

# 2. Confirm only the dummy ConfigMap exists in capi-system (no controller pods)
kubectl -n capi-system get all
kubectl -n capi-system get configmap capi-dummy-placeholder

# 3. Verify Rancher-bundled turtles is running
kubectl -n cattle-turtles-system get pods
# Should show: rancher-turtles-controller-manager with image rancher/turtles:v0.25.1

# 4. Verify CAPI core CRDs are installed (by turtles, not by Sylva's capi unit)
kubectl get crd clusters.cluster.x-k8s.io
kubectl get crd machinedeployments.cluster.x-k8s.io
# Labels should show: cluster.x-k8s.io/provider: cluster-api

# 5. Verify CAPIProvider core is ready
kubectl get capiproviders -A
# Should show: cattle-capi-system  cluster-api  core  cluster-api  v1.10.6  Ready

# 6. Verify Sylva's rancher-turtles unit is disabled
kubectl get helmrelease -A | grep turtles
# Should NOT show a Sylva-managed rancher-turtles HelmRelease

# 7. Check all kustomizations are healthy
kubectl get kustomization -A | grep -v True
# Should return no results (all True)
```

## Reverting

To go back to the default Sylva CAPI setup, remove all overrides from your `values.yaml`:

1. Remove the `source_templates.sylva-capi-dummy` section
2. Remove the `units.capi` override
3. Remove `units.rancher-turtles.enabled: false`
4. Remove the Rancher version pin if no longer needed
5. Uninstall the Rancher-bundled turtles chart from the Rancher UI
6. Run `./apply.sh` to reconcile

## Environment details

| Component | Version/Detail |
|-----------|---------------|
| Sylva | sylva-core (main branch) |
| Rancher | 2.13.1 (aligned with SUSE Edge v3.5) |
| Turtles (Rancher-bundled) | v0.25.1 (chart 108.0.1+up0.25.1) |
| Turtles (Sylva unit, disabled) | v0.24.4 |
| CAPI core (installed by turtles) | v1.10.6 |
| Infrastructure | CAPM3 (baremetal) |
| Bootstrap provider | CABPR (RKE2) |
| HA | Single node (`control_plane_replicas: 1`) |
| Dummy repo | https://github.com/andreacasini/project-sylva-suse-flavour.git |
