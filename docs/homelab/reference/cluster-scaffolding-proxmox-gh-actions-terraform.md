# Cluster scaffolding with Proxmox, GitHub Actions, and Terraform

Install microk8s:

```shell
sudo snap install microk8s --classic --channel=1.27
sudo usermod -a -G microk8s $USER
sudo chown -f -R $USER ~/.kube
microk8s status --wait-ready
```

