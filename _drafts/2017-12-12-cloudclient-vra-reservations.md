---
layout: single
title:  "Using CloudClient to bulk change vRA Reservations"
date:   2017-12-12
categories: ['Automation']
tags: ['archived', 'PowerShell', 'VMware']
---
What happens when you have ~500 VMs in vRealize Automation that need to have their reservations changed? You figure out a way to script that tedious process. (This will provide some background and more details on a recent [VMware Communities](https://code.vmware.com/forums/5098/vrealize-automation#577825) post.)

I first started looking at what I was most familiar with - [PowervRA](https://github.com/jakkulabs/PowervRA) - a great community supported, PowerShell wrapper for the vRA API stack.

The source gist can be found below

{% gist 479cdcd4609b03b58c3fbfb2df3f30f5 %}

