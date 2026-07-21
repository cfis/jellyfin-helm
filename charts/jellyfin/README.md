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

Upgrades from **0.4.2 or earlier to 0.4.3+** require a one-time manual step.

Those older versions included the chart and app version labels
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

0.4.3 fixes this by using only the stable `app.kubernetes.io/name` and
`app.kubernetes.io/instance` labels in the selector. Because the selector still
differs from what older releases deployed, you must recreate the StatefulSet
once. Delete it with `--cascade=orphan` so the running pod, PVCs, and your data
are left untouched — Helm then recreates the StatefulSet and re-adopts the pod:

```console
kubectl -n jellyfin delete statefulset jellyfin --cascade=orphan

helm upgrade --install jellyfin jellyfin/jellyfin --namespace jellyfin --create-namespace
```

Subsequent upgrades work normally with no manual steps.
