---
layout: post
title: "PowerShell Desired State Configuration"
modified:
categories: 
excerpt: 'An overview of a PowerShell Push Configuration'
tags: ['PowerShell', 'DSC','DNS','DHCP']
image:
  feature: 20151010_113344.jpg
date: 2015-09-24T17:06:40-05:00
---
{% include toc %}

## Introduction

PowerShell Desired State Configuration's been out since PowerShell 4.0, and when implemented properly, it's almost like cheating when it comes to provisioning and configuring servers.  The learning curve isn't very tough,
and there's a vibrant open-source community that supports the creation of resources.  
Furthermore, if there's not a resource available that you need, you can always create a resource yourself.  In this piece, I'll be walking through how to create a DSC Push configuration that will Provision and configure 
a Server 2012 installation with the DNS and DHCP server roles.

#### Environment Configuration
 
For the test environment, I used a Server 2012 R2 Core installation.  Before getting started, please run the following from a shell on the test server:

{% raw %}
	winrm qc
	winrm s winrm/config/client '@{TrustedHosts="*"}'
{% endraw %}

This opens up WinRM so it's not recommended to use this configuration in production.  Either add an IP, a fully qualified domain name, or a hostname to it to manually secure it.  Next, we'll have to Provision the server.
Since PowerShell 5's not out yet, we'll use Chocolatey, the preferred open source package manager for Windows to download our tools:

{% highlight powershell %}
set-executionpolicy remotesigned -force
iex ((new-object net.webclient).DownloadString('https://chocolatey.org/install.ps1'))
{% endhighlight %}

This installs Chocolatey and updates PATH for easy calls to the chocolatey binaries.  Next, we're going to grab git:

{% raw %}
	choco install git -y
	choco install poshgit -y
{% endraw %}

Posh-Git is a PowerShell module for git.  It gives you information about your local repos, git colors, and SSH, which can certainly make working with git more manageable on Windows.  Next, we'll need to download a couple of
DSC Resources:

{% highlight powershell %}
set-location $Home\documents\github
git clone 'https://github.com/PowerShell/DscResources'
copy-item DCSResources $home\documents\windowspowershell\modules\ -recurse
{% endhighlight %}

You may at this point need to restart the shell in order to load the modules.  To make sure that you've got access to them, enter the following:

{% highlight powershell %}
get-module -name dscresources
{% endhighlight %}

you should get a giant list of resources.  The ones we'll be interested in today are xNetworking, xDhcpServer, and xDnsServer.  Since the resources are loaded, let's go ahead and get started with planning our server configuration.

#### Configuration Requirements

Our server's going to be providing DNS and DHCP services to a theoretical web farm, say 3 web servers, a caching server and a SQL server.  Here's the network configuration:

{% raw %}
	Network ID: 192.168.0.0
	Broadcast:  192.168.0.15
	Gateway:    192.168.0.1
	SubMask:    28
	DNS Config: 192.168.0.2
	Zone Name:  internal.contoso.org
	Web Pool A Record: webpool1
{% endraw %}

Of course, this DNS configuration is going to use round robin since we've created a server pool.  The next step is to make sure that we've got the host information as well:

{% raw %}
	  Host Name    IP Address     Role         MAC Address
	------------------------------------------------------------
	  TestNode     192.168.0.2   DNS/DHCP    07-3D-D3-29-39-32
	  Web1         192.168.0.3   Web Server  FF-CA-2D-36-B6-4A
	  Web2         192.168.0.4   "           5A-62-17-7D-87-FA
	  Web3         192.168.0.5   "           AE-6B-C3-DE-39-1B
	  SQL1         192.168.0.6   DB          59-72-C2-C4-76-79
	  Cache1       192.168.0.7   Cache       A2-A1-AC-4D-06-8F
{% endraw %}

**Tip:** You may have noticed something about our set up.  If you're planning on deploying a network infrastructure server, you'll want to take a few precautions.  First, Make sure your server is on the appropriate network.  If you
do not want the server to be on the same network from which you are applying the push configuration, I'd recommend your test machines have two network adapters each.  We'll be labeling one of the adapters 'TestInt' for reference.
If you don't want to muck up your client machine with an extra adapter, make sure your client machine can find the route to the network of the test machine.
{: .notice}

---

## Creating a DSC Configuration Script

Now that we have our environment set up, and a reasonable set of requirements, let's start on the configuration script.  First, let's grab the configuration snippet from ISE to see the elements of a configuration:

{% highlight powershell %}
configuration Name
{
    # One can evaluate expressions to get the node list
    # E.g: $AllNodes.Where("Role -eq Web").NodeName
    node ("Node1","Node2","Node3")
    {
        # Call Resource Provider
        # E.g: WindowsFeature, File
        WindowsFeature FriendlyName
        {
           Ensure = "Present"
           Name = "Feature Name"
        }

        File FriendlyName
        {
            Ensure = "Present"
            SourcePath = $SourcePath
            DestinationPath = $DestinationPath
            Type = "Directory"
            Requires = "[WindowsFeature]FriendlyName"
        }       
    }
}
{% endhighlight %}

#### Networking

Here we see that the configuration keyword acts much like a function.  Inside the function, you'll see the node keyword.  This references all the machines that this configuration in particular will apply to.
Inside are a couple of resource providers.  These are the bread and butter of DSC.  Resource providers offer declarative domain specific language that simplifies the configuration process.  Resources abstract all the
dirty work that can go into writing home made PowerShell scripts that could do the same thing. Let's go ahead and configure this using the xNetworking resource:
  
 {% highlight powershell %}
configuration TestNodeConfig
{
	Param( $ServerIp,
		   $MachineName)
    
	import-dscresource -module xNetworking
	
	# One can evaluate expressions to get the node list
    # E.g: $AllNodes.Where("Role -eq Web").NodeName
    node ($MachineName)
    {
        # Call Resource Provider
        xIPAddress ServerIP	
        {
            IPAddress = $ServerIP
            InterfaceAlias = 'TestInt'
            DefaultGateway = '192.168.0.1'
            SubnetMask = 28
            AddressFamily = 'IPv4'
        }     
        xDNSServerAddress ServerDNS 
        {
            Address = $ServerIP
            InterfaceAlias = 'Ethernet'
            AddressFamily = 'IPv4'
        }
	}
}
{% endhighlight %}

There!  As you can see, we've created a configuration called 'TestNodeConfig', and just like a function, we've added the Param() keyword to allow parameterization of the Server's IP address and Computer name.
Next, we've imported the xNetworking resource, since it's not a default part of DSC.  Finally, we've called the xIPAddress and xDNSServerAddress resource providers and declared the network settings that we want on our test server.

But...how did we get from there to here?  Well, we have to look at the documentation for the [xNetworking Resource Provider](https://github.com/PowerShell/xNetworking).  In the Readme, we see there's a multitude of options that we can
call from to configure a network interface.  You'll also notice that there's a resource to configure a default gateway.  I didn't actually use that because the 192.168.0.0 network is a dummy network for what I'm using it for.
But, if using xDefaultGatewayAddress suits your environment better, by all means, go for it.

#### Server Roles

Next, let's add the resource providers for Windows Server Roles.  They are listed under Windows Features, which is built in by default.

**Tip:** Because DSC's configuration file is a builtin keyword, any resources you have in your $env:PSModulePath will benefit from auto tabulation through each resource's settings.
{: .notice}

{% highlight powershell %}
        WindowsFeature DHCP
        {
           Ensure = 'Present'
           Name = 'DHCP'
           DependsOn = '[xIPAddress]ServerIP','[xDNSServerAddress]ServerDNS'
        }
        WindowsFeature DNS
        {
           Ensure = 'Present'
           Name = 'DNS'
           DependsOn = '[xIPAddress]ServerIP','[xDNSServerAddress]ServerDNS'
        }
{% endhighlight %}

As you can see, creation of the Windows Feature resource providers are pretty straight forward.  But, there's a new property we should look at into further detail:  `DependsOn`.  To ensure that a DSC configuration
works properly, there must be forward thought regarding how a resource provider may conflict when working with other resources.

If you're already familiar with how DNS Servers work, you'll know immediately that the DNS server installation wizard doesn't like interfaces with DHCP addresses.  Therefore, it's prudent to create your static IP 
information before you install that feature.  The same goes for the DHCP role as well.  The way it works is that you call the resource in braces, and then reference the given name like `[SomeResource]FriendlyName`.

#### DHCP Server Scope

For our next configuration step, we'll set DHCP Scope and Options:

{% highlight powershell %}
        xDhcpServerScope ServerScope
        {
            IPStartRange = "192.168.0.3"
            IPEndRange = "192.168.0.7"
            Name = "TestScope1"
            SubnetMask = "255.255.255.240"
            State = "Active"            
            Ensure = "Present"
            LeaseDuration = "7:00:00"
            DependsOn = "[WindowsFeature]DHCP"
            
        }
        xDhcpServerOption ServerOpt
        {
            ScopeID = $ScopeID
            DnsServerIPAddress = $ServerIP
            DnsDomain = "TestDomain"
            AddressFamily = "IPv4"            
            Ensure = "Present"
            DependsOn = "[xDhcpServerScope]ServerScope"
        }
{% endhighlight %}

**Tip:** I'd like to point out that the `DependsOn` property in ServerOpt doesn't explicitly depend on DHCP being installed.  It depends on ServerScope, which depends on DHCP being installed.  Useful information for efficient coding.
{:notice}

#### DHCP Reservations

Our next step is the configuration of DHCP reservations.  If you're wondering where I got some of the data from, like
MAC address, I just used a MAC address generator.  These servers only exist theoretically at the moment.

{% highlight powershell %}
        xDhcpServerReservation Web1Res
        {
            ScopeID = $ScopeID
            IPAddress = '192.168.0.3'
            ClientMACAddress = 'FF-CA-2D-36-B6-4A'
            Name = 'Web1'
            AddressFamily = 'IPv4' 
            Ensure = 'Present'
            DependsOn = '[xDhcpServerScope]ServerScope'  
        }
        xDhcpServerReservation Web2Res
        {
            ScopeID = $ScopeID
            IPAddress = '192.168.0.4'
            ClientMACAddress = '5A-62-17-7D-87-FA'
            Name = 'Web2'
            AddressFamily = 'IPv4' 
            Ensure = 'Present'
            DependsOn = '[xDhcpServerScope]ServerScope'  
        }
        xDhcpServerReservation Web3Res
        {
            ScopeID = $ScopeID
            IPAddress = '192.168.0.5'
            ClientMACAddress = 'AE-6B-C3-DE-39-1B'
            Name = 'Web3'
            AddressFamily = 'IPv4' 
            Ensure = 'Present'
            DependsOn = '[xDhcpServerScope]ServerScope' 
        }
        xDhcpServerReservation SQL1Res
        {
            ScopeID = $ScopeID
            IPAddress = '192.168.0.6'
            ClientMACAddress = '59-72-C2-C4-76-79'
            Name = 'SQL1'
            AddressFamily = 'IPv4' 
            Ensure = 'Present'
            DependsOn = '[xDhcpServerScope]ServerScope'  
        }
        xDhcpServerReservation Cache1Res
        {
            ScopeID = $ScopeID
            IPAddress = '192.168.0.7'
            ClientMACAddress = 'A2-A1-AC-4D-06-8F'
            Name = 'Cache1'
            AddressFamily = 'IPv4'
            Ensure = 'Present' 
            DependsOn = '[xDhcpServerScope]ServerScope'  
        }
{% endhighlight %}

---

## The Script Resource Provider

**Note:** At the time this DSC configuration was made, there wasn't yet a resource provider for DNS Server role configuration.
Fortunately, there is one now but I think it's appropriate to go over the `Script` Resource provider to see what exactly goes on
behind the scenes of DSC.
{:notice}

Let's first start with the [resources available](https://technet.microsoft.com/en-us/library/dn282130.aspx):

{% highlight powershell %}
	Script [string] #ResourceName
	{
		GetScript = [string]
		SetScript = [string]
		TestScript = [string]
		[ Credential = [PSCredential] ]
		[ DependsOn = [string[]] ]
	}
{% endhighlight %}

The main properties we're going to be looking at are `GetScript`,`SetScript`, and `TestScript`.  

#### GetScript

`GetScript` is used by the `Get-DscConfiguration` CMDlet to bring back a hashtable -- which we must build ourselves.  For this step, we've got to figure
out what we want the script resource to do.  Well, we want to make sure the DNS server points us to our web farm.  So, we've got
to make sure it has the proper A records for each machine.  Secondly, we also want to make sure that the A records are all referenced
in the web pool.

{% highlight powershell %}
GetScript = { return @{ DNSZone = ((get-dnsserverzone).zonename | where-object {$_ -eq 'internal.contoso.org'})
						ARPZone = ((get-dnsserverzone).zonename | where-object {$_ -eq '0.0.168.192.in-addr.arpa'})
						 Web1ARec = ((get-dnsserverresourcerecord -zonename 'internal.contoso.org' -name 'webpool1' `
									 | select-object -expandproperty recorddata).ipv4address.ipaddresstostring | `
									 where-object {$_ -eq '192.168.0.3'})
						 Web2ARec = ((get-dnsserverresourcerecord -zonename 'internal.contoso.org' -name 'webpool1' `
									 | select-object -expandproperty recorddata).ipv4address.ipaddresstostring | `
									  where-object {$_ -eq '192.168.0.4'})
						 Web3ARec = ((get-dnsserverresourcerecord -zonename 'internal.contoso.org' -name 'webpool1' `
									 | select-object -expandproperty recorddata).ipv4address.ipaddresstostring | `
									  where-object {$_ -eq '192.168.0.5'})
						 SQL1ARec = ((get-dnsserverresourcerecord -zonename 'internal.contoso.org' -name 'sql1' `
									 | select-object -expandproperty recorddata).ipv4address.ipaddresstostring | `
									 where-object {$_ -eq '192.168.0.6'})
						 Cache1ARec = ((get-dnsserverresourcerecord -zonename 'internal.contoso.org' -name 'cache1' `
									   | select-object -expandproperty recorddata).ipv4address.ipaddresstostring | `
									   where-object {$_ -eq '192.168.0.7'})  
						 RoundRobin = ((get-dnsserver | select-object -expandproperty serversettings).roundrobin)
						 Fwdr1 = ((get-dnsserver | select-object -expandproperty serversettings).ipaddress.ipaddresstostring `
								  | where-object {$_ -eq '8.8.8.8'})
						 Fwdr2 = ((get-dnsserver | select-object -expandproperty serversettings).ipaddress.ipaddresstostring `
								  | where-object {$_ -eq '4.2.2.2'})   
					   }
			}
{% endhighlight %}

As you can see, we create a hashtable that returns the CNAME for the IPs provided.

#### SetScript

Next, we need to be able to apply whatever settings we need to the script.  `SetScript` will allow us to use the 
`Start-DscConfiguration` CMDlet to apply the settings to the server if the settings aren't what they should be:

{% highlight powershell %}
SetScript = {
			  Add-DnsServerPrimaryZone -Name 'internal.contoso.org' -ZoneFile 'internal.contoso.org.dns'
			  Add-DnsServerPrimaryZone -NetworkID 192.168.0.0/28 -ZoneFile '0.168.192.in-addr.arpa.dns'

			  Add-DnsServerForwarder -IPAddress 8.8.8.8 -PassThru
			  Add-DnsServerForwarder -IPAddress 4.2.2.2 –PassThru

			  dnscmd.exe localhost /config /roundrobin 1

			  Add-DnsServerResourceRecordA -Name 'WebPool1' -ZoneName 'Internal.contoso.org' `
				  -ipaddress 192.168.0.3,192.168.0.4,192.168.0.5
			  Add-DnsServerResourceRecordA -Name 'sql1' -ZoneName 'Internal.contoso.org' `
				  -ipaddress 192.168.0.6
			  Add-DnsServerResourceRecordA -Name 'cache1' -ZoneName 'Internal.contoso.org' `
				  -ipaddress 192.168.0.7

			  Add-DnsServerResourceRecord -name '3' -Ptr -ZoneName '0.0.168.192.in-addr.arpa' `
				  -PtrDomainName 'webpool1.internal.contoso.org'
			  Add-DnsServerResourceRecord -name '4' -Ptr -ZoneName '0.0.168.192.in-addr.arpa' `
				  -PtrDomainName 'webpool1.internal.contoso.org'
			  Add-DnsServerResourceRecord -name '5' -Ptr -ZoneName '0.0.168.192.in-addr.arpa' `
				  -PtrDomainName 'webpool1.internal.contoso.org'
			  Add-DnsServerResourceRecord -name '6' -Ptr -ZoneName '0.0.168.192.in-addr.arpa' `
				  -PtrDomainName 'sql1.internal.contoso.org'
			  Add-DnsServerResourceRecord -name '7' -Ptr -ZoneName '0.0.168.192.in-addr.arpa' `
				  -PtrDomainName 'cache1.internal.contoso.org'
			 }
{% endhighlight %}

#### TestScript

The final property is what `Start-DscConfiguration` uses to determine if it needs to apply the desired settings on the node.
It simply returns a value of true or false:

{% highlight powershell %}
TestScript = { if((get-dnsserverzone).zonename -ccontains '0.0.168.192.in-addr.arpa' -and 'internal.contoso.org') `
				   {return (Get-DnsServerResourceRecord -zoneName 'internal.contoso.org' -name 'webpool1' `
						| select-object -expandproperty recorddata).ipv4address.ipaddresstostring -ccontains `
						'192.168.0.3' -and '192.168.0.4' -and '192.168.0.5'}
			   else{
				 return $false
			   }
			 }
{% endhighlight %}

#### Result

And here's what the entire resource looks like:

{% highlight powershell %}
Script DNSConfig{
	SetScript = {
				  Add-DnsServerPrimaryZone -Name 'internal.contoso.org' -ZoneFile 'internal.contoso.org.dns'
				  Add-DnsServerPrimaryZone -NetworkID 192.168.0.0/28 -ZoneFile '0.168.192.in-addr.arpa.dns'

				  Add-DnsServerForwarder -IPAddress 8.8.8.8 -PassThru
				  Add-DnsServerForwarder -IPAddress 4.2.2.2 –PassThru

				  dnscmd.exe localhost /config /roundrobin 1

				  Add-DnsServerResourceRecordA -Name 'WebPool1' -ZoneName 'Internal.contoso.org' `
					  -ipaddress 192.168.0.3,192.168.0.4,192.168.0.5
				  Add-DnsServerResourceRecordA -Name 'sql1' -ZoneName 'Internal.contoso.org' `
					  -ipaddress 192.168.0.6
				  Add-DnsServerResourceRecordA -Name 'cache1' -ZoneName 'Internal.contoso.org' `
					  -ipaddress 192.168.0.7

				  Add-DnsServerResourceRecord -name '3' -Ptr -ZoneName '0.0.168.192.in-addr.arpa' `
					  -PtrDomainName 'webpool1.internal.contoso.org'
				  Add-DnsServerResourceRecord -name '4' -Ptr -ZoneName '0.0.168.192.in-addr.arpa' `
					  -PtrDomainName 'webpool1.internal.contoso.org'
				  Add-DnsServerResourceRecord -name '5' -Ptr -ZoneName '0.0.168.192.in-addr.arpa' `
					  -PtrDomainName 'webpool1.internal.contoso.org'
				  Add-DnsServerResourceRecord -name '6' -Ptr -ZoneName '0.0.168.192.in-addr.arpa' `
					  -PtrDomainName 'sql1.internal.contoso.org'
				  Add-DnsServerResourceRecord -name '7' -Ptr -ZoneName '0.0.168.192.in-addr.arpa' `
					  -PtrDomainName 'cache1.internal.contoso.org'
				 }
	TestScript = { if((get-dnsserverzone).zonename -ccontains '0.0.168.192.in-addr.arpa' -and 'internal.contoso.org') `
					   {return (Get-DnsServerResourceRecord -zoneName 'internal.contoso.org' -name 'webpool1' `
							| select-object -expandproperty recorddata).ipv4address.ipaddresstostring -ccontains `
							'192.168.0.3' -and '192.168.0.4' -and '192.168.0.5'}
				   else{
					 return $false
				   }
				 }
	GetScript = { return @{ DNSZone = ((get-dnsserverzone).zonename | where-object {$_ -eq 'internal.contoso.org'})
							ARPZone = ((get-dnsserverzone).zonename | where-object {$_ -eq '0.0.168.192.in-addr.arpa'})
							Web1ARec = ((get-dnsserverresourcerecord -zonename 'internal.contoso.org' -name 'webpool1' `
										| select-object -expandproperty recorddata).ipv4address.ipaddresstostring | `
										 where-object {$_ -eq '192.168.0.3'})
							Web2ARec = ((get-dnsserverresourcerecord -zonename 'internal.contoso.org' -name 'webpool1' `
										 | select-object -expandproperty recorddata).ipv4address.ipaddresstostring | `
										  where-object {$_ -eq '192.168.0.4'})
							Web3ARec = ((get-dnsserverresourcerecord -zonename 'internal.contoso.org' -name 'webpool1' `
										| select-object -expandproperty recorddata).ipv4address.ipaddresstostring | `
										 where-object {$_ -eq '192.168.0.5'})
							SQL1ARec = ((get-dnsserverresourcerecord -zonename 'internal.contoso.org' -name 'sql1' `
										| select-object -expandproperty recorddata).ipv4address.ipaddresstostring | `
										 where-object {$_ -eq '192.168.0.6'})
							Cache1ARec = ((get-dnsserverresourcerecord -zonename 'internal.contoso.org' -name 'cache1' `
										  | select-object -expandproperty recorddata).ipv4address.ipaddresstostring | `
										   where-object {$_ -eq '192.168.0.7'})  
							RoundRobin = ((get-dnsserver | select-object -expandproperty serversettings).roundrobin)
							Fwdr1 = ((get-dnsserver | select-object -expandproperty serversettings).ipaddress.ipaddresstostring `
									 | where-object {$_ -eq '8.8.8.8'})
							Fwdr2 = ((get-dnsserver | select-object -expandproperty serversettings).ipaddress.ipaddresstostring `
									 | where-object {$_ -eq '4.2.2.2'})   
						   }
				}
	DependsOn = '[WindowsFeature]DNS'
}
{% endhighlight %}

---

## Initializing the Script

The final part is initializing the configuration:

{% highlight Powershell %}
Set-Location c:

#### Push the DSC configuration to the Test server
TestNodeConfig -machinename 'TestNode' -serverIP '192.168.0.2' -scopeid '192.168.0.0'

start-dscconfiguration -computername 'TestNode' -path 'c:\testnodeconfig'
{% endhighlight %}

once initialized, you'll be able to see DSC work its magic.  Once the script is finished, run `Test-DscConfiguration`
to ensure that the configuration is correct.

Here's the Gist:

{% gist jonathanelbailey/21a7506893b97637be5b %}
