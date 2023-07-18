# Homelab Kubernetes Configuration Guide

## Installation

Run the following:

```shell
sudo usermod -a -G microk8s magicadmin
sudo chown -R magicadmin ~/.kube

microk8s enable hostpath-storage metallb:10.1.0.1-10.1.0.254
microk8s config
```

Copy the output of `microk8s config` to your `~/.kube/config` and add it to OpenLens

## Deploy ArgoCD

From your local terminal, run the following:

```shell
helm repo add argo https://argoproj.github.io/argo-helm
helm repo update

helm upgrade --install argocd argo/argo-cd -n argo --create-namespace --atomic --cleanup-on-fail --wait -f values.yaml
```

Then download the `argocd` CLI tool:

```shell
curl -sSL -o argocd-linux-amd64 https://github.com/argoproj/argo-cd/releases/latest/download/argocd-linux-amd64
sudo install -m 555 argocd-linux-amd64 /usr/local/bin/argocd
rm argocd-linux-amd64
```

And finally, add `argocd-manager` to the cluster:

```shell
argocd cluster add microk8s
```