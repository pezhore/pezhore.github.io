---
layout: single
title:  "Quick Post: Pbis Open"
date:   2014-11-12
categories: ['Home Lab']
tags: archived
---
I use PowerBroker Identity Services [PBIS Open][1] to connect my home lab's Ubuntu servers to my lab domain. There's an issue where new domain users are not configured with bash as their default shell. After a user connects, you can manually edit their entry in `/etc/passwd`, but to set default shell for all new users, you may run the following snippet.

```language-bash
sudo /opt/pbis/bin/regshell set_value '[HKEY_THIS_MACHINEServiceslsassParametersProvidersActiveDirectory]' LoginShellTemplate /bin/bash
sudo /opt/pbis/bin/regshell set_value '[HKEY_THIS_MACHINEServiceslsassParametersProvidersLocal]' LoginShellTemplate /bin/bash
sudo /opt/pbis/bin/lwsm refresh lsass
sudo /opt/pbis/bin/ad-cache --delete-all
```

Lastly, in order to allow domain admins to have full root privileges, I'll add domain admins to sudoers file via visudo (replacing spaces with '^'):

`%DOMAIN\Domain^Admins ALL=(ALL) ALL`

[1]: http://www.powerbrokeropen.org/
