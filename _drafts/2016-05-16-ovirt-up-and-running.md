---
title: "Ovirt Up and Running"
modified:
categories:
excerpt: 'Learn how to deploy Red Hat's open-source virtualization software, based on RHEV.'
tags:
  - homelab
  - centos
  - virtualization
header:
  overlay_image: 20151010_113344.jpg
  caption: "Photo credit: Jonathan Bailey"
date: 2016-05-16T21:41:40-05:00
---
{% include toc %}

<!--
BASH HISTORY:

4  fdisk
5  parted /dev/sdb
6  mkdir /mnt/storage
7  mount /dev/sdb1 /mnt/storage/
8  df -h
9   yum install http://plain.resources.ovirt.org/pub/yum-repo/ovirt-release36.rpm
10   yum install http://plain.resources.ovirt.org/pub/yum-repo/ovirt-release36.rpm -y
11  yum -y install ovirt-engine
12  engine-setup
13  vi /etc/exports
14  exportfs
15  systemctl start nfs.service
16  systemctl status nfs.service
17  reboot
18  systemctl status nfs.serv
19  systemctl restart nfs.service
20  cat /etc/exports
21  ls /mnt/storage/iso
22  ls /mnt/storage/
23  ls /mnt/
24  ls /mnt/storage/
25  df -h
26  partprobe
27  df -h
28  mount /dev/sdb1 /mnt/storage
29  df -h
30  ls /mnt/storage/iso
31  ls /mnt/storage/
32  systemctl restart nfs.service
33  journalctl -xe
34  chown 36:36 /mnt/storage/iso
35  mkdir /mnt/storage/data
36  mkdir /mnt/storage/import_export
37  chown 36:36 /mnt/storage/data/
38  chown 36:36 /mnt/storage/import_export/
39  cat /etc/exports
40  chmod 755 /mnt/storage/iso
41  chmod 755 /mnt/storage/data
42  chmod 755 /mnt/storage/import_export/
43  systemctl start nfs-server.service
44  cat /etc/fstab
45  vi /etc/fstab
46  reboot
47  df -h
48  firewall-cmd
49  systemctl
50  systemctl status
51  systemctl status | grep failed
52  systemctl status
53  systemctl status httpd.service
54  systemctl restart httpd.service
55  yum uninstall firewalld -y
56  yum remove firewalld -y
57  yum install firewalld -y
58  firewall-cmd --list-all
59  systemctl start firewalld.service
60  firewall-cmd --list-all
61  yum remove firewalld -y
62  systemctl status nfs-server.service
63  systemctl start nfs-server.service
64  journalctl -xe
65  exportfs -a
66  cat /etc/exports
67  systemctl start nfs-server.service
68  cat /etc/exports
69  exportfs -h
70  man exportfs
71  exportfs -r
72  man exportfs
73  exportfs -ra
74  exportfs -ua
75  cat /etc/exports
76  exportfs -a
77  systemctl start nfs.service
78  vi /etc/exports
79  exportfs -ua
80  systemctl start nfs.service
81  systemctl status nfs.service
82  systemctl status nfs.service -l
83  vi /etc/exports
84  systemctl status nfs.service -l
85  systemctl start nfs.service
86  engine-iso-uploader list
87  engine-iso-uploader list
88  engine-iso-uploader -i ISO_DOMAIN rhel-server-7.2-x86_64-dvd.iso
89  engine-iso-uploader upload -i ISO_DOMAIN rhel-server-7.2-x86_64-dvd.iso
90  ls /root
91  cp /root/anaconda-ks.cfg ~
92   pwd ~
93  cp /root/anaconda-ks.cfg /home/ovirtadmin/
94  ssh-keygen
95  mkdir /home/ovirtadmin/.ssh
96  ssh-keygen
97  su ovirtadmin
98  visudo
99  vi /etc/ssh/ssh_config
100  vi /etc/ssh/sshd_config
101  systemctl restart sshd.service
102  cat ~/.ssh
103  cat /.ssh/id_rsa
104  ls
105  ls .ssh
106  cat .ssh/id_rsa
107  cat .ssh/id_rsa.pub
108  rm .ssh/*
109  move id_rsa.pub .ssh/
110  cp id_rsa.pub .ssh/
111  rm id_rsa.pub
112  pwd
113  cd /home/ovirtadmin/
114  ls
115  cd .ssh/
116  ls
117  mkdir authorized_keys
118  cp id_rsa.pub authorized_keys/
119  rm id_rsa.pub
120  vi /etc/ssh/sshd_config
121  mkdir /.ssh
122  mkdir /.ssh/authorized_keys
123  cp .ssh/authorized_keys/id_rsa.pub /.ssh/authorized_keys/
124  systemctl restart sshd.service
125  vi /etc/ssh/sshd_config
126  cat .ssh/authorized_keys/id_rsa.pub
127  systemctl restart sshd.service
128  lscpu
129  lshw
130  yum install lshw -y
131  lshw -short
132  hwinfo --short
133  yum install hwinfo -y
134  hwinfo --short
135  inxi
136  yum install inxi -y
137  free -m
138  wget -q -O - http://linux.dell.com/repo/hardware/latest/bootstrap.cgi | bash
139  wget
140  yum install wget -y
141  wget -q -O - http://linux.dell.com/repo/hardware/latest/bootstrap.cgi | bash
142  yum install dell_ft_install
143  yum update -y
144  yum install dell_ft_install
145  yum install dell-system-update
146  dsu --inventory
147  history
  -->


## Introduction

This guide explains how to install Ovirt, a virtualization technology from Redhat that's the unstable release of RHEV.

### Why Ovirt?

I aim to use RHEV to replace my Hyper-V environment.  It's free, more feature-rich out of the box, and is more pleasant to use than VMWare, in my opinion.  While Hyper-V is nice, I am jumping to Ovirt mainly because it's what I use at work.

### Hardware Requirements

The Server that I'm deploying Ovirt to has the following specs:

|Name       |Value                                |
|-----------|-------------------------------------|
|Make       |Dell                                 |
|Model      |Poweredge 1900                       |
|Generation |10th                                 |
|CPU        |2 x L5320 Xeon 1.86 Quad Core        |
|RAM        |16GB PC2-5300 ECC                    |
|STORAGE    |2 x 160GB Mirrored, 3 x 1TB RAID 5   |
|RAID       |0,5                                  |
|UPS        |APC 1500                             |
|Ethernet   |3 x 1Gb Intel NIC                    |

Minimum requirements are far below this, but if you're ever interested in deploying a hypervisor environment in your own home -- and if you're even remotely technically skilled I implore that you get one.

check out ebay for super duper cheap rack servers around the $200 mark.  You're guaranteed to find a server with 2 quad core CPUs and at least 16GB of RAM.
{: .notice--success}

## Node Configuration

The first step to deploying any application is the configuration of the server, otherwise known as 'node config'.  The following steps must be completed before we start the deployment of Ovirt:
1. Centos 7/Fedora/RHEL 7 must first be installed.
2. (optional) configure SSH for your new server to secure it.
3. Run {% highlight bash %}yum -y update{% endhighlight %} on your machine to update it, then run {% highlight bash %}reboot{% endhighlight %}
4. Partition any drives and format the partitions for use, if needed.
5. Run {% highlight bash %}yum install nfs-utils nfs-utils-lib -y{% endhighlight %} to install NFS.

### Install OS

In my installation, I went with Centos 7 since it's a stable community version of RHEL.

### Secure with SSH

In order to secure with SSH, I decided to run the following commands on my workstation to get keys created:
**Note:** If you're not using a Linux console like Msysgit or Cygwin, you'll want to have access to the linux tools in your shell path.  If you're using git on your workstation, you can set a path to your `c:\program files\git\bin\` so you can have access to ssh tools.
{: .notice--info}
{% highlight bash %}
ssh-keygen -t rsa -b 4096
ssh-copy-id -i .ssh/id_rsa.pub testadmin@ovirt-test-machine.localdomain
{% endhighlight %}
**Warning!** using `ssh-copy-id` copies the key over the network.  It's not secure, so it mitigates this behavior by only allowing you to copy ONLY ONCE PER CONNECTION.
{: .notice--danger}

### Mount Virtual Drives

Since I had a RAID 5 virtual drive that needed to be mounted, I ran the following to find out the name of my drive:
{% highlight bash %}
ll /dev/sd*
{% endhighlight %}

And here was the output:
{% capture fig_img %}
![output]({{ base_path }}/images/sdb.png)
{% endcapture %}
I noted it was /dev/sdb, so from there, I ran the following:

{% highlight bash %}
parted /dev/sdb
  print                                                              # check to verify the drive is correct
  mkpart primary ext4                                                # create the partition.  Will prompt you for partition size
  mklabel data
  q                                                                  # quits parted shell
mkdir /mnt/storage                                                   # create folder to mount drive
vi /etc/fstab "LABEL=data  /mnt/storage  ext3  defaults  1 2"       # append the /etc/fstab file with a line for the new partition
partprobe                                                            # rescan for new partitions
mount /dev/sdb1 /mnt/storage                                         # mount the new partition
df -h                                                                # check the mount to see that it's registered

{% endhighlight %}

### Set up NFS

## Installing Ovirt

### Engine Installation

### Host installation

## Configuring Ovirt

### Create Your First Datacenter

### Configure Your Host

### Configuring Storage

### Configure Power Management

### Upload ISOs

### Creating a Template VM
