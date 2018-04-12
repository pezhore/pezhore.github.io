---
layout: single
title:  "Gitlab, Ubuntu 14.04, and LDAP"
date:   2014-06-10
categories: ['Home Lab']
tags: ['archived', 'Source Control']
---
Time to install source control at home (mostly to organize my home lab's puppet manifests). I started with my typical Ubuntu 14.04 server install and followed the directions on Gitlab's [docs](https://gitlab.com/gitlab-org/gitlab-ce/blob/6-9-stable/doc/install/installation.md).

Unfortunately, when the time came to install postgresql, it failed. I found a [work around](http://ubuntuhandbook.org/index.php/2014/02/install-postgresql-ubuntu-14-04/) that involved installing the official postgresql repos and managed to get through nearly the rest of the Gitlab instructions.

# nginx issues

When it came time to restart nginx, I ran into further problems. The restart failed, and trying to stop, then start also failed but it didn't give any indication of what was wrong.

Tried troubleshooting:

`sudo -u git -H bundle exec rake gitlab:check RAILS_ENV=production`

And noticed this:

```
Check GitLab API access: /usr/local/lib/ruby/2.0.0/net/http.rb:878:in `initialize': Connection refused - connect(2) (Errno::ECONNREFUSED)
        from /usr/local/lib/ruby/2.0.0/net/http.rb:878:in `open'
        from /usr/local/lib/ruby/2.0.0/net/http.rb:878:in `block in connect'
        from /usr/local/lib/ruby/2.0.0/timeout.rb:52:in `timeout'
        from /usr/local/lib/ruby/2.0.0/net/http.rb:877:in `connect'
        from /usr/local/lib/ruby/2.0.0/net/http.rb:862:in `do_start'
        from /usr/local/lib/ruby/2.0.0/net/http.rb:851:in `start'
        from /home/git/gitlab-shell/lib/gitlab_net.rb:76:in `get'
        from /home/git/gitlab-shell/lib/gitlab_net.rb:43:in `check'
        from /home/git/gitlab-shell/bin/check:11:in `<main>'
gitlab-shell self-check failed
  Try fixing it:
  Make sure GitLab is running;
  Check the gitlab-shell configuration file:
  sudo -u git -H editor /home/git/gitlab-shell/config.yml
  Please fix the error above and rerun the checks.
```

The config looked right, ended up dropping it and looking into why nginx wouldn't start.

looked in the logs:

```
cat /var/log/nginx/error.log
2014/06/11 16:28:19 [emerg] 16806#0: a duplicate default server for 0.0.0.0:80 in /etc/nginx/sites-enabled/gitlab:23
2014/06/11 16:28:25 [emerg] 16965#0: a duplicate default server for 0.0.0.0:80 in /etc/nginx/sites-enabled/gitlab:23
2014/06/11 16:30:46 [emerg] 20265#0: a duplicate default server for 0.0.0.0:80 in /etc/nginx/sites-enabled/gitlab:23
```

Turns out the default site was still enabled, deleted that, nginx started just fine.

# Active Directory Integration

I ran into some issues getting LDAPS working with my home domain. Enabling it is simple enough:

```language-yml
  ldap:
    enabled: true
    host: 'dc1.mydomain.local'
    port: 636
    uid: 'sAMAccountName'
    method: 'ssl'
    bind_dn: 'CN=gitlab,OU=ServiceAccounts,OU=localUsers,DC=mydomain,DC=local'
    password: 'password'
    allow_username_or_email_login: true
    base: 'OU=localUsers,DC=mydomain,DC=local'
```

However, when trying to log in, I would get invalid credential messages. There didn't appear to be much by way of logs, so I started to look into how to troubleshoot LDAP errors. I opted to use LDP to see the result of my LDAPS connection attempt:

![Gitlab Post Hook](/images/ldap.png)

Lo and behold, there was an error:

```
ld = ldap_sslinit("dc1.mydomain.local", 636, 1);
Error 0 = ldap_set_option(hLdap, LDAP_OPT_PROTOCOL_VERSION, 3);
Error 81 = ldap_connect(hLdap, NULL);
Server error: <empty>
Error <0x51>: Fail to connect to dc1.mydomain.local.
```

Googling the error `<0x51>` led me to believe that my self-signed ssl cert was to blame. As this is a home lab and gitlab will remain unavailable from the internet, I opted to change the configuration to LDAP.

```language-yml
  ldap:
    enabled: true
    host: 'dc1.mydomain.local'
    port: 389
    uid: 'sAMAccountName'
    method: 'plain' # "tls" or "ssl" or "plain"
    bind_dn: 'CN=gitlab,OU=ServiceAccounts,OU=localUsers,DC=mydomain,DC=local'
    password: 'password'
    allow_username_or_email_login: true
    base: 'OU=localUsers,DC=mydomain,DC=local'
```

A service restart later and I have a functioning Gitlab instance with LDAP authentication!
