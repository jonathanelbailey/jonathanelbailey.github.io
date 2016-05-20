---
title: "Handling UEFI Activation Problems During Windows 8.1 Deployment"
modified:
categories:
excerpt: 'A function that mitigates activation problems during Windows 8.1 Deployment.'
tags:
  - powershell
  - deployment
  - uefi
header:
  overlay_image: 20151010_114929.jpg
  caption: "Photo credit: Jonathan Bailey"
date: 2015-10-07T13:47:33-05:00
---

{% include toc %}

## Introduction

When I transitioned my company's imaging process from Windows 7 Professional to Windows 8.1, there  were
some growing pains that were difficult to overcome.  I started imaging our workstations with Windows 8.1 after
testing, but the impetus was really necessity.  As soon as reserves of Windows 7 Certified Hardware disappeared,
there were consistent activation problems that couldn't be resolved.  The issue that we ran into was with our
process, which was admittedly a cost saving measure, of using the OEM keys that came with a machine instead of
Volume Licensed keys.

## Licensing Legality

Best Practices tells us that Microsoft wants you to use Volume Licensing for your deployment.  It's easier to manage,
and I'll be the first one to tell you that it's by far the better way to go.  However, sometimes there's just not that option
available.  Fortunately, [Microsoft Licensing](http://download.microsoft.com/download/3/D/4/3D42BDC2-6725-4B29-B75A-A5B04179958B/Reimaging.pdf)
is actually kind of lenient when it comes to deployment:

>One key benefit of licensing Microsoft software under a Microsoft Volume Licensing program is the right for
customers to use Volume Licensing media to deploy the same standard image of software across multiple licensed
devices. **It does not matter whether those devices are licensed under that particular Volume Licensing program,
through an OEM, or through retail channels, so long as certain eligibility rules are followed.**

**Note:** You'll want to peruse the Imaging documentation to understand what those eligibility rules are.
{: .notice}

## The Tools

The first thing that we want to do is download [Microsoft ADK](http://www.microsoft.com/en-us/download/details.aspx?id=39982).
Once we have that downloaded, we'll want to grab `oa3tool.exe` from out of that and either set an alias to it, add it to `PATH`, or
keep the executable near the script.  The tool itself is designed to interface with OA3, Microsoft's Windows 8/8.1 activation technology.
There's a nifty little switch that allows you to grab the hex value of the oem product key injected into the motherboard.
Using PowerShell to harness the power of .NET, we can use that hex value to activate Windows legitimately.

## The Script

Our first task is to see what the output is for oa3tool:

{% include base_path %}

{% capture fig_img %}
![output]({{ base_path }}/images/oa3tool.jpg)
{% endcapture %}

<figure>
  {{ fig_img | markdownify | remove: "<p>" | remove: "</p>" }}
  <figcaption>oa3tool's output.</figcaption>
</figure>

It's straight text.  So the first thing we'll have to do is convert this into something meaningful.

### Running the Tool

What we'll want to do is ensure that PowerShell properly runs the command, and we also want to ensure that the output is saved properly:

{% highlight powershell %}
Start-Process -FilePath $OA3ToolPath -ArgumentList '/validate' -RedirectStandardOutput .\validate.txt
$HexData = Get-Content .\validate.txt
{% endhighlight %}

### Getting rid of the useless stuff

Next, we're only after the hex values under the title `The ACPI MSDM table in hex:`.  So we'll first need to get rid of everything else:

{% highlight powershell %}
$HexData = $HexData | select -First 33 | select -Last 4
$HexData = $HexData -replace '\s+', ' '
$HexData = $HexData.trimstart(' ')
$HexData = $HexData.trimend(' ')
{% endhighlight %}

This first throws away everything after the first 33 lines of text, then throws everything before the last 4 of those lines of text.
Next, we replace all double spaces with single spaces, and trim any whitespace that's left.

### Converting from Hex to ASCII

Once we have the hex values isolated, we now need to split them into an array:

{% highlight powershell %}
$HexData.split(" ")
{% endhighlight %}

Now that we have an array, we can work with each hex value.  The next step is to convert each one to decimal:

{% highlight powershell %}
$HexData.split(" ") | FOREACH {[BYTE]([CONVERT]::toint16($_,16))}
{% endhighlight %}

And then we can convert that to ASCII and pop it into a variable:

{% highlight powershell %}
$HexData.split(" ") | FOREACH {[CHAR][BYTE]([CONVERT]::toint16($_,16))} | Set-Variable -name prodkey -PassThru
{% endhighlight %}

And finally, we join the array back to a single string, and clean up any unprintable characters:

{% highlight powershell %}
$prodkey = $prodkey -join ''
$prodkey = $prodkey -replace "[^ -x7e]",""
{% endhighlight %}

### Sending the key to slmgr.vbs

Our final step is to send the slmgr script to activate.  We'll want to run this in batch mode so we don't get any
annoying message windows, and cleanup our text file:

{% highlight powershell %}
cscript //b slmgr.vbs /ipk $prodkey
Remove-Item .\validate.txt
{% endhighlight %}

I hope the script helps!

## The Gist

{% gist jonathanelbailey/78817a36121b231f94b5 %}
