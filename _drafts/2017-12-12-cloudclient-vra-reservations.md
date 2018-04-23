---
layout: single
title:  "Using CloudClient to bulk change vRA Reservations"
date:   2017-12-12
categories: ['Automation']
tags: ['archived', 'PowerShell', 'VMware']
---
What happens when you have ~500 VMs in vRealize Automation that need to have their reservations changed? You figure out a way to script that tedious process. (This will provide some background and more details on a recent [VMware Communities](https://code.vmware.com/forums/5098/vrealize-automation#577825) post.)

I first started looking at what I was most familiar with - [PowervRA](https://github.com/jakkulabs/PowervRA) - a great community supported, PowerShell wrapper for the vRA API stack. Unfortunately, PowervRA does not offer reservation manipulation - only reporting.

[CloudClient](https://code.vmware.com/web/dp/tool/cloudclient/4.5.0) is a Java application wrapper for the vRA API. Bulk changing reservations is one use case, but several other uses exist in the documentation from that download site.

To use CloudClient in a wrapper script, we must first save our password by interactively launching CloudClient and executing the `login keyfile` command. This command specifies an output location for the encrypted file and prompts for a password. Most CloudClient activities require authentication with both vRA and vRA's IaaS component.
![CloudClient](/images/posts/cloudclient-01.png)

With the files saved, I can then create a CloudConfig.properties file

{% highlight powershell %}

{% endhighlight %}

The source gist can be found below

{% gist 479cdcd4609b03b58c3fbfb2df3f30f5 %}

