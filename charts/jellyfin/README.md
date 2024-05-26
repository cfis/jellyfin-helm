# Jellyfin Helm Chart

## Contents

- [Introduction](#introduction)
- [Installation](#prerequisites)

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
