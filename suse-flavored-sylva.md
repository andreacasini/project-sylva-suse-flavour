# Aligning Sylva 1.6.x with SUSE Edge 3.5: Technical Progress Report

## Executive Summary

This effort aimed to align an open-source Sylva 1.6.x deployment with SUSE Edge 3.5, the commercial Telco Cloud offering that shares a similar technical stack. The primary focus was ensuring Helm charts and container images originate from the same sources as SUSE Edge, enabling consistency for support scenarios and knowledge transfer. 

**Status**: A fully functional management cluster has been deployed with charts and images aligned to SUSE Edge sources. Workload cluster deployment has been validated using the conventional Sylva workflow, with alignment work remaining as a straightforward next step.

**Key Contribution**: This work established a reusable methodology for unit migration and resolved the complex Rancher Turtles integration challenge, enabling peaceful coexistence between Sylva's CAPI provider management and Rancher's bundled Turtles operator.

---

## Operating System: SUSE Linux Micro 6.2

### Why SL Micro 6.2?

SUSE Edge 3.5 targets SL Micro as the deployment operating system. Aligning the OS layer ensures consistency with the commercial offering and validates Sylva's compatibility with SUSE's enterprise-grade minimal OS.

### Image Building Process

SL Micro images were built using a two-stage process:

1. **KIWI Base Image**: Boot a SL Micro 6.2 system and use the SUSE Edge kiwi-builder container to produce a raw base image
2. **EIB Customization**: Edge Image Builder (EIB) processes the base image, adding telco-specific packages (DPDK, dpdk-tools, pf-bb-config, tuned, cpupower, open-iscsi) and configuring kernel arguments, systemd services, and user credentials

The EIB definition sets `ignition.platform.id=openstack` for Metal3 compatibility and `net.ifnames=1` for predictable NIC naming.

### Key Findings & Workarounds

| Challenge | Root Cause | Solution Applied |
|-----------|-----------|------------------|
| Ignition format required | SL Micro uses Ignition configuration format, not cloud-init | `agent_config_format: ignition` in values.yaml |
| Static IP allocation | SL Micro doesn't work with Metal3-ipam dynamic allocation (known issue) | `ip_preallocations` for explicit per-host IP assignment |
| User creation via Sylva | `additionalUserData` users not reliably applied to SL Micro | Create users directly in EIB image definition |
| Cilium rke2-install.sh | Script contains unnecessary `sudo` that fails on minimal SL Micro | Manual fix (remove `sudo` from script) required |

### Longhorn Disk Provisioning

A positive finding: Longhorn disk provisioning works seamlessly with SL Micro. Sylva's Ignition code path handles disk setup via systemd units and node labels. No additional EIB scripts are needed beyond defining `longhorn_disk_config` in values.yaml.

---

## Project Context

**Sylva 1.6.x** is an open-source project providing GitOps-based Kubernetes cluster lifecycle management for bare metal deployments. It uses Flux for continuous reconciliation, Kustomize for unit composition, and supports multiple infrastructure providers (CAPM3, CAPV, CAPD).

**SUSE Edge 3.5** is a commercial Telco Cloud offering that includes hardened container images, enterprise support, and validated configurations. It shares technical foundations with Sylva but differs in deployment tooling, integration depth, and supported component versions.

**Alignment Philosophy**: Configuration drift between Sylva and SUSE Edge is acceptable. The priority is ensuring chart and image provenance matches SUSE Edge sources, enabling consistency for support scenarios while preserving Sylva's OSS flexibility.

---

## Alignment Status Summary

### Fully Aligned Units

The following units achieved full alignment with SUSE Edge 3.5 chart versions and container image registries:

| Unit | Chart Source | Image Registry | Notes |
|------|-------------|----------------|-------|
| MetalLB | OCI `registry.suse.com/edge/charts` | `registry.suse.com` | Three-block override pattern |
| Metal3 | OCI `registry.suse.com/edge/charts` | `registry.suse.com` | Flux-only (no bootstrap HelmChart) |
| Rancher | Prime chart repo | Built-in | Feature flags for Turtles enabled |
| Longhorn | Rancher charts repo (unchanged) | `registry.suse.com` | Version pin + registry override |
| Cilium | RKE2 charts repo (unchanged) | `registry.suse.com` | Version pin + registry override |
| Multus | RKE2 charts repo (unchanged) | `registry.suse.com` | Version pin + registry override |
| ingress-nginx | RKE2 charts repo (unchanged) | `registry.suse.com` | Version pin + registry override |
| metrics-server | RKE2 charts repo (unchanged) | `registry.suse.com` | Version pin + registry override |
| CoreDNS | RKE2 charts repo (unchanged) | `registry.suse.com` | Explicit image overrides (no `systemDefaultRegistry` support) |
| KubeVirt | OCI `registry.suse.com/edge/charts` | Chart defaults | No image override needed |
| KubeVirt CDI | OCI `registry.suse.com/edge/charts` | Chart defaults | Chart name mismatch handled |

### Architectural Constraints

| Unit | Status | Explanation |
|------|--------|-------------|
| vSphere CSI | Upstream registry | Kustomize unit pulling from GitHub; no registry override mechanism via values.yaml. Images are version-identical to SUSE Edge mirrors; functional equivalence achieved for connected environments. |
| local-path-provisioner | Upstream chart | SUSE Edge embeds this component in RKE2/k3s, no standalone chart exists. Images overridden to SUSE registry; chart source remains upstream GitRepository. |
| NeuVector | Custom units | Deep Sylva integration prevented direct override. Alternative units created for SUSE Edge alignment (see Technical Challenges section). |

---

## Key Technical Challenges

### 1. Rancher Turtles Integration (CAPI Architecture)

**Context**: Rancher 2.13.1 (bundled with SUSE Edge 3.5) includes Turtles v0.25.1, which installs CAPI core CRDs via a `CAPIProvider` custom resource. This differs fundamentally from Sylva's standalone `rancher-turtles` unit (v0.24.4), which does NOT install CAPI CRDs.

**Architecture Achieved**:

| Cluster Phase | CAPI Unit Behavior | Turtles | CAPI CRDs Installed By |
|---------------|--------------------|---------|-----------------------|
| Bootstrap | Real CAPI (sylva-core kustomization) | Not present | Real CAPI controller |
| Management (after pivot) | Dummy (placeholder ConfigMap only) | Rancher-bundled v0.25.1 | Turtles via CAPIProvider CR |

**Implementation Approach**:

Sylva's `_internal.mgmt_cluster` internal variable enables conditional behavior: it evaluates to `false` during bootstrap cluster operations and `true` after pivot to the management cluster. This variable was leveraged with GoTpl ternary expressions to switch the CAPI unit between two deployments:

```yaml
units:
  capi:
    repo: '{{ .Values._internal.mgmt_cluster | ternary "sylva-capi-dummy" "sylva-core" }}'
    kustomization_spec:
      path: '{{ .Values._internal.mgmt_cluster | ternary "./dummy-capi" "./kustomize-units/capi" }}'
```

A custom GitRepository was created containing a minimal kustomization that provisions:
- The `capi-system` namespace (expected by downstream units)
- A placeholder ConfigMap with informational content

This ensures the Flux Kustomization reports "Ready" status without deploying an actual CAPI controller, allowing Rancher's bundled Turtles to take over CAPI operations.

**Bootstrap Process Preserved**: The bootstrap cluster uses Sylva's standard CAPI provider unchanged. Metal3/CAPM3 provisions BareMetalHosts normally. Only the management cluster sees the architectural shift.

**Dependency Adjustments**: The `kyverno-policy-prevent-mgmt-cluster-delete` unit depends on CAPI CRDs being present. Since Turtles installs these asynchronously after Rancher starts, the initial deployment shows a timeout. Flux continues reconciling and the unit becomes Ready once CRDs are installed. This is cosmetic—the second run passes immediately.

### 2. NeuVector Deep Integration

**Challenge**: Sylva's NeuVector deployment is integrated with:
- Keycloak SSO (OIDC configuration managed by `neuvector-init` unit)
- Vault secrets (admin password, TLS certificates via external-secrets-operator)
- Kyverno `add-sylva-ca` ClusterPolicy (mutates controller pods to inject `sylva-ca.crt` volume)

**Why Direct Override Wasn't Possible**:

The Kyverno policy operates at Kubernetes admission time, injecting volumes into pods regardless of what the HelmRelease specifies. Setting `volumes: []` in HelmRelease values has no effect—the mutation happens after Helm renders templates. Disabling `neuvector-init` removes not only the Vault-managed secrets but also the namespace creation (handled conditionally by `namespace-defs` unit).

Additionally, Sylva's defaults set image repositories using upstream naming (`neuvector/controller`). The Rancher chart uses different naming (`rancher/neuvector-controller`). With `registry: registry.rancher.com`, these produce mismatched paths. Sylva's deep-merge behavior preserves conflicting defaults even when attempting to override.

**Solution**: Create custom units (`suse-neuvector-init`, `suse-neuvector-crd`, `suse-neuvector`) with entirely new names, bypassing Sylva defaults completely. This approach:
- Creates namespace with required PSA labels (`privileged`)
- Deploys CRD chart separately (Rancher chart requires this)
- Deploys main chart with clean SUSE Edge configuration
- Disables Flux drift detection (NeuVector rotates internal certificates)

**Trade-off**:
- **Gains**: SUSE Edge alignment—correct chart versions, SUSE registry images, Rancher SSO integration
- **Loses**: Sylva security integrations—Keycloak OIDC, Vault-managed secrets, Sylva CA certificate injection

This reflects different design priorities: Sylva's NeuVector is designed to integrate with Sylva's security infrastructure, while SUSE Edge's NeuVector uses Rancher's authentication. Both approaches are valid; alignment requires choosing one path.

### 3. CAPI Provider Management Coexistence

**Problem**: When both Sylva's Flux Kustomizations and Turtles' `CAPIProvider` custom resources manage the same infrastructure providers (capm3, metal3-ipam, cabpr), certificate reconciliation loops occur. Both controllers attempt to deploy identical resources in identical namespaces, causing:
- Certificate instability (repeated deletion/recreation)
- CA bundle mismatch (webhook configuration out of sync with server certificate)
- Bootstrap failures with `x509: certificate signed by unknown authority`

**Solution**:

In the `rancher-turtles-providers` chart values, disable providers that Sylva manages:

```yaml
providers:
  infrastructureMetal3:
    enabled: false   # Sylva manages capm3
  ipamMetal3:
    enabled: false   # Sylva manages metal3-ipam
  bootstrapRKE2:
    enabled: false   # Sylva manages cabpr bootstrap
  controlplaneRKE2:
    enabled: false   # Sylva manages cabpr controlplane
```

Additionally, patch Sylva's provider images to match versions Turtles expects:

| Provider | Sylva Default | Patched to Match Turtles |
|----------|---------------|-------------------------|
| capm3 | `quay.io/metal3-io/cluster-api-provider-metal3:v1.10.5` | `registry.suse.com/rancher/cluster-api-provider-metal3:v1.10.4` |
| metal3-ipam | `quay.io/metal3-io/ip-address-manager:v1.10.5` | `registry.rancher.com/rancher/ip-address-manager:v1.10.4` |

**Responsibility Split**:
| Responsibility | Owner |
|----------------|-------|
| Provider installation & upgrades | Sylva (Flux Kustomization) |
| Provider certificates | Sylva (cert-manager) |
| Cluster lifecycle (create, scale, upgrade, delete) | Turtles |
| Cluster import to Rancher | Turtles |

### 4. Chart Name Mismatches

The Sylva unit `kubevirt-cdi` maps to an OCI chart named simply `cdi`. Flux attempts to pull the chart name derived from the unit name, resulting in `oci://registry.suse.com/edge/charts/kubevirt-cdi` (which doesn't exist).

**Fix**: Two settings are required:
- `helm_chart_artifact_name: cdi` — controls Flux artifact naming
- `helmrelease_spec.chart.spec.chart: cdi` — sets HelmRelease chart reference

Both must be set; neither alone is sufficient.

---

## Methodology: Unit Migration Pattern

A reusable checklist emerged from this work:

1. **Find chart coordinates**: Registry URL, chart name, version tag (use `skopeo list-tags` for OCI)
2. **Set `helm_repo_url`**: OCI path without chart name (Sylva auto-appends unit name)
3. **Set version**: Use the `+` form (e.g., `305.0.1+up0.15.2`); Helm converts to OCI tag encoding
4. **Set image overrides**: Point to SUSE registry paths via `<unit>_helm_values` or `helmrelease_spec.values`
5. **Verify**: Check Flux HelmRelease status, chart version, pod images

### Gotchas Learned

| Issue | Explanation |
|-------|-------------|
| URL conventions differ | `units.<name>.helm_repo_url` excludes chart name; `cluster.helm_oci_url.<name>` includes it |
| Deep merge preservation | Sylva defaults persist; explicit overrides needed for each changed value |
| Bootstrap timing | `cluster.helm_oci_url` changes baked at cluster creation; redeploy required |
| RKE2 charts vary | Some support `global.systemDefaultRegistry`, others require explicit image overrides |

---

## Workload Cluster Status

A workload cluster was deployed successfully using Sylva's conventional workflow. The management cluster customization (CAPI architecture shift, chart/image alignment) did not block workload cluster creation.

Alignment of workload cluster charts and images to SUSE Edge sources is pending. Given the structural similarity between management and workload cluster configuration files, this is expected to be straightforward—applying similar overrides with workload-specific parameters.

---

## Lessons for the Community

### Sylva's Architecture Enables Customization

This work demonstrates Sylva's flexibility:
- Flux-based reconciliation allows fine-grained unit overrides
- `_internal.mgmt_cluster` variable enables bootstrap/management differentiation without core changes
- Unit templates system supports dependency injection and conditional enablement
- Deep merge preserves sensible defaults while allowing targeted overrides
- Kustomize patches can modify deployments without forking upstream manifests

### OSS vs Commercial Design Priorities

Full alignment isn't always possible or desirable. The NeuVector case illustrates that Sylva and SUSE Edge have different integration philosophies—Sylva emphasizes its security infrastructure (Keycloak, Vault), while SUSE Edge leverages Rancher's ecosystem. Both approaches serve their audiences.

Focus on provenance—chart and image sources—for support scenarios. Configuration differences are acceptable when functional equivalence is achieved. This work provides a template for future alignment efforts, whether targeting SUSE Edge releases or other commercial derivatives of the Sylva stack.

---

## References

| File | Content |
|------|---------|
| `units-override/*.md` | Detailed migration guides per component |
| `mgmt-cluster/vanilla-rke2-capm3/values.yaml` | Working management cluster configuration |
| `workload-cluster/my-rke2-capm3/values.yaml` | Workload cluster configuration (pending alignment) |
| `Images/README.md` | SL Micro 6.2 image building process |
| `units-override/rancher-turtles-investigation/*.md` | Detailed root cause analysis of certificate conflict |