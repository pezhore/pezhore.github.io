---
layout: single
title:  "Using CloudClient to bulk change vRA Reservations"
date:   2017-12-12
categories: ['Automation']
tags: ['PowerShell', 'VMware']
---
What happens when you have ~500 VMs in vRealize Automation that need to have their reservations changed? You figure out a way to script that tedious process. (This will provide some background and more details on a recent [VMware Communities](https://code.vmware.com/forums/5098/vrealize-automation#577825) post.)

I first started looking at what I was most familiar with - [PowervRA](https://github.com/jakkulabs/PowervRA) - a great community supported, PowerShell wrapper for the vRA API stack. Unfortunately, PowervRA does not offer reservation manipulation - only reporting.

[CloudClient](https://code.vmware.com/web/dp/tool/cloudclient/4.5.0) is a Java application wrapper for the vRA API. Bulk changing reservations is one use case, but several other uses exist in the documentation from that download site.

To use CloudClient in a wrapper script, we must first save our password by interactively launching CloudClient and executing the `login keyfile` command. This command specifies an output location for the encrypted file and prompts for a password. Most CloudClient activities require authentication with both vRA and vRA's IaaS component.
![CloudClient](/images/posts/CloudClient_Login.png)

With the files saved, I can then create a CloudConfig.properties file
{% gist 7852c15bf46f9deb3c9f6c681740420a %}

With the CloudConfig.properties file in place at the same level as the `./bin` folder, I can execute my commands directly from PowerShell

{% highlight powershell %}
.\bin\cloudclient.sh vra machines change reservation --ids "restest01" --reservationName "Windows\ -\ 02"
{% endhighlight %}

This works great for single VMs, but for migrating several hundred - and ensuring the VMs you're requesting are indeed on the wrong reservation takes a bit more effort. If I connect to vRA with PowervRA, I can gather a list of VMs on the "old" reservation and pass that list of applicable servers to the CloudClient command line. Alternatively, if I provide a list of VMs to PowerShell, I can validate which are on the old reservation and update only those.

The final source can be found below:
{% gist 479cdcd4609b03b58c3fbfb2df3f30f5 %}

