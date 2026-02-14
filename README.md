# Cluster API Provider Hetzner (CAPH) Helm Chart

Helm chart for deploying the [Cluster API Provider Hetzner (CAPH)](https://github.com/syself/cluster-api-provider-hetzner) as a k0rdent-managed infrastructure provider.

## Overview

This chart installs CAPH into a k0rdent management cluster, enabling Kubernetes cluster provisioning on Hetzner Cloud. It is deployed as a k0rdent `ProviderTemplate` and managed by the k0rdent lifecycle.

**Chart Version:** 0.0.26
**CAPH Version:** v1.0.7

## What This Chart Installs

| Resource | Kind | Description |
|----------|------|-------------|
| `hetzner` | InfrastructureProvider | CAPH controller with custom image (`ghcr.io/enopax/caph`) |
| `cluster-api-provider-hetzner` | ProviderInterface | Registers Hetzner as a k0rdent provider |
| `provider-interface-hetzner` | ClusterRole | RBAC for k0rdent to manage Hetzner resources |

## Usage

This chart is not installed directly. It is packaged and published to an OCI registry, then referenced by a k0rdent `ProviderTemplate`:

```bash
# Package
helm package .

# Publish to GHCR
helm push cluster-api-provider-hetzner-0.0.26.tgz oci://ghcr.io/enopax/charts

# Reference in ProviderTemplate manifest
```

Example `ProviderTemplate`:

```yaml
apiVersion: k0rdent.mirantis.com/v1alpha1
kind: ProviderTemplate
metadata:
  name: cluster-api-provider-hetzner-0-0-26
spec:
  helm:
    chartSpec:
      chart: cluster-api-provider-hetzner
      version: 0.0.26
      sourceRef:
        kind: HelmRepository
        name: enopax-charts
```

## Repository Structure

```
.
├── Chart.yaml                          # Chart metadata (v0.0.26)
└── templates/
    ├── provider.yaml                   # InfrastructureProvider (CAPH v1.0.7)
    ├── providerinterface.yaml          # ProviderInterface for k0rdent
    └── rbac.yaml                       # ClusterRole for Hetzner resources
```

## Version Compatibility

| Component | Version |
|-----------|---------|
| CAPH | v1.0.7 |
| CAPH Image | `ghcr.io/enopax/caph:v1.0.7-diagnostics` |
| Cluster API | v1beta1 |
| k0rdent API | v1beta1 / v1alpha2 |
| Target Namespace | `kcm-system` |

## Custom CAPH Image

This chart uses a custom CAPH image (`ghcr.io/enopax/caph`) instead of the upstream `ghcr.io/syself/caph-controller`. The custom image includes diagnostics flag support for improved debugging.

## Related Charts

| Chart | Repository | Description |
|-------|------------|-------------|
| [hetzner-standalone-cp](https://github.com/enopax/hetzner-standalone-cp) | `enopax/hetzner-standalone-cp` | k0s cluster with standalone control plane |
| [hetzner-hosted-cp](https://github.com/enopax/hetzner-hosted-cp) | `enopax/hetzner-hosted-cp` | k0s cluster with k0smotron hosted control plane |

## References

- [CAPH upstream](https://github.com/syself/cluster-api-provider-hetzner)
- [k0rdent documentation](https://docs.k0rdent.io/)
- [Hetzner Cloud](https://www.hetzner.com/cloud)
