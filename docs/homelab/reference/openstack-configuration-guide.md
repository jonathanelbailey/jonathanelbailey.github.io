# Openstack Configuration Guide

## Installation

Run the following:

```shell title="Openstack Install"
sudo snap install openstack --channel 2023.1
sunbeam prepare-node-script | bash -x && newgrp snap_daemon
sunbeam cluster bootstrap --accept-defaults
sunbeam configure --openrc demo-openrc
```

You will then receive a series of prompts:

``` title="sumbeam configure Prompts"
Local or remote access to VMs [local/remote] (local): local
Populate OpenStack cloud with demo user, default images, flavors etc [y/n] (y): y
Username to use for access to OpenStack (demo): magicadmin
Password to use for access to OpenStack (mt********): <PASSWORD>
Network range to use for project network (192.168.122.0/24): 192.168.122.0/24
Enable ping and SSH access to instances? [y/n] (y): y
Writing openrc to demo-openrc ... done
```

## Configuration

- enabled nested virtualization
- enable openstack cli commands
- configure tls

## Safely Stopping Openstack

Run the following:

```shell
sudo snap stop microk8s
sudo snap stop openstack-hypervisor
```