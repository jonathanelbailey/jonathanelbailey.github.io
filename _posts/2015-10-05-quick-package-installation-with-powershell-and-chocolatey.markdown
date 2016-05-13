---
layout: post
title: "Quick Package Installation With PowerShell and Chocolatey"
modified:
categories: 
excerpt: 'A small script that demonstrates the simplicity of package installation using Active Directory, Chocolatey, and PsRemoting'
tags: ['PowerShell', 'Chocolatey', 'Active Directory']
image:
  feature: 20151010_112438.jpg
date: 2015-10-05T20:16:08-05:00
---

{% include toc %}

## Introduction

Sometimes you may be required to ensure the installation of application on a number of machines.  With
PowerShell and Chocolatey, it's extremely easy.

## Chocolatey

Chocolatey is a fantastic package manager that I have tested and installed throughout my production environment.
its reliability and abstraction makes for a perfect tool that simplifies the deployment of packages. To install 
Chocolatey, run the following:

{% highlight powershell %}
	iex ((new-object net.webclient).DownloadString('https://chocolatey.org/install.ps1'))
{% endhighlight %}

#### Choco Packages

Once you've installed chocolatey on your system, it's extremely simple to install a package:

{% raw %}
	choco install adobereader -y
{% endraw %}

## Scripting with Chocolatey

Building on this information, we can now build a script:

{% highlight powershell %}
	$scriptblock = {choco.exe install adobereader -y
					choco.exe install openoffice -y}

	$servers = Get-ADComputer -Filter * -Properties * | Where-Object name -match $SomeName
	foreach ($s in $servers){
		$session = new-pssession $s
		Invoke-Command -ScriptBlock $scriptblock -Session $session
	}
{% endhighlight %}

#### Step By Step

**Note:** This script is designed to be run from a domain controller.  It parses Active Directory objects
to find all servers that will be subject to remote installation.

The first part of the script creates a script block - or a lambda if it matters - and assigns it to a variable:

{% highlight powershell %}
	$scriptblock = {choco.exe install adobereader -y
					choco.exe install openoffice -y}
{% endhighlight %}

Next, Active Directory is queried to get a list of servers who require installation.  In my environment, this is
easily done with identifiable computer names:

{% highlight powershell %}
	$servers = Get-ADComputer -Filter * -Properties * | Where-Object name -match $SomeName
{% endhighlight %}

And finally, the remote sessions are created and the script block is run:

{% highlight powershell %}
	foreach ($s in $servers){
		$session = new-pssession $s
		Invoke-Command -ScriptBlock $scriptblock -Session $session
	}
{% endhighlight %}

## The Gist

Here's the gist for this one:

{% gist jonathanelbailey/1af6cc4f5828471650e6 %}
