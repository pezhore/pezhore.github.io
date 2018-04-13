---
layout: single
author_profile: true
classes: wide
title:  "Adventures with CoreOS - Part 2"
date:   2014-08-19
categories: ['Home Lab']
tags: archived coreos
---
In the previous article, I created three CoreOS VMs and got them to talk to each other using etcd. In this article, we'll be going over cloud-config/config-drive and getting an etcd cluster that automatically starts and discovers new nodes using the freely available discovery.etcd.io service.

## Cloud Config

Using the documentation as a good base, I created a custom cloud-config for each of my CoreOS VMs. In the config, I specified the VM's hostname, etcd connection information, and specified that the etcd service should be running. The config is saved as a file, `user_data`.

```language-yml
#cloud-config
hostname: coreos1

coreos:
    etcd:
        name: coreos1
        discovery: "https://discovery.etcd.io/token"
        addr: 10.1.1.220:4001
        peer-addr: 10.1.1.220:7001

    units:
        - name: etcd.service
        command: start
```

## Creating/Mounting the ISO

Now that the `user_data` file was created, we need to get it into an ISO for mounting in VMware. The commands below were taken from the [cloudinit config document][1].

```
# Create a temp directory with the proper folder structure
mkdir -p /tmp/new-drive/openstack/latest

# Copy the user_data file to the proper location
cp user_data /tmp/new-drive/openstack/latest/user_data

# Create the ISO
mkisofs -R -V config-2 -o configdrive.iso /tmp/new-drive

# Move the ISO to the ISOs NFS share
mv configdrive.iso /nfs/isos/coreos1-config.iso

# Remove the temporary folder structure
rm -rf /tmp/new-drive/
```

After the ISO was accessible by VMware, I mounted it on my coreos1 VM and rebooted. Rinse and repeat for the other two (coreos2, coreos3) and I have a working etcd cluster!

## Troubleshooting

Originally, I put the `hostname:` directive under `coreos:` and I had trouble determining what was going on - the hostname wasn't changed, and it didn't appear that etcd was running.

I needed to see the logs from the last cloud-init run, and found the journalctl command: `journalctl _EXE=/usr/bin/coreos-cloudinit`.

## Known Issues & Improvements

This initial cloud config worked pretty well. The only problem I had was when new discovery tokens were used, the etcd service needed to be restarted manually before the node would join the cluster. I'm guessing that's because the first run of etcd registers the node with the discovery service (maybe?).

I wouldn't mind trying to figure out a way of setting a static IP through the cloud-config. If I could programmatically create an ISO with the "next" CoreOS details - `hostname: coreos[1|2|3]`, `IP: 10.1.1.22[1|2|3]`, etc - it would be trivial to create a workflow that creates the ISO, clones an existing VM, and mounts the new ISO.

It also might be cool to figure out how to run my own discovery endpoint. I'm not sure if that requires its own etcd cluster - it would definitely need more research.

[1]: https://coreos.com/docs/cluster-management/setup/cloudinit-config-drive/
