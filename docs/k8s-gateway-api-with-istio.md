# Exploring the Kubernetes Gateway API with Istio

## Motivations

The Kubernetes Gateway API leverages ingress controllers to deploy and manage gateways using a unified API.  Essentially, the Gateway API can turn Istio into an ingress controller, providing a simple interface to deploy and manage multiple Istio gateways.

## Prerequisites

* ArgoCD
* Kubectl

## Clone the Project

Clone the [project](https://github.com/jonathanelbailey/homelab.catalog):

```shell
git clone https://github.com/jonathanelbailey/homelab.catalog.git
cd homelab.catalog/istio-k8s-gateway-api/
```

## Deploy Kuberentes Gateway API CRDs

This `Application` manifest will conifigure ArgoCD to deploy the Gateway API CRDs:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: k8s-gateway-api
  namespace: argocd
spec:
  destination:
    server: https://kubernetes.default.svc
  project: default
  source:
    path: config/crd
    repoURL: https://github.com/kubernetes-sigs/gateway-api/
    targetRevision: release-0.7     # Latest stable version at this time
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    retry:
      backoff:
        duration: 5s
        factor: 2
        maxDuration: 3m0s
      limit: 2
```

deploy the `Application` manifest using `kubectl`:

```shell
kubectl apply -f applications/k8s-gateway-api/k8s-gateway-crds.yaml
```

In ArgoCD the CRDs should sync successfully.

## Deploy Istio Components

Deploy `istio-base` and `istiod`:

```shell
kubectl apply -f applications/istio-base/istio-base-1-16-5.yaml
kubectl apply -f applications/istiod/istiod-1-16-5.yaml
```

!!! info "What about Istio Gateway?"
    If you are wondering why an Istio gateway was not deployed, it is because the Kubernetes Gateway API uses `istiod` as an ingress-controller.  An Istio gateway will spawn as soon as ArgoCD applies a `Gateway` resource.

## Deploy Bookinfo

Bookinfo's `Application` manifest has some interesting configurations to note:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: bookinfo
  namespace: argocd
spec:
  destination:
    namespace: bookinfo
    server: https://kubernetes.default.svc
  project: default
  # Use multiple sources
  sources:
    # pull Bookinfo from Istio GitHub Repository
    - directory:
        include: bookinfo.yaml
        jsonnet: {}
      path: samples/bookinfo/platform/kube
      repoURL: https://github.com/istio/istio.git
      targetRevision: HEAD
    # use bookinfo-gateway from another repository
    - directory:
        include: bookinfo-gateway.yaml
        jsonnet: {}
      path: istio-k8s-gateway-api/services/bookinfo
      repoURL: 'https://github.com/jonathanelbailey/homelab.catalog.git'
      targetRevision: HEAD
  syncPolicy:
    # Label bookinfo namespace with istio revision tag automatically
    managedNamespaceMetadata:
      labels:
        istio.io/rev: stable
    automated:
      prune: true
      selfHeal: true
    retry:
      backoff:
        duration: 5s
        factor: 2
        maxDuration: 3m0s
      limit: 2
    syncOptions:
      - CreateNamespace=true
```

ArgoCD provides powerful flexibility with the use of multiple sources.  We can now use the original bookinfo manifest straight from Istio, and then we can replace Bookinfo's gateway manifest with our own.  Now deploy `bookinfo`:

```shell
kubectl apply -f applications/bookinfo/bookinfo.yaml
```

## Validate Bookinfo-Gateway

Access `https://bookinfo.internal.magiccityit.com/productpage` from your browser.  A 404 response is returned.  However, when attempting to access `http://bookinfo.internal.magiccityit.com/productpage`, Product Page appears, albeit on an insecure connection.  Even though the `Gateway` has port 443 configured, and includes the correct TLS configuration, Istio cannot natively access secrets from another namespace.  To remedy this, we will add a `ReferenceGrant` to a namespace that does have the required certificate:

```yaml
apiVersion: gateway.networking.k8s.io/v1beta1
kind: ReferenceGrant
metadata:
  name: allow-bookinfo-gateways-to-ref-secrets
  namespace: istio-ingress
spec:
  from:
  - group: gateway.networking.k8s.io
    kind: Gateway
    namespace: bookinfo
  to:
  - group: ""
    kind: Secret
```

Once we add this manifest to `services/bookinfo/referencegrant.yaml` and push to the project's remote repository, ArgoCD will automatically pick up the change.  If we try again, we will receive a sucessful response.