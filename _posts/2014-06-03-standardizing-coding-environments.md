---
layout: single
title:  "Standardizing coding environments with Dropbox"
date:   2014-06-03
categories: ['Source Control']
tags: ['archived']
---

In a previous post, I showed how I rely on Dropbox for synchronizing a few Windows application data folders. I also use it for a rather ghetto source code replication/backup solution.

Similar to the Windows settings sync, I have a scripting folder on my Dropbox that contains several subfolders - each broken down by language (Ruby, PowerShell, Perl, etc). The tree structure is shown below:

```
pezhore@mcpp:~/Dropbox/scripting/git-code$ tree -d
.
├── bat
├── OpenPastebin-v0.3
├── perl
├── powershell
├── puppet
├── python
├── ruby
├── SORT ME
│   ├── BCLU
│   ├── bulkfilemanager
│   ├── create-db
│   ├── Powershell
├── VBScript
├── viewgit
└── vip
```

# The good stuff

* Dropbox works pretty well with those few git controlled projects (with the caveat below). Its version control is server side and doesn't affect contents of the synced files.
* Automatic offsite code backup/replication to all connected/authorized servers, laptops, desktops.
* Primitive version control (sort of).

# The bad stuff

* If Dropbox goes away, any syncing will stop and my code may be at an inconsistent state, until I identify another cloud sync solution.
* I'm not sure it's a great idea to store secure code on Dropbox. Luckily, the vast majority (if not all of it) lacks hard-coded sensitive information.
* Conflicted copies fill up my .git folders for proper version controlled scripting projects.

# Secret Sauce

Just as in my other post, I'll use junctions to link the scripting folder to `c:\scripts\` or in on my Linux boxes to `~/dbsrc/` via hard links.

In a future iteration, I may look into using an encrypted volume in lieu of raw files. This will break the primitive version control and will open me up to possible corruption, but then at least my code will be encrypted (all ~100MB of it). If I opted to go this route, I would probably leverage the previous version of TrueCrypt (7.1a).
