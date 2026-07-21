# Jellyfin Helm Chart

## Contents

- [Introduction](#introduction)
- [Installation](#prerequisites)
- [Accessing Jellyfin](#accessing-jellyfin)
- [Mounting media libraries](#mounting-media-libraries)
- [Testing the deployment](#testing-the-deployment)
- [Uninstalling](#uninstalling)
- [Configuration](#configuration)
- [Upgrading](#upgrading)

## Introduction

This chart deploys [Jellyfin](https://jellyfin.org/) into a
Kubernetes cluster. Jellyfin is a volunteer built, open-source media server.

## Prerequisites

- [Helm](https://helm.sh) 3
- A [Kubernetes](https://kubernetes.io/releases/) cluster with a dynamic
  provisioner (a default `StorageClass`), or pre-provisioned `PersistentVolume`s.
  The chart creates PVCs for config and cache via `volumeClaimTemplates`; set
  `persistence.config.storageClassName` / `persistence.cache.storageClassName`
  if you don't want the cluster default.
- An Ingress controller, only if you enable `ingress`.

## Installation

To get started, first add the helm repo and install Jellyfin:

```console
helm repo add jellyfin https://cfis.github.io/jellyfin-helm

helm upgrade --install jellyfin jellyfin/jellyfin --namespace jellyfin --create-namespace
```

## Accessing Jellyfin

The Service is headless (`clusterIP: None`) and no Ingress is created by default,
so the quickest way to reach a fresh install is a port-forward:

```console
kubectl -n jellyfin port-forward svc/jellyfin 8096:80
```

Then open <http://localhost:8096>. For permanent external access, enable and
configure the [ingress](#service--ingress) values instead.

> **Note:** DLNA relies on multicast discovery on the local network. Because the
> Service is a headless `ClusterIP`, DLNA auto-discovery will not work from your
> LAN out of the box; the port is exposed for in-cluster use only.

## Mounting media libraries

Your media (movies, music, etc.) is mounted via `mediaVolumes`. Each key becomes a
volume mounted at `/media/<key>` inside the pod. You can either have the chart
provision a PVC, or reference an existing claim.

```yaml
mediaVolumes:
  # Chart-provisioned PVC, mounted at /media/movies
  movies:
    storageClassName: ""
    accessModes:
      - ReadWriteOnce
    resources:
      requests:
        storage: 1Ti
  # Reuse a PVC you already created, mounted at /media/music
  music:
    existingClaim: my-music-pvc
```

Point Jellyfin's library at `/media/movies`, `/media/music`, etc. when adding
libraries in the web UI.

## Testing the deployment

The chart ships a Helm test that checks the web endpoint responds:

```console
helm test jellyfin --namespace jellyfin
```

## Uninstalling

```console
helm uninstall jellyfin --namespace jellyfin
```

This does not remove the PVCs. Delete them separately if you no longer need them:

```console
kubectl -n jellyfin delete pvc -l app.kubernetes.io/instance=jellyfin
```

## Configuration

The following tables list the configurable parameters and their defaults. Override
them with `--set key=value` or a custom `-f values.yaml`.

### General

| Key | Type | Default | Description |
|-----|------|---------|-------------|
| `nameOverride` | string | `""` | Override the chart name used in resource names. |
| `fullnameOverride` | string | `""` | Override the fully qualified release name. |

### Image

| Key | Type | Default | Description |
|-----|------|---------|-------------|
| `image.registry` | string | `docker.io` | Image registry. |
| `image.account` | string | `jellyfin` | Registry account/namespace. |
| `image.name` | string | `jellyfin` | Image repository name. |
| `image.tag` | string | _chart `appVersion`_ | Image tag. Defaults to the chart's `appVersion` when unset. |
| `image.pullPolicy` | string | `IfNotPresent` | Image pull policy. |
| `image.imagePullSecrets` | list | `[]` | Secrets used to pull the image. |

### Workload

| Key | Type | Default | Description |
|-----|------|---------|-------------|
| `podAnnotations` | object | `{}` | Additional annotations on the pod. |
| `podLabels` | object | `{}` | Additional labels on the pod. |
| `strategy` | object | `{}` | StatefulSet `updateStrategy` (`RollingUpdate` or `OnDelete`). |
| `nodeSelector` | object | `{kubernetes.io/os: linux}` | Node selector for pod assignment. |
| `affinity` | object | `{}` | Pod affinity rules. |
| `tolerations` | list | `[]` | Pod tolerations. |
| `topologySpreadConstraints` | list | `[]` | Topology spread constraints. |
| `resources.jellyfin.requests` | object | `{cpu: 500m, memory: 512Mi}` | Container resource requests. |
| `resources.jellyfin.limits` | object | `{cpu: "2", memory: 6Gi}` | Container resource limits. |
| `podSecurityContext` | object | runs as UID/GID `64710`, non-root, `RuntimeDefault` seccomp | Pod-level security context. |
| `containerSecurityContext` | object | drops all caps, read-only root FS, no privilege escalation, UID/GID `64710` | Container-level security context. |

### Storage

| Key | Type | Default | Description |
|-----|------|---------|-------------|
| `tmpfs.sizeLimit` | string | `10Mi` | Size of the in-memory `/tmp` volume (root FS is read-only). |
| `persistence.config.storageClassName` | string | `nil` (cluster default) | StorageClass for the config PVC. |
| `persistence.config.accessModes` | list | `[ReadWriteOnce]` | Access modes for the config PVC. |
| `persistence.config.resources.requests.storage` | string | `250Mi` | Size of the config PVC. |
| `persistence.cache.storageClassName` | string | `nil` (cluster default) | StorageClass for the cache PVC. |
| `persistence.cache.accessModes` | list | `[ReadWriteOnce]` | Access modes for the cache PVC. |
| `persistence.cache.resources.requests.storage` | string | `2Gi` | Size of the cache PVC. |
| `mediaVolumes` | object | `{}` | Additional PVCs mounted under `/media/<key>` (e.g. movies, music). |

### ServiceAccount

| Key | Type | Default | Description |
|-----|------|---------|-------------|
| `serviceAccount.create` | bool | `false` | Create a dedicated ServiceAccount. |
| `serviceAccount.annotations` | object | `{}` | Annotations for the ServiceAccount. |
| `serviceAccount.automount` | bool | `false` | Mount the SA token in the pod (Jellyfin doesn't use the K8s API). |

### Service & Ingress

| Key | Type | Default | Description |
|-----|------|---------|-------------|
| `service.webport` | int | `80` | Service port for the web UI. |
| `service.dlnaport` | int | `1900` | Service port for DLNA. |
| `ingress.enabled` | bool | `false` | Create an Ingress resource. |
| `ingress.className` | string | `""` | IngressClass name. |
| `ingress.annotations` | object | `{}` | Ingress annotations (e.g. cert-manager). |
| `ingress.host` | string | `jellyfin.local` | Hostname to route. |
| `ingress.path` | string | `/` | Ingress path. |
| `ingress.pathType` | string | `Prefix` | Ingress path type. |
| `ingress.tls` | bool | `false` | Enable TLS for `host`. |
| `ingress.tlsSecret` | string | `""` | TLS Secret name. Defaults to `<fullname>-tls` when empty. |

### NetworkPolicy

| Key | Type | Default | Description |
|-----|------|---------|-------------|
| `networkPolicy.enabled` | bool | `false` | Create a NetworkPolicy for the pod. |
| `networkPolicy.allowExternal` | bool | `true` | Allow inbound from any source. When `false`, only pods labelled `<fullname>-client: "true"`. |
| `networkPolicy.allowAllOutbound` | bool | `true` | Allow all egress. When `false`, only DNS plus `networkPolicy.egress`. |
| `networkPolicy.ingress` | list | `[]` | Extra ingress rules appended to the policy. |
| `networkPolicy.egress` | list | `[]` | Extra egress rules (used when `allowAllOutbound` is `false`). |

## Upgrading

### Upgrading to 0.5.0 from 0.4.x

This release bundles several fixes. Two of them need your attention before upgrading.

#### 1. The StatefulSet must be recreated once (selector fix)

Versions **0.4.2 and earlier** included the chart and app version labels
(`helm.sh/chart`, `app.kubernetes.io/version`) in the StatefulSet's
`spec.selector.matchLabels`. Because a StatefulSet's selector is immutable, any
release that changed the chart or Jellyfin version changed the selector and made
`helm upgrade` fail with:

```
cannot patch "jellyfin" with kind StatefulSet: StatefulSet.apps "jellyfin" is
invalid: spec: Forbidden: updates to statefulset spec for fields other than
'replicas', 'ordinals', 'template', 'updateStrategy', 'revisionHistoryLimit',
'persistentVolumeClaimRetentionPolicy' and 'minReadySeconds' are forbidden
```

This release uses only the stable `app.kubernetes.io/name` and
`app.kubernetes.io/instance` labels in the selector. Because the selector still
differs from what older releases deployed, you must recreate the StatefulSet
once. Delete it with `--cascade=orphan` so the running pod, PVCs, and your data
are left untouched — Helm then recreates the StatefulSet and re-adopts the pod:

```console
kubectl -n jellyfin delete statefulset jellyfin --cascade=orphan

helm upgrade --install jellyfin jellyfin/jellyfin --namespace jellyfin --create-namespace
```

Subsequent upgrades work normally with no manual steps.

#### 2. `persistence.*.enabled` has been removed

Earlier versions exposed `persistence.config.enabled` and
`persistence.cache.enabled` with a comment claiming that setting them to `false`
would fall back to an `emptyDir`. That never worked — a PVC was always created
regardless of the flag. The flag has been removed. Jellyfin's config and cache
are stateful and are always persisted.

If your values file sets either key, delete it. No behavior changes as a result.

#### 3. The `global.*` values have been removed

The `global:` block (`global.imageRegistry`, `global.imagePullSecrets`,
`global.commonLabels`, `global.debug`) has been removed. This chart is only used
standalone, so the umbrella-chart `global` layer was unused ceremony —
`imageRegistry` and `commonLabels`/`debug` were never wired to anything.

Image pull secrets now come from `image.imagePullSecrets` instead of
`global.imagePullSecrets`. If your values file sets `global.imagePullSecrets`,
move it to `image.imagePullSecrets`; any other `global.*` keys can be deleted.
