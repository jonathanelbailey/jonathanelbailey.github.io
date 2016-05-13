---
layout: post
title: "Archiving VMs With PowerShell and Hyper-V 2012 R2"
modified:
categories: 
excerpt: 'A set of functions that allow you to create VM archives using live exports with PowerShell and Hyper-V'
tags: ['Hyper-V','PowerShell']
image:
  feature: 20151010_123519_HDR.jpg
date: 2015-10-07T10:31:04-05:00
---

{% include toc %}

## Introduction

For a time, I didn't have access to Enterprise level backup utilities like Veeam, and I had to make do with less
desirable, less effective, and less efficient methods of Virtual Machine management.  Today, there are better, free
solutions, but I feel like this demonstrates the underpinnings of how something like [Veeam Free](http://www.veeam.com/virtual-machine-backup-solution-free.html)
actually work.

**Note:** I want to go over a couple of things before I start with my script examples.  Veeam as a tool handles Backup, replication,
exports, reporting.  It offers the flexibility to use backups for testing, and it simplifies recovery of Virtual Machines
and all the files those VMs may contain.  Rolling your own on a backup  solution is stupid.  Don't do it.  But To understand how some of these
solutions work, it's appropriate to know how it all works.
{: .notice }

## Hyper-V PowerShell Commands

The commands we'll be working with are:

{% highlight powershell %}
get-vm
export-vm
get-vmsnapshot
remove-vmsnapshot
{% endhighlight %}

The first thing that we'll want to do is gather a list of Virtual Machines for Backups.  When I created these functions,
I had 4 separate roles that I used to categorize the backups that were done; ALL, RDP,SQL, and DC.  Using this information,
I simplified the querying process using a switch:

{% highlight powershell %}
switch ($VMScope){
	ALL { $VMs = Get-VM }
	RDP { $VMs = Get-VM | Where-Object name -Match 'RDP' }
	SQL { $VMs = Get-VM | Where-Object name -Match 'SQL' }
	DC { $VMs = Get-VM | Where-Object name -Match 'DC' }
}
{% endhighlight %}

Obviously I support an RDP environment, so the RDP switch referred to our production application servers.

## The Functions

for ease of use, I separated the logic of the code into functions.  Abstracting and simplifying each function
creates more readable and maintainable code.  Here's the step-by-step process that I took to ensure backups were created:

#### Remove-CheckpointsForExport

The first step is to ensure that any checkpoints/snapshot differencing disks are merged back into the VM's VHDX file:

{% highlight powershell %}
Function Remove-Checkpointsforexport{
    Param( [ValidateSet('All','RDP','SQL','DC')]
           $VMScope )
    BEGIN{}
    PROCESS{
        switch ($VMScope){
            ALL { $VMs = Get-VM }
            RDP { $VMs = Get-VM | Where-Object name -Match 'RDP' }
            SQL { $VMs = Get-VM | Where-Object name -Match 'SQL' }
            DC { $VMs = Get-VM | Where-Object name -Match 'DC' }
        }
    }
    END{
        $VMs | Get-VMSnapshot | Remove-VMSnapshot
    }
}
{% endhighlight %}

the function, when run, will simply go through each virtual machine in the category and remove any checkpoints.

#### New-VmExportArchive

Next, we'll need to ensure that the VMs are ready for upload to FTP ( I know, but yes.  At the time I was using this tool, we were using FTP for backups.  It was not my design.)
New-VmExportArchive will grab the Exported VMs and archive them for upload:

{% highlight powershell %}
Function New-VmExportArchive{
    Param( $Path,
           $Archivepath )
    BEGIN{}
    PROCESS{
        if ((test-path $path) -ne $true){
            write-host -foregroundcolor yellow "$path does not exist."
        }
        else{
            7z a $archivepath $path
        }
    }
    END{
        7z t * $archivepath
    }
}
{% endhighlight %}

#### Get-VmExports

Get-VmExports is the function that brings the rest of the functions together for an actionable command:

{% highlight powershell %}
Function Get-VMExports{
    Param( [ValidateSet('All','RDP','SQL','DC')]
           $VMScope,
           [switch]$RemoveCheckpoint,
           $Path )
    BEGIN{
        switch($VMScope){
            ALL {$VMs = Get-VM}
            RDP { $VMs = Get-VM | Where-Object name -Match 'RDP' }
            SQL { $VMs = get-vm | where-object name -Match 'SQL' }
            DC { $VMs = get-vm | Where-Object name -Match 'DC' }
        } 
		if ($removecheckpoint -eq $true){
			remove-checkpointsforExport -vmscope $vmscope
		}
        $result = @()
        $i = 0
    }
    PROCESS{
        foreach ($v in $VMS){
            $vmexport = Export-VM -ComputerName $v -Path $Path
            $result += $vmexport
            $i++
            Write-Progress -Activity "Exporting $($v.name)..." ` -Status "$i of $($vms.count) Completed" `
                -PercentComplete "(($i / $($vms.count)) * 100)"
        }
    }
    END{
        new-vmexportarchive -path $path -archivename "$((get-date).datetime)Vm-Backup.7z"
        write-output $result
    }
}
{% endhighlight %}

It ensures checkpoints are cleared, exports the VMs, and archives them for upload.

#### Initializing the Functions

to run it, type:

{% highlight powershell %}
	get-vmexports -vmscope ALL -removecheckpoint -path d:\exports
{% endhighlight %}

## The Gist

{% gist jonathanelbailey/1571c9e3dc3e07f4cbe2 %}