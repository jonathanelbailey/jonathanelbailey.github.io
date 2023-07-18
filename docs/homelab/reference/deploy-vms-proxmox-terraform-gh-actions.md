# Deploy VMs on Proxmox using Terraform and Github Actions

## Deploy Proxmox

## Deploy GH Runner VM

## Create Self-Hosted Runner in GitHub

## Configure Terraform User In Proxmox

token id:
terraform@pve!terraform
secret:
404cf760-80ba-4a7f-8155-c44cf8a69b20

## Configure Terraform

Install Terraform:

```shell
sudo apt-get update && sudo apt-get install -y gnupg software-properties-common
wget -O- https://apt.releases.hashicorp.com/gpg | \
gpg --dearmor | \
sudo tee /usr/share/keyrings/hashicorp-archive-keyring.gpg
gpg --no-default-keyring \
--keyring /usr/share/keyrings/hashicorp-archive-keyring.gpg \
--fingerprint
echo "deb [signed-by=/usr/share/keyrings/hashicorp-archive-keyring.gpg] \
https://apt.releases.hashicorp.com $(lsb_release -cs) main" | \
sudo tee /etc/apt/sources.list.d/hashicorp.list
sudo apt update
sudo apt-get install terraform

# verify install
terraform -help
```

## Configure Workflows