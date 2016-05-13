---
title: "Automating SQL Server Installation With PowerShell and Chocolatey"
modified:
categories:
excerpt: 'A script that allows the installation and initial configuration of SQL Server 2012 Express'
tags: ['SQL Server', 'PowerShell', 'Chocolatey']
image:
  feature: 20151010_122533.jpg
date: 2015-10-05T22:00:10-05:00
---

{% include toc %}

## Introduction

Installing SQL Server can be time consuming as well as complicated.  The best way to tackle these pain points
are to automate the entire process.  In this post I will install and configure SQL Server using Chocolatey and
a custom configuration file.

## The Configuration File

The first part of the process is the creation of the configuration file.  I use a here string in order to dynamically generate one:

{% highlight powershell %}
	$Source = @"
	;SQL Server 2012 Configuration File
	; SQL-2012-Base.ini
	;
	[OPTIONS]
	; Specify the Instance ID for the SQL Server features you have specified. SQL Server directory structure, registry structure, and service names will incorporate the instance ID of the SQL Server instance.
	INSTANCEID=”$SqlInstance”
	; Specifies a Setup work flow, like INSTALL, UNINSTALL, or UPGRADE. This is a required parameter.
	ACTION=”Install”
	; Specifies features to install, uninstall, or upgrade. The list of top-level features include SQL, AS, RS, IS, MDS, and Tools. The SQL feature will install the Database Engine, Replication, Full-Text, and Data Quality Services (DQS) server. The Tools feature will install Management Tools, Books online components, SQL Server Data Tools, and other shared components.
	FEATURES=SQLENGINE,REPLICATION,FULLTEXT,SSMS
	; Displays the command line parameters usage
	HELP=”False”
	; Specifies that the detailed Setup log should be piped to the console.
	INDICATEPROGRESS=”False”
	; Setup will not display any user interface.
	QUIET=”True”
	; Setup will display progress only without any user interaction.
	QUIETSIMPLE=”$QuietSimple”
	; Specifies that Setup should install into WOW64. This command line argument is not supported on an IA64 or a 32-bit system.
	X86=”$x86”
	; Specifies the path to the installation media folder where setup.exe is located.
	MEDIASOURCE=”c:\programdata\chocolatey\lib\mssqlserver2012express”
	; Detailed help for command line argument ENU has not been defined yet.
	ENU=”False”
	; Parameter that controls the user interface behavior. Valid values are Normal for the full UI,AutoAdvance for a simplied UI, and EnableUIOnServerCore for bypassing Server Core setup GUI block.
	; UIMODE=”AutoAdvance”
	; Specify if errors can be reported to Microsoft to improve future SQL Server releases. Specify 1 or True to enable and 0 or False to disable this feature.
	ERRORREPORTING=”True”
	; Specify the root installation directory for shared components.  This directory remains unchanged after shared components are already installed.
	INSTALLSHAREDDIR=”D:\Program Files\Microsoft SQL Server”
	; Specify the root installation directory for the WOW64 shared components.  This directory remains unchanged after WOW64 shared components are already installed.
	INSTALLSHAREDWOWDIR=”D:\Program Files (x86)\Microsoft SQL Server”
	; Specify the installation directory.
	INSTANCEDIR=”D:\Program Files\Microsoft SQL Server”
	; Specify that SQL Server feature usage data can be collected and sent to Microsoft. Specify 1 or True to enable and 0 or False to disable this feature.
	SQMREPORTING=”True”
	; Specify a default or named instance. MSSQLSERVER is the default instance for non-Express editions and SQLExpress for Express editions. This parameter is required when installing the SQL Server Database Engine (SQL), Analysis Services (AS), or Reporting Services (RS).
	INSTANCENAME=”$InstanceName”
	; Agent account name
	AGTSVCACCOUNT=”NT AUTHORITYNETWORK SERVICE”
	; Auto-start service after installation.
	AGTSVCSTARTUPTYPE=”Automatic”
	; Startup type for the SQL Server service.
	SQLSVCSTARTUPTYPE=”Automatic”
	; Level to enable FILESTREAM feature at (0, 1, 2 or 3).
	FILESTREAMLEVEL=”0″
	; Set to “1” to enable RANU for SQL Server Express.
	ENABLERANU=”False”
	; Specifies a Windows collation or an SQL collation to use for the Database Engine.
	SQLCOLLATION=”SQL_Latin1_General_CP1_CI_AS”
	; Account for SQL Server service: DomainUser or system account.
	SQLSVCACCOUNT=”NT AUTHORITYNETWORK SERVICE”
	; Provision current user as a Database Engine system administrator for SQL Server 2012 Express.
	ADDCURRENTUSERASSQLADMIN=”False”
	; Specify 0 to disable or 1 to enable the TCP/IP protocol.
	TCPENABLED=”1″
	; Specify 0 to disable or 1 to enable the Named Pipes protocol.
	NPENABLED=”0″
	; Startup type for Browser Service.
	BROWSERSVCSTARTUPTYPE=”Enabled”
	; Accept SQL Server license terms
	IACCEPTSQLSERVERLICENSETERMS=”TRUE”
	; Specify whether SQL Server Setup should discover and include product updates. The valid values are True and False or 1 and 0. By default SQL Server Setup will include updates that are found.
	UpdateEnabled=”True”
	; Specify the location where SQL Server Setup will obtain product updates. The valid values are “MU” to search Microsoft Update, a valid folder path, a relative path such as .MyUpdates or a UNC share. By default SQL Server Setup will search Microsoft Update or a Windows Update service through the Window Server Update Services.
	UpdateSource=”MU”
	; CM brick TCP communication port
	COMMFABRICPORT=”0″
	; How matrix will use private networks
	COMMFABRICNETWORKLEVEL=”0″
	; How inter brick communication will be protected
	COMMFABRICENCRYPTION=”0″
	; TCP port used by the CM brick
	MATRIXCMBRICKCOMMPORT=”0″
	SECURITYMODE="SQL"
	SAPWD="$Pass"
	"@
{% endhighlight %}

#### Variables

In the here-string, you'll see that there's a few variables.  These are the only parts of the configuration file
that vary from installation to installation.  Let's go ahead and populate it now:

{% highlight powershell %}
$SqlInstance = 'MsSqlServer'
$QuietSimple = $true
$x86 = $false
$InstanceName = 'SqlExpress'
$Pass = 'P@$$w0rd'
$chocopath = 'c:\programdata\chocolatey'
{% endhighlight %}

Next, let's create the configuration file:

{% highlight powershell %}
set-content -Value $Source -Path 'configurationfile.ini'
{% endhighlight %}

 #### Checking for Chocolatey

 The first thing I want to do is ensure that Chocolatey is in fact installed, and if not, install it:

 {% highlight powershell%}
	$chocopath = 'c:\programdata\chocolatey'
	 if ((test-path $chocopath) -eq $false){
		set-executionpolicy remotesigned -force
		iex ((new-object net.webclient).DownloadString('https://chocolatey.org/install.ps1'))
	}
 {% endhighlight %}

#### Chocolatey Parameters

The next step is to run Chocolatey.  There's a great feature about Chocolatey that allows you to override the default parameters
that exist in the package install script.  Since we have a configuration file, all we have to do is tell the install package to
use the file:

{% highlight powershell %}
	choco install mssqlserver2012express --params='/ConfigurationFile="ConfigurationFile.ini"' -y
{% endhighlight %}

#### Cleanup

Once the install is finished, the last thing to do is remove the configuration file:

{% highlight powershell %}
	Remove-Item configurationfile.ini -Force
{% endhighlight %}

And that's it. Enjoy.

## The Gist

{% gist jonathanelbailey/2f72ba70da32c124edd8 %}
