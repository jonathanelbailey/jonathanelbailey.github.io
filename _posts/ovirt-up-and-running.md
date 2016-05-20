---
title: "Ovirt Up and Running"
modified:
categories:
excerpt: "Learn how to deploy Red Hat's open-source virtualization software, based on RHEV."
tags:
  - homelab
  - centos
  - virtualization
header:
  overlay_image: the-black-crowes-overlay.jpg
  caption: "Photo credit: Jonathan Bailey"
date: 2016-05-19T17:41:40-05:00
---

{% include toc %}

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

Next, we'll need to ensure that NFS is properly configured.  If you're not familiar with NFS, it's a service that shares folder on the server to client host.  To set up the server, first install the packages:

{% highlight bash %}
yum install nfs-utils nfs-utils-lib
{% endhighlight %}

Then, create your folders.  I created mine in my mounted partition:

{% highlight bash %}
mkdir /mnt/storage/iso /mnt/storage/data /mnt/storage/import_export         # creates the directories
chmod /mnt/storage/iso /mnt/storage/data /mnt/storage/import_export         # sets the proper permissions on the dirs
chown 36:36 /mnt/storage/data /mnt/storage/iso /mnt/storage/import_export   # sets ownership of the folders to the proper gid.
{% endhighlight %}

Then, configure the exports file and start the service:

{% highlight bash %}
echo '/mnt/storage/data           \*(rw,sync,no_subtree_check,all_squash,anonuid=36,anongid=36)' >> /etc/exports
echo '/mnt/storage/import_export  \*(rw,sync,no_subtree_check,all_squash,anonuid=36,anongid=36)' >> /etc/exports
exportfs -a                       # initializes the newly modified exports file
systemctl start nfs.service       # start the nfs service
{% endhighlight %}

## Installing Ovirt

Next, let's get the Ovirt installation started:

{% highlight bash %}
yum install http://plain.resources.ovirt.org/pub/yum-repo/ovirt-release36.rpm -y # downloads the ovirt repo
yum -y install ovirt-engine                                                      # installs the libraries
engine-setup                                                                     # run the installation routine
{% endhighlight %}

### Engine Installation

If you follow the instructions and use the default installation, you should get an output like this:

{% highlight bash %}
[ INFO  ] Stage: Initializing
[ INFO  ] Stage: Environment setup
        Configuration files: ['/etc/ovirt-engine-setup.conf.d/10-packaging.conf']
        Log file: /var/log/ovirt-engine/setup/ovirt-engine-setup-20140310163840.log
        Version: otopi-1.2.0_rc2 (otopi-1.2.0-0.7.rc2.fc19)
[ INFO  ] Stage: Environment packages setup
[ INFO  ] Stage: Programs detection
[ INFO  ] Stage: Environment setup
[ INFO  ] Stage: Environment customization

        --== PRODUCT OPTIONS ==--
        --== PACKAGES ==--

[ INFO  ] Checking for product updates...
[ INFO  ] No product updates found

        --== NETWORK CONFIGURATION ==--

        Host fully qualified DNS name of this server [server.name]: example.ovirt.org
        Setup can automatically configure the firewall on this system.
        Note: automatic configuration of the firewall may overwrite current settings.
        Do you want Setup to configure the firewall? (Yes, No) [Yes]:
[ INFO  ] firewalld will be configured as firewall manager.

        --== DATABASE CONFIGURATION ==--

        Where is the Engine database located? (Local, Remote) [Local]:
        Setup can configure the local postgresql server automatically for the engine to run. This may conflict with existing applications.
        Would you like Setup to automatically configure postgresql and create Engine database, or prefer to perform that manually? (Automatic, Manual) [Automatic]:

        --== OVIRT ENGINE CONFIGURATION ==--

        Application mode (Both, Virt, Gluster) [Both]:
        Default storage type: (NFS, FC, ISCSI, POSIXFS) [NFS]:
        Engine admin password:
        Confirm engine admin password:

        --== PKI CONFIGURATION ==--

        Organization name for certificate [ovirt.org]:

        --== APACHE CONFIGURATION ==--

        Setup can configure apache to use SSL using a certificate issued from the internal CA.

        Do you wish Setup to configure that, or prefer to perform that manually? (Automatic, Manual) [Automatic]:
        Setup can configure the default page of the web server to present the application home page. This may conflict with existing applications.
        Do you wish to set the application as the default page of the web server? (Yes, No) [Yes]:

        --== SYSTEM CONFIGURATION ==--

        Configure WebSocket Proxy on this machine? (Yes, No) [Yes]:
        Configure an NFS share on this server to be used as an ISO Domain? (Yes, No) [Yes]:
        Local ISO domain path [/var/lib/exports/iso-20140310143916]:
        Local ISO domain ACL - note that the default will restrict access to example.ovirt.org only, for security reasons [example.ovirt.org(rw)]:
        Local ISO domain name [ISO_DOMAIN]:

        --== MISC CONFIGURATION ==--

        --== END OF CONFIGURATION ==--
{% endhighlight %}

What the routine installs is the ovirt administration server, and an ovirt host server.  Basically, it's an all-in-one configuration.  Everything you need to get a VM up and running has just been installed.  To access the administration portal, go to `https://<NAMEOFYOURSERVER>/webadmin`.  Enter admin as your username, and enter the password you provided in the installation.

## Configuring Ovirt

The next step is to make your first initial configurations so that you can start spinning up VMs.  Let's get started.

### Create Your First Datacenter

The datacenter is a top level organizational structure.  Consider it a top-level folder to organize your hosts.  By default, the 'default' datacenter is created for you.  To create a new datacenter, click 'Data Centers' in the left hand pane, and click 'New' in the right:

{% capture fig_img %}
![output]({{ base_path }}/images/administration-panel.png)
{% endcapture %}

Name your datacenter, and select storage type: shared.  Click ok, and you'll be taken to a new popup with a list of tasks to complete the configuration of your new datacenter.  The first step is to configure clusters.

### Configure Clusters

Once you've gotten the datacenter configured, you'll want to configure your cluster, which is a collection of Ovirt Hosts (the physical servers that run Ovirt).  Since this is only being done on one machine for now, this part's easy.  By default, the 'default' data center has a cluster called 'default'.  Let's create a cluster for our new datacenter:

{% capture fig_img %}
![output]({{ base_path }}/images/create-cluster.png)
{% endcapture %}

Enter in the appropriate information on the cluster, and click ok.  Next, let's configure the host.

### Configure Your Host

The host requires a little bit more configuration:

{% capture fig_img %}
![output]({{ base_path }}/images/new-host.png)
{% endcapture %}

enter the host name, IP, and root password.  Click advanced params and uncheck 'Automatically configure host firewall'.  This will block port 80, thus disconnecting you from the server.  Click ok, and continue to storage.

### Configuring Storage

This is where things get a bit tricky.  We're going to create 3 new domains with function values 'data' and 'export'.  Enter their names, and type the export path, which is going to be the <SERVERADDRESS>:/PATH/TO/SHARE and click ok.  Next, let's import the iso domain that we created during the engine installation.  Under the data center page, look at the bottom right hand pane and select the 'storage' tab.  Click 'attach iso'

{% capture fig_img %}
![output]({{ base_path }}/images/iso-domain.png)
{% endcapture %}

Select 'ISO_DOMAIN' and attach the domain to your datacenter.

### Configure Networks

Click on 'networks' on the left pane, and click 'new'.  Name your network.  If you're going to add the network to a VLAN, you'll probably want to have that VLAN number in the name of the network.  If not, don't worry, vlan tagging is optional.

{% capture fig_img %}
![output]({{ base_path }}/images/new-network.png)
{% endcapture %}

### Upload ISOs

Next, ssh into your ovirt box, and upload/download some initial ISOs to import into Ovirt for use.  CentOS is free, and there's now a free tier [developer's license](http://developers.redhat.com/blog/2016/03/31/no-cost-rhel-developer-subscription-now-available/) for RHEL.

{% highlight bash %}
engine-iso-uploader upload -i ISO_DOMAIN rhel-server-7.2-x86_64-dvd.iso
{% endhighlight %}

### Creating a Template VM

From there, you'll be able to create a brand new VM, and configure it to your liking.  First, click on 'VMs' in the left hand pane, and click 'new' in the right hand pane.

{% capture fig_img %}
![output]({{ base_path }}/images/new-vm.png)
{% endcapture %}

There's a score of options that you can play with to configure your VM, but for now we'll just focus on getting a demo machine up and running.  Select RHEL as your operating system, and select 'tiny' as your instance type.  Optimize for server, and enter the name of the machine.

In instances images, click 'create', and create a small 8GB disk and attach it to the machine.  Next, select your network for the VM's NIC to connect to.  Under 'boot options' tab, add a cd drive, and attach the ISO to the drive.  Click okay, and start the machine. Voila!  You've created your first VM!
