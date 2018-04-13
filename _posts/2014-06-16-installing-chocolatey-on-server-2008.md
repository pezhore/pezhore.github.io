---
layout: single
author_profile: true
classes: wide
title:  "Installing Chocolatey on Server 2008 R2 Core"
date:   2014-06-16
categories: ['Home Lab']
tags: archived powershell
---

While trying to install Chocolately on my Server 2008 R2 Core boxes, I ran into an issue where the .NET 4.0 download would fail repeatedly - "webpicmdline.exe has stopped working". No worries! I already had WinRM enabled, so I looked around for a Windows equivalent of scp so that I could get the .NET installer up there some other way.

Not long after I found a great script on PoshCode called Send-File. I manually downloaded the standalone .NET 4.0 installer from Microsoft, then followed the usage:

```
PS >$session = New-PsSession myServer
PS >Send-File c:\temp\dotNetFx40_Full_x86_x64_SC.exe c:\temp\dotNetFx40_Full_x86_x64_SC.exe $session
```

## Fail!

Unfortunately, due to a limitation with the default PsSession settings I couldn't send a file that big:

```
Sending data to a remote command failed with the following error message: The total data received from the remote
client exceeded allowed maximum. Allowed maximum is 52428800. For more information, see the
about_Remote_Troubleshooting Help topic.
    + CategoryInfo          : OperationStopped: (CLI-002:String) [], PSRemotingTransportException
    + FullyQualifiedErrorId : JobFailure
    + PSComputerName        : myServer
```

After more Googling, I came upon a [stackoverflow](https://stackoverflow.com/questions/13561730/maximum-data-size-in-a-remote-command) question whose answer fixed my problem!

On the destination (myServer), I ran the following in an elevated PowerShell command prompt:

{% highlight powershell %}
Register-PSSessionConfiguration -Name DataNoLimits
Set-PSSessionConfiguration -Name DataNoLimits -MaximumReceivedDataSizePerCommandMB 500 -MaximumReceivedObjectSizeMB 500
{% endhighlight %}

On my sending computer, I just changed my $session variable assignment to reference the new PS Session config:

{% highlight powershell %}
$Session = New-PSSession -ComputerName myServer -ConfigurationName DataNoLimits
{% endhighlight %}

The original Send-File command worked like a beauty and within ~30 seconds I had my .NET installer on my remote server. I was able to run the installer, and Chocolatey was installed successfully shortly thereafter.

## Updated: Prior .NET/32 bit compatibility

If you are still having trouble (as I was on one of my servers), you may need to enable the 32 bit/prior .NET functionality:

```
c:\> Start /w ocsetup ServerCore-WOW64
c:\> Start /w ocsetup NetFx2-ServerCore
c:\> Start /w ocsetup NetFx2-ServerCore-WOW64
```