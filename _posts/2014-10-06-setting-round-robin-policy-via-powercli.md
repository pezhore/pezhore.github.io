---
layout: single
title:  "Setting round robin policy via PowerCLI"
date:   2014-10-06
categories: VMware
tags: archived powercli
---
Another quick PowerCLI script for today. We recently discovered that multiple hosts had datastores configured with fixed path that should have been configured as round robin.

After first verifying that a cluster did not contain any RDMs, this script goes host-by-host finding all Fibre Channel connected Datastores. It checks to see if each datastore's multipath policy is configured for Round Robin, and if not, the policy is configured correctly.

```language-powercli

$cluster = "cluster01"

# Check to see if this cluster has any RDMs
if ((get-cluster $cluster | Get-VM | Get-HardDisk -DiskType "RawPhysical","RawVirtual" | Select Parent,Name,DiskType,ScsiCanonicalName,DeviceName | measure).count -ne 0) {
    Write-Host -ForegroundColor Red "There are RDMs in this cluster. Skipping!"
} else {

    # Get an array of hosts in this cluster
    $hosts = get-cluster $cluster |Get-VMHost

    # For each host
    foreach ($vmhost in $hosts) {
        write-host "Proceeding with host $($vmhost.name)"

        $vmhost | Get-VMHostHba -Type "FibreChannel" | Get-ScsiLun -LunType "disk"|
                    where {$_.MultipathPolicy -ne "RoundRobin"}|Set-ScsiLun -MultipathPolicy RoundRobin

    }
}
```