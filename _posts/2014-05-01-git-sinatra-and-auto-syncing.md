---
layout: single
author_profile: true
classes: wide
title:  "Git, Sinatra, and Auto-Syncing Source Control"
date:   2014-05-01
categories: ['Source Control']
tag: archived
---

I develop code (PowerShell mostly) from my work desktop - code/cmdlets/scripts that are destined to run on various remote servers. I got tired of manually copying the scripts to each server, or manually pulling down the git repository on each individual server. I found a solution, perhaps not the best solution... but something that works for me.

This is pretty heavily inspired by how Puppet handles code repositories in their training sessions.

## Sinatra

Sinatra is a Ruby DSL designed for quickly creating web applications. It can be as simple as installation:

`gem install sinatra`

Creating a file:

```language-ruby
# myapp.rb
require 'sinatra'
get '/' do
  'Hello world!'
end
```

And running it:

`ruby myapp.rb`

## Gitlab Post Hook

I use gitlab at work as an internal shared repository for various code projects. I chose gitlab because (a) it was pretty good looking, and (b) because it was free. Not great reasons, but it turns out I made a decent choice as gitlab easily handles post hooks. Below is an example taken from our internal Gitlab server.

![Gitlab Post Hook](/images/posts/PostHook.png)

Git post hooks will send post web requests to specified URLs when the repository is updated.

# Everything comes together

Simply put, updated code is pushed to gitlab, gitlab submits a post request to sinatra, and sinatra git fetch/resets the target repository. Sample source can be found on my github.

The bulk of the work is done by Sinatra:

```language-ruby
# Sync Git repository to local box
post '/General-Automation' do

    # Set variables - Manifest/environment, git directory, git repo
    # Change these when appropriate
    @gitpath  = "c:/scripts/powershell/General-Automation"
    @gitrepo  = "git@server:bmarsh/general-automation.git"

    # Check to see if the folder exists
    if File.exist?(@gitpath)
            # If it does, just fetch/reset
            %x[cd #{@gitpath} && "c:\\Program Files (x86)\\Git\\bin\\git.exe" fetch --all && "c:\\Program Files (x86)\\Git\\bin\\git.exe" reset --hard origin/master]
    else
            # If it doesn't, git clone
            %x["c:\\Program Files (x86)\\Git\\bin\\git.exe" clone #{@gitrepo} #{@gitpath}]
    end
end
```

My github repository has the full source for turning that Sinatra portion into a Windows service - ensuring the syncing continues even after reboot. There's some tricks to getting git/ssh to work when run as a System service, namely copying over the ssh keys to the system profile (more details on this blog). The absolute path to git.exe is also required. If I were to change the Run As user to something other than System, I could probably get around both of these constraints.

# The good

* Any updates to gitlab are immediately pushed down to hosts.
* No more manual pulls/file copies.
* Good bye configuration drift!

# The bad

* Yet another service to monitor.
* Another port/program must be running.
* Network connectivity is required between everything (my laptop to gitlab, gitlab to each server).
