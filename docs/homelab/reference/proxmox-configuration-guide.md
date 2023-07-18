# Proxmox VE Configuration Guide

## Install

## Resize Disk

```shell
lvremove /dev/pve/data
lvresize -l +100%FREE /dev/pve/root
resize2fs /dev/mapper/pve-root
```

Then update the disk to support all disk objects (vmdisk, snippets, etc)

## Create Cloud-init Template

```shell
wget https://cloud-images.ubuntu.com/releases/releases/jammy/release/ubuntu-22.04-server-cloudimg-amd64-disk-kvm.img
qm create 9000 --memory 2048 --net0 virtio,bridge=vmbr0 --scsihw virtio-scsi-pci
qm set 9000 --scsi0 local:0,import-from=/root/ubuntu-22.04-server-cloudimg-amd64-disk-kvm.img
qm set 9000 --ide2 local:cloudinit
qm set 9000 --boot order=scsi0
qm set 9000 --serial0 socket --vga serial0
qm template 9000
```

Also update the following:

1. Memory: 8GB
1. CPU: 4 cores
1. User Name: magicadmin
1. Set password
1. Add ssh key
1. Set network to DHCP

## To Clone a VM

1. Resize disk
1. then start VM