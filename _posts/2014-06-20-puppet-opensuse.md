---
layout: single
author_profile: true
classes: wide
title:  "Puppet & OpenSUSE"
date:   2014-06-20
categories: ['Home Lab']
tags: ['archived', 'Source Control', 'configuration management']
---
I've been using the Zabbix Appliance to handle monitoring/graphing of my home lab's resources to much success, but after recently expanding my Puppet Enterprise setup I decided to try and manage Zabbix via puppet.

## Background

At the time of this writing, the Zabbix appliance is an OpenSUSE 12.3 VM. While SUSE Linux Enterprise is on the [Puppet Supported Platforms List](http://docs.puppetlabs.com/guides/platforms.html#linux), OpenSUSE is not.

I decided to roll the dice (after first taking a snapshot of the Zabbix appliance for rollback purposes.

## The Attempt

Tried to do just `zypper install puppet`, then run `puppet agent -t`, but there was a pretty nasty problem:

```
Notice: Finished catalog run in 0.47 seconds
Debug: report supports formats: b64_zlib_yaml pson raw yaml; using pson
Debug: report supports formats: b64_zlib_yaml pson raw yaml; using pson
Debug: report supports formats: b64_zlib_yaml pson raw yaml; using pson
Error: Could not send report: Error 400 on SERVER: Could not intern from pson: undefined method `intern' for nil:NilClass
/usr/lib64/ruby/vendor_ruby/1.9.1/puppet/indirector/rest.rb:177:in `is_http_200?'
/usr/lib64/ruby/vendor_ruby/1.9.1/puppet/indirector/rest.rb:145:in `save'
/usr/lib64/ruby/vendor_ruby/1.9.1/puppet/indirector/indirection.rb:266:in `save'
/usr/lib64/ruby/vendor_ruby/1.9.1/puppet/configurer.rb:200:in `send_report'
/usr/lib64/ruby/vendor_ruby/1.9.1/puppet/configurer.rb:194:in `run'
/usr/lib64/ruby/vendor_ruby/1.9.1/puppet/agent.rb:45:in `block (5 levels) in run'
/usr/lib64/ruby/vendor_ruby/1.9.1/puppet/agent/locker.rb:20:in `lock'
/usr/lib64/ruby/vendor_ruby/1.9.1/puppet/agent.rb:45:in `block (4 levels) in run'
/usr/lib64/ruby/1.9.1/sync.rb:227:in `sync_synchronize'
/usr/lib64/ruby/vendor_ruby/1.9.1/puppet/agent.rb:45:in `block (3 levels) in run'
/usr/lib64/ruby/vendor_ruby/1.9.1/puppet/agent.rb:119:in `with_client'
/usr/lib64/ruby/vendor_ruby/1.9.1/puppet/agent.rb:42:in `block (2 levels) in run'
/usr/lib64/ruby/vendor_ruby/1.9.1/puppet/agent.rb:84:in `run_in_fork'
/usr/lib64/ruby/vendor_ruby/1.9.1/puppet/agent.rb:41:in `block in run'
/usr/lib64/ruby/vendor_ruby/1.9.1/puppet/application.rb:175:in `call'
/usr/lib64/ruby/vendor_ruby/1.9.1/puppet/application.rb:175:in `controlled_run'
/usr/lib64/ruby/vendor_ruby/1.9.1/puppet/agent.rb:39:in `run'
/usr/lib64/ruby/vendor_ruby/1.9.1/puppet/application/agent.rb:338:in `onetime'
/usr/lib64/ruby/vendor_ruby/1.9.1/puppet/application/agent.rb:312:in `run_command'
/usr/lib64/ruby/vendor_ruby/1.9.1/puppet/application.rb:346:in `block (2 levels) in run'
/usr/lib64/ruby/vendor_ruby/1.9.1/puppet/application.rb:438:in `plugin_hook'
/usr/lib64/ruby/vendor_ruby/1.9.1/puppet/application.rb:346:in `block in run'
/usr/lib64/ruby/vendor_ruby/1.9.1/puppet/util.rb:496:in `exit_on_fail'
/usr/lib64/ruby/vendor_ruby/1.9.1/puppet/application.rb:346:in `run'
/usr/lib64/ruby/vendor_ruby/1.9.1/puppet/util/command_line.rb:87:in `execute'
/usr/bin/puppet:4:in `<main>'
```

Errors, errors everywhere. Turns out the particular error only really shows up one other place: [grokbase](http://grokbase.com/t/gg/puppet-users/143nhbt2ad/error-could-not-send-report-error-400-on-server-could-not-intern-from-pson-undefined-method-%60intern-for-nil-nilclass).

Unfortunately, the user's suggested fix (removing the stdlib library) would break several different modules in use by my other puppet managed systems.

## The Fix

It turns out that the default repo's version is a bit outdated:

```
linux-xhfb:~ # puppet --version
dnsdomainname: Name or service not known
3.0.2
```

I found through some searching the OpenSUSE puppet repo, and downloaded the one for my system (found by `cat /etc/os-release`).

A quick wget/rpm -ivh later...

```
linux-xhfb:~ # rpm -ivh puppet-3.6.2-2.1.x86_64.rpm
warning: puppet-3.6.2-2.1.x86_64.rpm: Header V3 DSA/SHA1 Signature, key ID a0e46e11: NOKEY
Preparing...                          ################################# [100%]
Updating / installing...
   1:puppet-3.6.2-2.1                 ################################# [100%]
linux-xhfb:~ # puppet agent -t --noop
...
dnsdomainname: Name or service not known
Info: Caching catalog for linux-xhfb.peztopia.local
Info: Applying configuration version '1403169911'
Notice: Finished catalog run in 0.68 seconds
```

Now I have a working Zabbix appliance managed by Puppet (which is for the moment not doing anything). In the future I'll add basic things like package management (git, vim, etc) and some config management (vim packages).