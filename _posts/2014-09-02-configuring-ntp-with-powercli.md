---
layout: single
author_profile: true
classes: wide
title:  "Configuring NTP with PowerCLI"
date:   2014-09-02
categories: ['Home Lab']
tags: archived powershell powercli
---
This is going to be another quick one. A new cluster was configured without any NTP settings. I threw this bit of code together to set our default NTP servers, open the firewall to allow for NTP traffic, and set the service to started/automatic.

I cleaned it up a bit and you may find it below.

{% highlight powershell %}
[CmdletBinding()]
param ($NTPServers,
        [Parameter(Mandatory=$true)]
        $TargetDatacenter)
BEGIN{

    # If no NTP Servers are provided, go with the defaults for North America
    if (!$NTPServers) {
        $NTPServers = "0.pool.ntp.org", "1.pool.ntp.org", "2.pool.ntp.org", "3.pool.ntp.org"
    }
}
PROCESS{

    # Get all the hosts in this particular datacenter (Assumes you're already connected to vCenter)
    $VMHosts = get-datacenter $TargetDatacenter | get-vmhost

    # Go through each host
    foreach($VMHost in $VMHosts){

        # Add each NTP Server one at a time
        foreach( $NTPServer in $NTPServers) {
            Add-VmHostNtpServer -VMHost $VMHost -NtpServer $NTPServer
        }

        # Open up the firewall
        Get-VMHostFirewallException -VMHost $VMHost | ?{$_.Name -eq "NTP client"} | Set-VMHostFirewallException -Enabled:$true

        # Start the NTP service and set it to automatic
        Get-VMHostService -VMHost $VMHost | ?{$_.key -eq "ntpd"} | Start-VMHostService
        Get-VMHostService -VMHost $VMHost | ?{$_.key -eq "ntpd"} | Set-VMHostService -Policy Automatic
    }
}
END {

    # Last minute debugging (if necessary)
    Write-Debug "Anything Else?"
}
{% endhighlight %}
