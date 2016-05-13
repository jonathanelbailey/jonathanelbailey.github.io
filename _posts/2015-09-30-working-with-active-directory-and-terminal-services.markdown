---
title: "Working With Active Directory and Terminal Services"
modified:
categories:
excerpt: 'A small script that allows an administrator to get a list of all users who have not logged in to a terminal server.'
tags: ['Active Directory', 'PowerShell', 'RDS']
image:
  feature: 20151010_113639.jpg
date: 2015-09-30T16:29:11-05:00
---
{% include toc %}

## Introduction

In my production environment, I administer a number of tenanted terminal servers for our customers.  Sometimes
it can be a pain to administer not just Microsoft licensing for the servers, but also Suburban Software's licensing
for their LOB product.  Sometimes I'm asked to find out how many users are actually using the software.  Since the
LOB product doesn't have any built in logging for that, I turn to PowerShell and Active Directory Users and Computers.

## PsSessions

The first thing I do is log into the terminal server whose licensing is in question.  I open up a shell and
type the following:

{% highlight powershell %}
	$session = New-PSSession -ComputerName $ComputerName # this must be the name of the DC whose session you must import.
{% endhighlight %}

What I'm doing is starting a remote session on the DC because I need access to its built in ActiveDirectory module:

{% highlight powershell %}
	Invoke-Command -ScriptBlock {Import-Module ActiveDirectory} -Session $session
	Import-PSSession -Module activedirectory -Session $session
{% endhighlight %}

I run a command to import the module into the session, and then I import the SESSION into my current terminal server session!
It's absolutely amazing what you can do with PowerShell.  Now, since we've gotten our session imported, we can query the server
for that information.

## ActiveDirectory CMDlets

Next, I am going to query Active Directory to get a list of all users in the OU whose container represents the tenant organization:

{% highlight powershell %}
	$ou = Get-ADOrganizationalUnit -Filter * -Properties * | where-object name -match "$OUName"
	$users = Get-ADUser -Filter * -Properties * | Where-Object distinguishedname -Match $ou.distinguishedname
{% endhighlight %}

And finally, I pipe the `$users` variable to get my list:

{% highlight powershell %}
	$users | Select-Object -Property name,lastlogondate | Where-Object lastlogondate -eq $null
{% endhighlight %}

And if I'd like to have a list of users who actually HAVE logged in?  Simply flip the bit:

{% highlight powershell %}
	$users | Select-Object -Property name,lastlogondate | Where-Object lastlogondate -ne $null
{% endhighlight %}

Here's the gist:

{% gist jonathanelbailey/b97c17e712284cae93e0 %}
