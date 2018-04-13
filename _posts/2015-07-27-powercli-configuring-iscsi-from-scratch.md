---
layout: single
author_profile: true
classes: wide
title:  "PowerCLI - configuring iSCSI from scratch"
date:   2015-07-27
categories: Automation
tags: archived powercli vmware
---
I threw this together a while ago, it will help configure iSCSI on a VM host using PowerCLI. Note that once you do a find/replace to set your iSCSI network (e.g. 192.168.0.0/24 192.168.1.0/24), all you'll need to change is the $thisHost, $lastOctet, and $targets values before running.

Requires magic from [Jonathan Medd's Blog][1] (for the Set-VMHostiSCSIBinding cmdlet).

{% highlight powershell %}
# This configures iSCSI from scratch!

# Set the specific information for this host
$thisHost = Get-VMHost dc2server03*
$lastOctet = 62
$iSCSI_addr1 = "192.168.0.$lastOctet"
$iSCSI_addr2 = "192.168.1.$lastOctet"

# Get the iSCSI dvSwitch Port Groups
$iSCSI_dvPG1 = Get-VDPortgroup "iSCSI dvPG1"
$iSCSI_dvPG2 = Get-VDPortgroup "iSCSI dvPG2"
$iSCSI_dvSw  = Get-VirtualSwitch -name "UCS iSCSI dvSwitch"

# Create a new host network adapter for this host on both dvPGs
New-VMHostNetworkAdapter -VMHost $thisHost -PortGroup $iSCSI_dvPG1 -IP $iSCSI_addr1 -SubnetMask 255.255.255.0 -Mtu 9000 -VirtualSwitch $iSCSI_dvSw  
New-VMHostNetworkAdapter -VMHost $thisHost -PortGroup $iSCSI_dvPG2 -IP $iSCSI_addr2 -SubnetMask 255.255.255.0 -Mtu 9000 -VirtualSwitch $iSCSI_dvSw

# Setup the iSCSI Binding
$thisHost | Set-VMHostiSCSIBinding -HBA vmhba32 -VMKernel "vmk1"
$thisHost | Set-VMHostiSCSIBinding -HBA vmhba32 -VMKernel "vmk2"

# Configure the iSCSI targets
$targets = "192.168.0.10","192.168.0.20","192.168.1.10","192.168.1.20"
        $hba = $thisHost | Get-VMHostHba -Type iScsi | Where {$_.Model -eq "iSCSI Software Adapter"}
        foreach($target in $targets){
            if(Get-IScsiHbaTarget -IScsiHba $hba -Type Send | Where {$_.Address -cmatch $target}){
                Write-Host "The target $target does exist on $thisHost" -ForegroundColor Green
            }
            else{
                Write-Host "The target $target doesn't exist on $thisHost" -ForegroundColor Red
                Write-Host "Creating $target" -ForegroundColor Yellow
                New-IScsiHbaTarget -IScsiHba $hba -Address $target | Out-Null
                Write-Host "done..." -ForegroundColor Green
            }
        }

# Rescan to find iSCSI datastores
$thisHost | Get-VMHostStorage -RescanAllHba

# Just a little clean up, rename any default local datastores
$thisHost | get-datastore datastore* | set-datastore -name "$($thisHost.name.Substring(0,$thisHost.Name.IndexOf(".")))_boot"
{% endhighlight %}

[1]: http://www.jonathanmedd.net/2013/07/using-powercli-for-iscsi-vmkernel-port-binding.html
