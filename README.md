# Cluster API Provider Hetzner (CAPH) Helm Chart

Helm chart for deploying the [Cluster API Provider Hetzner (CAPH)](https://github.com/syself/cluster-api-provider-hetzner) as a k0rdent-managed infrastructure provider.

## Overview

This chart installs CAPH into a k0rdent management cluster, enabling Kubernetes cluster provisioning on Hetzner Cloud. It is deployed as a k0rdent `ProviderTemplate` and managed by the k0rdent lifecycle.

**Chart Version:** 0.0.28
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
  name: cluster-api-provider-hetzner-0-0-28
spec:
  helm:
    chartSpec:
      chart: cluster-api-provider-hetzner
      version: 0.0.28
      sourceRef:
        kind: HelmRepository
        name: enopax-charts
```

## Advanced Usage (k0rdent + flux)

### 1. Package and publish the chart

The chart is hosted as an OCI package on `ghcr.io/enopax/charts`. To build and push a new version:

```bash
# Authenticate with a token that has write:packages scope
gh auth token | helm registry login ghcr.io --username <your-github-user> --password-stdin

helm package .
helm push cluster-api-provider-hetzner-<version>.tgz oci://ghcr.io/enopax/charts
```

---

### 2. Create a pull secret in the management cluster

The chart package is private. Create a Kubernetes `docker-registry` secret so Flux can pull it:

```bash
GH_TOKEN=$(gh auth token)
kubectl create secret docker-registry ghcr-enopax-charts \
  --docker-server=ghcr.io \
  --docker-username=<your-github-user> \
  --docker-password="${GH_TOKEN}" \
  --namespace=kcm-system

# Required: the Flux source-controller only watches resources with this label
kubectl label secret ghcr-enopax-charts \
  -n kcm-system k0rdent.mirantis.com/managed=true
```

> **Important:** The source-controller in k0rdent is started with
> `--watch-label-selector=k0rdent.mirantis.com/managed=true`. Any Flux resource
> (`HelmRepository`, `HelmChart`, `Secret`) that lacks this label will be silently
> ignored, causing a `"not found"` error at reconciliation time.

---

### 3. Create the Flux HelmRepository

```bash
kubectl apply -f - <<'EOF'
apiVersion: source.toolkit.fluxcd.io/v1
kind: HelmRepository
metadata:
  name: enopax-charts
  namespace: kcm-system
  labels:
    k0rdent.mirantis.com/managed: "true"
spec:
  type: oci
  url: oci://ghcr.io/enopax/charts
  interval: 10m0s
  secretRef:
    name: ghcr-enopax-charts
EOF
```

---

### 4. Create the ProviderTemplate

The `ProviderTemplate` causes k0rdent to create a `HelmChart` object and validates the chart against the k0rdent schema. Replace `<version>` with the chart version (e.g. `0.0.27` → name suffix `0-0-27`):

```bash
kubectl apply -f - <<'EOF'
apiVersion: k0rdent.mirantis.com/v1beta1
kind: ProviderTemplate
metadata:
  name: cluster-api-provider-hetzner-0-0-27
  namespace: kcm-system
spec:
  helm:
    chartSpec:
      chart: cluster-api-provider-hetzner
      version: 0.0.27
      interval: 10m0s
      reconcileStrategy: ChartVersion
      sourceRef:
        kind: HelmRepository
        name: enopax-charts
EOF
```

Wait for the `ProviderTemplate` to become valid:

```bash
kubectl get providertemplate cluster-api-provider-hetzner-0-0-27 -n kcm-system
# Expected: valid: true, providers: [infrastructure-hetzner]
```

---

### 5. Register the provider in Management

The k0rdent Management controller owns all `HelmRelease` objects labeled `k0rdent.mirantis.com/managed=true`. Creating a standalone HelmRelease will be garbage-collected unless it is registered in `Management.spec.providers`.

Patch the Management object to add hetzner with the explicit template name:

```bash
kubectl patch management kcm -n kcm-system --type=json \
  -p='[{"op":"add","path":"/spec/providers/-","value":{"name":"cluster-api-provider-hetzner","template":"cluster-api-provider-hetzner-0-0-27"}}]'
```

The Management controller will then create the `HelmRelease` and install the chart.

Wait for everything to become ready:

```bash
kubectl wait management kcm -n kcm-system --for=condition=Ready --timeout=300s
kubectl get infrastructureprovider hetzner -n kcm-system
# Expected: READY=True, INSTALLEDVERSION=v1.0.7
```

---

## Providing the Hetzner token

> **Sources consulted:**
> - [k0rdent Credentials Process](https://docs.k0rdent.io/latest/admin/access/credentials/credentials-process/) — defines the Secret → Credential → ClusterDeployment chain
> - [CAPH preparation guide](https://github.com/syself/cluster-api-provider-hetzner/blob/main/docs/caph/01-getting-started/03-preparation.md) — documents the Hetzner secret format
> - `templates/providerinterface.yaml` in this chart — lists bare `Secret` as the valid identity type for k0rdent credential resolution

Unlike AWS (which requires global IAM credentials for the controller itself), **CAPH v1.x uses per-cluster credentials only**. All credentials are provided through k0rdent's `Credential` system and referenced from each `HetznerCluster` via `spec.hetznerSecretRef`.

Because CAPH v1.0.7 has no `HetznerClusterIdentity` CRD, the `ProviderInterface` in this chart allows a plain `Secret` as the identity reference directly.

> **Why no `HetznerClusterIdentity` CRD exists:** The original Hetzner CAPI project (`hetznercloud/cluster-api-provider-hcloud`), which had this CRD, was archived. Syself wrote a fresh implementation that simplified credentials to a plain Secret referenced directly by `HetznerCluster.spec.hetznerSecretRef`. The identity indirection layer was not carried over.

> **Note on the secret key:** `HetznerCluster.spec.hetznerSecretRef.key.hcloudToken` is a CRD field that holds the **name of the key** in the secret. Its default is `hcloud-token`. Use the default unless you have a specific reason to override it.

```bash
# Step 1 — create the credential secret
kubectl apply -f - <<'EOF'
apiVersion: v1
kind: Secret
metadata:
  name: hetzner-cluster-identity-secret
  namespace: kcm-system
type: Opaque
stringData:
  hcloud-token: "<your-hcloud-token>"
EOF

# Step 2 — create the k0rdent Credential referencing the secret directly
# (plain Secret is registered as a valid identity type in the ProviderInterface)
kubectl apply -f - <<'EOF'
apiVersion: k0rdent.mirantis.com/v1beta1
kind: Credential
metadata:
  name: hetzner-credential
  namespace: kcm-system
spec:
  description: "Hetzner Cloud API token"
  identityRef:
    apiVersion: v1
    kind: Secret
    name: hetzner-cluster-identity-secret
    namespace: kcm-system
EOF
```

Verify the Credential is accepted:

```bash
kubectl get credential hetzner-credential -n kcm-system
# Expected: READY=True
```

Reference the Credential in your `ClusterDeployment` (using the default key name `hcloud-token`):

```yaml
spec:
  credential: hetzner-credential
  config:
    hetznerSecretRef:
      name: hetzner-cluster-identity-secret
      # key.hcloudToken defaults to "hcloud-token" — omit if using the default
```

> **Important:** k0rdent does **not** auto-inject the secret name into `HetznerCluster.spec.hetznerSecretRef`. The `Credential` only handles cross-namespace distribution of the identity object. You must explicitly set `spec.config.hetznerSecretRef.name` in your `ClusterDeployment` to match the secret name above.

---

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
