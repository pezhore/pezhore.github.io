---
layout: single
title:  "Adventures with CoreOS"
date:   2014-08-11
categories: "Home Lab"
tags: archived coreos
---
I now have three CoreOS boxes setup in my home lab - following the steps here. But just having three CoreOS boxes doing nothing is boring, so I've decided to try and accomplish the following:

* Figure out how to cluster the three CoreOS VMs / get containers to bounce between VMs
* Determine how to add/remove VMs to a cluster automatically
* Run a container for OpenVPN connectivity
* Run a container host a proxy website - maybe [poxy](http://sourceforge.net/projects/poxy/)?

These are some potentially complex goals, so for this post, I'll be dealing with basic setup/prerequisites to getting a clustered CoreOS instance working.

## Networking

Before I do anything, I needed to set some static IPs. This can be accomplished with the presence of a file under `/etc/systemd/network` called `static.network`. (Actually the name doesn't matter too much, but for clarity I opted for this nomenclature.)

```
[Match]
Name=ens32

[Network]
Description=Static IP Config for Home Lab
Address=10.1.1.220/24
Gateway=10.1.1.1
DNS=10.1.1.70
DNS=10.1.1.71
DNS=10.1.1.1
```

Quick explanation for those variables: Name - the name of the NIC whose settings shall be changed, Description - human readable definition for this config, Address/Gateway/DNS - self explanatory.

After this file is present, either reboot or run `sudo systemctl restart systemd-networkd` to have changes take affect. My updated environment had the following config:

* coreos1: 10.1.1.220
* coreos2: 10.1.1.221
* coreos3: 10.1.1.222

## etcd

I needed to get these three VMs configured in a cluster - that's where etcd comes in. But for some reason, wrapping my head around the etcd configuration was proving difficult - until I found the [github docs](https://github.com/coreos/etcd/tree/master/Documentation). The example commands worked with some tweaking:

etcd is in the path, so the ./etcd is not required.
The example runs on a single server, so either multiple sessions are required, or each process could be put in the background
After I was successfully able to run things on a single VM, I restored to previous snapshot (otherwise there was a port conflict in the next steps). I tweaked the commands as follows:

coreos1:

`etcd -peer-addr 10.1.1.220:7001 -addr 10.1.1.220:4001 -bind-addr 0.0.0.0 -data-dir machines/coreos1 -name coreos1`

coreos2:

`etcd -peer-addr 10.1.1.221:7001 -addr 10.1.1.221:4001 -bind-addr 0.0.0.0 -peers 10.1.1.220:7001,10.1.1.222:7001 -data-dir machines/coreos2 -name coreos2`

coreos3:

`etcd -peer-addr 10.1.1.222:7001 -addr 10.1.1.222:4001 -bind-addr 0.0.0.0 -peers 10.1.1.221:7001,10.1.1.222:7001 -data-dir machines/coreos3 -name coreos3`

I verified things worked by going to `http://<coreos ip>:4001/v2/leader` (where `<coreos ip>` is replaced by each of my VM's IP addresses) - each returned the correct leader. Time to put it in a static config file.

## etcd.conf

The etcd config file resides in /etc/etcd/ and the basic format can be derived from the github docs. I opted to remove all the extra settings and only include the flags I used in my command line runs. Each of the three VMs has their own etcd.conf file:

coreos1

```
addr = "10.1.1.220:4001"
bind_addr = "0.0.0.0"
peers = []
name = "coreos1"

[peer]
addr = "10.1.1.220:7001"
bind_addr = "0.0.0.0"
```

coreos2

```
addr = "10.1.1.221:4001"
bind_addr = "0.0.0.0"
peers = ["10.1.1.220:7001","10.1.1.222:7001"]
name = "coreos2"

[peer]
addr = "10.1.1.221:7001"
bind_addr = "0.0.0.0"
```

coreos3

```
addr = "10.1.1.222:4001"
bind_addr = "0.0.0.0"
peers = ["10.1.1.220:7001","10.1.1.221:7001"]
name = "coreos3"

[peer]
addr = "10.1.1.222:7001"
bind_addr = "0.0.0.0"
```

At this point, running etcd on each of the three servers successfully joins them to the cluster, however etcd doesn't start automatically upon reboot. In the future posts, I'll be trying to figure out cloud-config - a file which helps define not only auto-starting of services like etcd, but also defines hostnames and some other cool stuff, as well as the config drive which holds this config file.
