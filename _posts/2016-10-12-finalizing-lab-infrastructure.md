---
layout: single
title:  "Finalizing the lab infrastructure"
date:   2016-10-12
categories: ['Home Lab']
tags: ['archived', 'home lab']
---
In order for my home lab setup to be finalized, I needed to do two things:

* I need to provide "jumpbox" capabilities - with the self-imposed restriction that no ssh/rdp ports will be opened to the outside world. 
* CoreOS IP Schema (Public/Private)
* Setup port forwarding for ports 80 & 443 to the CoreOS cluster

## Jumpbox

I'm using [guacamole][1] to provide this access. As previously stated, I'm isolating my homelab networks from the everything else with rules in my Edgerouter Lite, so the guacamole VM resides inside of the home lab network segment. I'm forwarding to port 8443 to free up the standard 443/80 for the CoreOS cluster.

## CoreOS IP Schema

To better mirror how CoreOS works in the public cloud, I need to set a public and private IP for each of my CoreOS instances. It took some tweaking, but now my PowerShell Script will deploy CoreOS with "public" IP addresses on VLAN 30, and "private" IP addresses on VLAN 40. The etcd service and fleet runs on VLAN 40 while the docker services are running on VLAN 30.

## CoreOS Port Forwarding

I'm forwarding to an HAProxy instance in the home lab segment - I've tested a bit with SSL pass through in the past (I'm sure that'll come up again in the future). HAProxy's check options will cover the cases when services are running on different nodes.

[1]: https://guacamole.incubator.apache.org/
