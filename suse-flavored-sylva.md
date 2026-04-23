# Aligning Sylva 1.6.x with SUSE Edge Telco Cloud 3.5: Technical Progress Report

## Executive Summary

This effort aimed to align an open-source Sylva 1.6.x deployment with SUSE Edge Telco Cloud 3.5, the commercial Telco Cloud offering that shares a similar technical stack. The primary focus was ensuring Helm charts and container images originate from the same sources as SUSE Edge Telco Cloud, enabling consistency for support scenarios and knowledge transfer. Configuration drift between the two deployments is acceptable—the priority is provenance alignment, not identical configurations.

**Status**: A fully functional management cluster has been deployed with charts and images aligned to SUSE Edge Telco Cloud sources. Workload cluster deployment has been validated using the conventional Sylva workflow, with alignment work remaining as a straightforward next step.

**Key Contribution**: This work established a reusable methodology for unit migration and resolved the complex Rancher Turtles integration challenge, enabling peaceful coexistence between Sylva's CAPI provider management and Rancher's bundled Turtles operator.

---

## Operating System: SUSE Linux Micro 6.2

### Why SL Micro 6.2?

SUSE Edge Telco Cloud 3.5 targets SL Micro as the deployment operating system. Aligning the OS layer ensures consistency with the commercial offering and validates Sylva's compatibility with SUSE's enterprise-grade minimal OS.

### Image Building Process

SL Micro images were built using a two-stage process:

1. **KIWI Base Image**: Boot a SL Micro 6.2 system and use the SUSE Edge Telco Cloud kiwi-builder container (`registry.suse.com/edge/3.5/kiwi-builder:10.2.29.1`) to produce a raw base image
2. **EIB Customization**: Edge Image Builder (`registry.suse.com/edge/3.4/edge-image-builder:1.3.0`) processes the base image, adding telco-specific packages (DPDK, dpdk-tools, pf-bb-config, tuned, cpupower, open-iscsi, jq) and configuring kernel arguments, systemd services, and user credentials

The EIB definition sets `ignition.platform.id=openstack` for Metal3 compatibility and `net.ifnames=1` for predictable NIC naming.

### Key Findings & Workarounds

| Challenge | Root Cause | Solution Applied |
|-----------|-----------|------------------|
| Ignition format required | SL Micro uses Ignition configuration format, not cloud-init | `cluster.agent_config_format: ignition` in values.yaml |
| Static IP allocation | SL Micro doesn't work with Metal3-ipam dynamic allocation (known issue) | `ip_preallocations` for explicit per-host IP assignment (see below) |
| User creation via Sylva | `additionalUserData` users not reliably applied to SL Micro | Create users directly in EIB image definition |
| Cilium rke2-install.sh | Script contains unnecessary `sudo` that fails on minimal SL Micro | Manual fix (remove `sudo` from 'rke2-install.sh' script) required |

### Static IP Allocation: ip_preallocations Mechanism

Since Metal3-ipam's dynamic IP allocation doesn't function correctly with SL Micro, explicit IP addresses must be pre-allocated for each BareMetalHost. This is configured per-host in the `baremetal_hosts` section:

```yaml
baremetal_hosts:
  mgmt-server:
    ip_preallocations:
      primary: 10.160.1.32      # IP for the primary (production) network interface
      provisioning: 10.150.1.110 # IP for the provisioning (Metal3) network interface
```

Each host requires both a `primary` IP (used for cluster communication) and a `provisioning` IP (used during Metal3 provisioning). The network pool ranges defined in `capm3.networks` (start/end) are effectively bypassed—the IPs come from those ranges but are assigned statically rather than dynamically.

### Longhorn Disk Provisioning

A positive finding: Longhorn disk provisioning works seamlessly with SL Micro. Sylva's Ignition code path handles disk setup via systemd units and node labels. No additional EIB scripts are needed beyond defining `longhorn_disk_config` in values.yaml.

---

## Alignment Status Summary

### Fully Aligned Units

The following units achieved full alignment with SUSE Edge Telco Cloud 3.5 chart versions and container image registries. Configuration extracted from `mgmt-cluster/vanilla-rke2-capm3/values.yaml`:

| Unit | Chart Source Override | Version Override | Image Registry Override | Notes |
|------|----------------------|------------------|------------------------|-------|
| MetalLB | `oci://registry.suse.com/edge/charts` | `305.0.1+up0.15.2` | `registry.suse.com/edge/3.5/*` | Bootstrap (`helm_oci_url`) + Flux + image overrides |
| Metal3 | `oci://registry.suse.com/edge/charts` | `305.0.21+up0.13.0` | `registry.suse.com/edge/3.5/*` | No bootstrap component (Flux-only) |
| Rancher | `https://charts.rancher.com/server-charts/prime` | `2.13.1` | Feature flags override | Prime chart repo, Turtles enabled |
| Longhorn | (unchanged: Rancher repo) | `107.2.0+up1.10.1` | `registry.suse.com` via `systemDefaultRegistry` | OSS charts used; not AppCo catalog |
| Longhorn CRD | (unchanged: Rancher repo) | `107.2.0+up1.10.1` | (none - CRD chart) | Version pin only |
| Cilium | (unchanged: RKE2 repo) | `1.18.300` | `registry.suse.com` via `systemDefaultRegistry` | Also `cluster.coredns.helm_values` path |
| Multus | (unchanged: RKE2 repo) | `v4.2.300` | `registry.suse.com` via `systemDefaultRegistry` | Version pin + registry |
| ingress-nginx | (unchanged: RKE2 repo) | `4.13.400` | `registry.suse.com` via `systemDefaultRegistry` | Version pin + registry |
| metrics-server | (unchanged: RKE2 repo) | `3.13.002` | `registry.suse.com` via `systemDefaultRegistry` | Version pin + registry |
| CoreDNS | (unchanged: RKE2 repo) | `1.44.300` | Explicit `repository`/`tag` in `cluster.coredns.helm_values` | No `systemDefaultRegistry` support in chart |
| KubeVirt | `oci://registry.suse.com/edge/charts` | `305.0.1+up0.6.0` | (chart defaults) | Both chart source AND version override |
| KubeVirt CDI | `oci://registry.suse.com/edge/charts` | `305.0.1+up0.6.0` | (chart defaults) | Chart name mismatch + both overrides |
| local-path-provisioner | (unchanged: upstream GitRepository) | (none) | `registry.suse.com/rancher/*` | Image override only; no SUSE chart exists |

### Why Cluster-Level Configuration Is Required

Some overrides must be placed in the `cluster` configuration block rather than the `units` section:

**MetalLB (`cluster.helm_oci_url.metallb`)**: Sylva deploys MetalLB via two mechanisms—a bootstrap-time HelmChart baked into RKE2 node manifests, and a Flux HelmRelease for post-pivot lifecycle. The `helm_oci_url` setting controls the bootstrap component, which is created at cluster deployment time and cannot be modified afterward without redeploying.

**CoreDNS (`cluster.coredns.helm_values`)**: The `rke2-coredns` chart does not support the `global.systemDefaultRegistry` mechanism used by other RKE2 charts. Image registry overrides must be specified explicitly via `image.repository` and `image.tag` fields, and the values path is `cluster.coredns.helm_values` (not under `units.coredns`).

### Longhorn: OSS Charts vs AppCo Catalog

SUSE Edge Telco Cloud 3.5 ships "SUSE Storage" via the Rancher AppCo (Application Collection) catalog, which requires a subscription account. For this OSS Sylva deployment, the standard Rancher charts from `charts.rancher.io` were used instead, combined with SUSE registry images via `systemDefaultRegistry`. This achieves functional alignment without requiring commercial licensing.

This approach is open for discussion: it provides SUSE-hardened images with OSS chart management, but does not use the exact SUSE Edge Telco Cloud deployment mechanism.

### Architectural Constraints

| Unit | Status | Explanation |
|------|--------|-------------|
| vSphere CSI | Not tested | Documented in `units-override/vsphere-csi-driver.md` but not validated in this deployment. Kustomize unit; no registry override mechanism via values.yaml. |
| local-path-provisioner | Image-only alignment | SUSE Edge Telco Cloud embeds this in RKE2/k3s; no standalone Helm chart exists in SUSE Edge Telco Cloud catalog. Chart source remains upstream GitRepository. |

---

## Key Technical Challenges

### 1. Rancher Turtles Integration (CAPI Architecture)

**Context**: Rancher 2.13.1 (bundled with SUSE Edge Telco Cloud 3.5) includes Turtles v0.25.1, which installs CAPI core CRDs via a `CAPIProvider` custom resource. This differs fundamentally from Sylva's standalone `rancher-turtles` unit (v0.24.4), which does NOT install CAPI CRDs.

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

**Dependency Adjustment**: The `kyverno-policy-prevent-mgmt-cluster-delete` unit originally depended on CAPI CRDs being present. Since Turtles installs these asynchronously after Rancher starts, the unit's dependency was changed:

```yaml
kyverno-policy-prevent-mgmt-cluster-delete:
  depends_on:
    rancher: true
```

This ensures the Kyverno policy waits for Rancher (which installs Turtles, which installs CAPI CRDs), resolving the timing issue. The timeout observed during initial deployment is no longer present.

### 2. CAPI Provider Management Coexistence

**Problem**: When both Sylva's Flux Kustomizations and Turtles' `CAPIProvider` custom resources manage the same infrastructure providers (capm3, metal3-ipam, cabpr), certificate reconciliation loops occur. Both controllers attempt to deploy identical resources in identical namespaces, causing:
- Certificate instability (repeated deletion/recreation)
- CA bundle mismatch (webhook configuration out of sync with server certificate)
- Bootstrap failures with `x509: certificate signed by unknown authority`

**Solution**:

A custom unit `suse-rancher-turtles-providers` was created to deploy the Rancher Turtles providers chart with overlapping providers disabled:

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

Additionally, Sylva's provider images were patched via Kustomize `_patches` to match versions Turtles expects:

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

**Note**: The `suse-rancher-turtles-providers` unit is currently disabled (`enabled: false`) in the working configuration. Sylva's native provider units are active, and Rancher's bundled Turtles manages cluster lifecycle without additional provider configuration.

### 3. NeuVector: Two Possible Approaches

**Challenge**: Sylva's NeuVector deployment is integrated with:
- Keycloak SSO (OIDC configuration managed by `neuvector-init` unit)
- Vault secrets (admin password, TLS certificates via external-secrets-operator)
- Kyverno `add-sylva-ca` ClusterPolicy (mutates controller pods to inject `sylva-ca.crt` volume)

**Why SUSE Edge Telco Cloud Alignment Is Complex**:

The Kyverno policy operates at Kubernetes admission time, injecting volumes into pods regardless of what the HelmRelease specifies. Setting `volumes: []` in HelmRelease values has no effect—the mutation happens after Helm renders templates. Additionally, Sylva's defaults use upstream image naming (`neuvector/controller`) while the Rancher chart uses different naming (`rancher/neuvector-controller`), producing mismatched registry paths.

**Approach A: Leave Sylva's Native NeuVector Unchanged** (current deployment)

The working configuration uses Sylva's default NeuVector stack:

```yaml
units:
  neuvector-init:
    enabled: true
  neuvector:
    enabled: true
```

This preserves Sylva's security integrations (Keycloak SSO, Vault secrets, Sylva CA injection) but does not achieve SUSE Edge Telco Cloud alignment for NeuVector. This approach is functional and suitable for deployments where Sylva's security infrastructure is valued.

**Approach B: Custom Units for SUSE Edge Telco Cloud Alignment** (documented but not deployed)

Custom units (`suse-neuvector-init`, `suse-neuvector-crd`, `suse-neuvector`) can be created with entirely new names, bypassing Sylva defaults completely. This approach:
- Creates namespace with required PSA labels (`privileged`)
- Deploys CRD chart separately (Rancher chart requires this)
- Deploys main chart with clean SUSE Edge Telco Cloud configuration
- Disables Flux drift detection (NeuVector rotates internal certificates)

**Trade-off**:
- **Gains**: SUSE Edge Telco Cloud alignment—correct chart versions, SUSE registry images, Rancher SSO integration
- **Loses**: Sylva security integrations—Keycloak OIDC, Vault-managed secrets, Sylva CA certificate injection

This reflects different design priorities: Sylva's NeuVector integrates with Sylva's security infrastructure, while SUSE Edge Telco Cloud's NeuVector uses Rancher's authentication. Both approaches are valid; alignment requires choosing one path. The choice depends on deployment priorities and is open for community discussion.

### 4. Chart Name Mismatches

The Sylva unit `kubevirt-cdi` maps to an OCI chart named simply `cdi`. Flux attempts to pull the chart name derived from the unit name, resulting in `oci://registry.suse.com/edge/charts/kubevirt-cdi` (which doesn't exist).

**Fix**: Two settings are required:
- `helm_chart_artifact_name: cdi` — controls Flux artifact naming
- `helmrelease_spec.chart.spec.chart: cdi` — sets HelmRelease chart reference

Both must be set; neither alone is sufficient.

---

## Methodology: Unit Migration Pattern

A reusable checklist emerged from this work:

1. **Find chart coordinates**: Registry URL, chart name, version tag
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
| RKE2 charts vary | Some support `global.systemDefaultRegistry`, others require explicit image overrides (CoreDNS) |

---

## Workload Cluster Status

A workload cluster was deployed successfully using Sylva's conventional workflow. The management cluster customization (CAPI architecture shift, chart/image alignment) did not block workload cluster creation.

Alignment of workload cluster charts and images to SUSE Edge Telco Cloud sources is pending. Given the structural similarity between management and workload cluster configuration files, this is expected to be straightforward—applying similar overrides with workload-specific parameters.

---

## Lessons for the Community

### Sylva's Architecture Enables Customization

This work demonstrates Sylva's flexibility:
- Flux-based reconciliation allows fine-grained unit overrides
- `_internal.mgmt_cluster` variable enables bootstrap/management differentiation without core changes
- Unit templates system supports dependency injection and conditional enablement
- Deep merge preserves sensible defaults while allowing targeted overrides
- Kustomize `_patches` can modify deployments without forking upstream manifests

### OSS vs Commercial Design Priorities

Full alignment isn't always possible or desirable. The NeuVector case illustrates that Sylva and SUSE Edge Telco Cloud have different integration philosophies—Sylva emphasizes its security infrastructure (Keycloak, Vault), while SUSE Edge Telco Cloud leverages Rancher's ecosystem. Both approaches serve their audiences.

Focus on provenance—chart and image sources—for support scenarios. Configuration differences are acceptable when functional equivalence is achieved. This work provides a template for future alignment efforts, whether targeting SUSE Edge Telco Cloud releases or other commercial derivatives of the Sylva stack.

---

## References

| File | Content |
|------|---------|
| `mgmt-cluster/vanilla-rke2-capm3/values.yaml` | Working management cluster configuration |
| `workload-cluster/my-rke2-capm3/values.yaml` | Workload cluster configuration (pending alignment) |
| `units-override/*.md` | Detailed migration guides per component |
| `units-override/rancher-turtles-investigation/*.md` | Detailed root cause analysis of certificate conflict |
| `Images/README.md` | SL Micro 6.2 image building process |
