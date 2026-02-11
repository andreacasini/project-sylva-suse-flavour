# Dummy CAPI Unit for Sylva-Core (Rancher 2.13.1 + Turtles)

## Goal

Replace the CAPI core operator with a no-op dummy **only on the management cluster** (after pivot), so that `rancher-turtles` from Rancher 2.13.1 takes over CAPI operations. The bootstrap cluster keeps using the real CAPI controller as normal (Metal3/CAPM3 needs it to provision BareMetalHosts).

## How it works

- The **bootstrap cluster** uses the unmodified local sylva-core repo (including the real `kustomize-units/capi/`), so CAPI runs normally during bootstrap.
- The **management cluster** (after pivot) gets its Flux `GitRepository` source from the remote URL defined in `source_templates`. We add a **new source** pointing to your temporary public git repo that contains only the dummy kustomization. Then we override the `capi` unit to use that new source instead of `sylva-core`.
- The capi unit stays enabled (satisfying the core-package requirement) but deploys only a harmless ConfigMap.

## What you need to do

### Part 1: Create the temporary public git repo

Create a public git repo (e.g., `https://gitlab.com/your-user/sylva-capi-dummy.git`) with this structure:

```
sylva-capi-dummy/
└── dummy-capi/
    ├── kustomization.yaml
    └── configmap.yaml
```

#### File: `dummy-capi/kustomization.yaml`

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - configmap.yaml
```

#### File: `dummy-capi/configmap.yaml`

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: capi-dummy-placeholder
  namespace: capi-system
data:
  info: "CAPI core operator replaced by rancher-turtles from Rancher 2.13.1"
```

Push this to the `main` branch of your public repo.

> **Why `capi-system` namespace?** The real CAPI unit deploys into `capi-system`. Using the same namespace keeps things consistent. The namespace should already exist from the pivot (CAPI resources are moved there). If it doesn't exist yet when the management cluster reconciles, you can add a namespace resource to the kustomization (see "Optional" section below).

### Part 2: Modify your local `values.yaml` (environment-values)

Add these overrides to your environment's `values.yaml`:

```yaml
# Add a new Flux GitRepository source pointing to your dummy repo
source_templates:
  sylva-capi-dummy:
    kind: GitRepository
    spec:
      url: https://gitlab.com/your-user/sylva-capi-dummy.git
      ref:
        branch: main

# Override the capi unit to use the dummy source + path
units:
  capi:
    repo: sylva-capi-dummy
    kustomization_spec:
      path: ./dummy-capi
      wait: true
```

**That's it.** These values merge on top of the chart defaults. The capi unit remains `enabled: true` (its default for a core-component), but its `repo` and `kustomization_spec.path` are now redirected to your dummy.

### What happens during deployment

1. **Bootstrap cluster** — `bootstrap.sh` creates a kind cluster. Flux in the bootstrap cluster uses the **local** sylva-core content (pushed to a local gitea inside kind). Your local `kustomize-units/capi/` is **unmodified**, so the real CAPI core operator installs normally. Metal3/CAPM3 works, BareMetalHosts get provisioned, management cluster gets created.

2. **Management cluster (after pivot)** — Flux in the management cluster reconciles `sylva-units`. The `capi` unit now resolves to:
   - Source: `sylva-capi-dummy` GitRepository → your public repo
   - Path: `./dummy-capi/`
   - Result: only the dummy ConfigMap is applied in `capi-system`
   - The CAPI core controller pods are **not** deployed
   - `rancher-turtles` (installed by the `rancher` unit with Rancher 2.13.1) handles CAPI operations instead

3. The Flux Kustomization for the `capi` unit reports as **Ready** because it successfully applied the ConfigMap.

## Optional: Include namespace creation in the dummy

If `capi-system` namespace might not exist on the management cluster, add it:

#### Updated `dummy-capi/kustomization.yaml`

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - namespace.yaml
  - configmap.yaml
```

#### File: `dummy-capi/namespace.yaml`

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: capi-system
```

## Important notes

- **Do NOT modify** anything under `kustomize-units/capi/` in your local clone. The bootstrap cluster needs the real CAPI. The trick is purely in `values.yaml` overrides that only take effect on the management cluster's Flux GitRepository source.
- The bootstrap cluster's Flux uses a **local** git source (gitea in kind), which mirrors your local repo including `kustomize-units/capi/` as-is. But the management cluster's Flux uses the **remote** `source_templates` URLs. Since you've overridden `capi.repo` to point to `sylva-capi-dummy`, the management cluster fetches from your dummy repo instead.
- When you're done testing and want to go back to normal, just remove the `source_templates.sylva-capi-dummy` and `units.capi` overrides from your `values.yaml` and run `apply.sh`.
- Make sure your temporary public git repo is accessible from the management cluster nodes (consider proxies/firewall).

## Verification

After deployment, on the management cluster:

```bash
# Check that the capi Kustomization is Ready
kubectl get kustomization capi

# Check that only the dummy ConfigMap exists in capi-system (no controller pods)
kubectl -n capi-system get all
kubectl -n capi-system get configmap capi-dummy-placeholder

# Verify rancher-turtles is running
kubectl -n rancher-turtles-system get pods
```
