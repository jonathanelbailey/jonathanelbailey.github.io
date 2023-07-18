# Exploring Istio's Ambient Mesh

## Overview

## Prerequisites

1. kubectl
1. istioctl
1. kind

!!! warning
    Ambient Mesh is incompatible with [calico](https://github.com/istio/istio/issues/40973).  `Kind` is the only supported method of evaluating ambient mesh for local testing.
    However, EKS is also supported fully.

## Deployment

These steps are based on the [Istio Getting Started Guide](https://istio.io/latest/docs/ops/ambient/getting-started/).

1. Download the latest [Istio 1.18](https://github.com/istio/istio/releases/download/1.18.0/istio-1.18.0-linux-amd64.tar.gz) release.
1. Install `kind`:
   ```shell
   [ $(uname -m) = x86_64 ] && curl -Lo ./kind https://kind.sigs.k8s.io/dl/v0.20.0/kind-linux-amd64
   chmod +x ./kind
   sudo mv ./kind /usr/local/bin/kind
   ```
1. Deploy a cluster using `kind`:
   ```shell
   kind create cluster --config=- <<EOF
   kind: Cluster
   apiVersion: kind.x-k8s.io/v1alpha4
   name: ambient
   nodes:
   - role: control-plane
   - role: worker
   - role: worker
   EOF
   ```
1. Install the Kubernetes Gateway CRDs:
   ```shell
   kubectl get crd gateways.gateway.networking.k8s.io &> /dev/null || \
       { kubectl kustomize "github.com/kubernetes-sigs/gateway-api/config/crd/experimental?ref=v0.6.1" | kubectl apply -f -; }
   ```
1. Install Ambient Mesh:
   ```shell
   istioctl install --set profile=ambient --skip-confirmation
   ```
1. Install bookinfo

## Managing Via ArgoCD



## Canary Upgrades