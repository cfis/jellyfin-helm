# Jellyfin Helm Chart

## Contents

- [Introduction](#introduction)
- [Installation](#prerequisites)
- [Upgrading](#upgrading)

## Introduction

This chart deploys [Jellyfin](https://jellyfin.org/) into a
Kubernetes cluster. Jellyfin is a volunteer built, open-source media server.

## Prerequisites

- [Helm](https://helm.sh)
- A [Kubernetes](https://kubernetes.io/releases/) cluster with persistent storage 

## Installation

To get started, first add the helm repo and install Jellyfin:

```console
helm repo add jellyfin https://cfis.github.io/jellyfin-helm

helm upgrade --install jellyfin jellyfin/jellyfin --namespace jellyfin --create-namespace
```

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
