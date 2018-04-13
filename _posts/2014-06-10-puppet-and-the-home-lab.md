---
layout: single
author_profile: true
classes: wide
title:  "Puppet and the Home Lab"
date:   2014-06-10
categories: ['Home Lab']
tags: ['puppet', 'home lab', 'archived', 'configuration management']
---

I've decided to try my hand at managing my ever-growing lab environment using Puppet. Mostly because I'm curious to find use cases at work, but also because I'd like to expand my automation skill set. With that in mind, I'm opting to use Puppet Enterprise and keep my total managed devices to under 10 - allowing me to use the "free" Enterprise version.

# The Layout

To get Puppet going, I'll need a minimum of two Linux VMs - one for Puppet Enterprise and one for Gitlab. Due to my expected workload, they won't need to be particularly beefy; I'm expecting a vCPU/1GB RAM each.

## GitLab

I'm building a git source server ([GitLab](https://www.gitlab.com/)) to keep track of my manifests and internal scripts. I've debated hosting a gitlab instance somewhere offsite... maybe on [Digital Ocean](https://www.digitalocean.com/) where this blog is currently hosted. Upside? It would be running on slightly higher class hardware reducing the possibility of data corruption. Keeping my data offsite also means if there are issues at home (power outage, acts of god, etc) my data is off in a data center somewhere. I'm not sure if that's really worth it, so for now I'm opting to keep the Gitlab instance internal. I will be joining the Gitlab box to my domain - requiring the modern equivalent of Likewise Open, [PowerBroker Open](http://www.powerbrokeropen.org/).

The Gitlab server will also have Puppet Agent installed on it to allow for standardizing basic Linux things (vim config, etc).

## Puppet Master

Puppet Master will also be local, and on my home domain. I will be going off of my old documentation for Puppet 3.0 for LDAP configuration. LDAP integration requires the following two edits.

Uncomment the following: 

**`/etc/puppetlabs/console-auth/cas_client_config.yml`**

```language-yml
activedirectoryldap:
  default_role: admin
  description: Active Directory
```

Edit the following file to match AD settings:

**`/etc/puppetlabs/rubycas-server/config.yml`**

```language-yml
authenticator:
class: CASServer::Authenticators::ActiveDirectoryLDAP
ldap:
  host: homelab.local
  port: 636
  base: dc=homelab,dc=local
  filter: (memberof=CN=Infrastructure Admins,OU=Groups,OU=homelab,DC=local) & !(msExchHideFromAddressLists=TRUE)
  auth_user: puppetsvc
  auth_password: Password_Goes_Here
  encryption: simple_tls
extra_attributes: cn, mail
```

Additionally, the Puppet Master will use [Sinatra](http://www.sinatrarb.com/) to allow for auto-updating of local repositories through [Gitlab's Post Update Hook]({% post_url 2014-05-01-git-sinatra-and-auto-syncing %}).

# Final thoughts

You may have noticed my typing has been in the future-tense. I'm just deploying the VMs now and hoping that the final result will match this post somewhat closely. If need be, I'll post an update later to cover issues I've had with the roll out.