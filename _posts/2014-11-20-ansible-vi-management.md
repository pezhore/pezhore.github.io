---
layout: single
title:  "Ansible & VI Management"
date:   2014-11-20
categories: ['Home Lab']
tags: ['archived', 'configuration management']
---
I've done it with Puppet, and with Chef - but I thought I would give it a go with Ansible. VI management! Today I spent an hour brute forcing my way through ansible with one goal in mind: create a playbook that will get vim installed and configured with the plugins I use on a daily basis.

# Part One: Installing VIM

Create a vimplaybook.yaml file that ensures my .vimrc file is present with the correct content (just one line actually), then gradually add tasks until everything works!

First things first: we're targeting the localhost, although this can be changed later. I'm running this as my local user account, pezhore and there's just one task at first: installing vim using apt, specifically looking for the latest version.

```language-yml
---
- hosts: localhost
    remote_user: pezhore
    tasks:
    - name: ensure vim is installed
      apt: pkg=vim state=latest
```

Let's see how this goes...

```
pezhore@ubuntu:~⟫ ansible-playbook vimplaybook.yaml

PLAY [localhost] **************************************************************

GATHERING FACTS ***************************************************************
ok: [localhost]

TASK: [ensure vim is installed] ***********************************************
failed: [localhost] => {"failed": true}
stderr: E: Could not open lock file /var/lib/dpkg/lock - open (13: Permission denied)
E: Unable to lock the administration directory (/var/lib/dpkg/), are you root?

msg: 'apt-get install 'vim' ' failed: E: Could not open lock file /var/lib/dpkg/lock - open (13: Permission denied)
E: Unable to lock the administration directory (/var/lib/dpkg/), are you root?


FATAL: all hosts have already failed -- aborting

PLAY RECAP ********************************************************************
            to retry, use: --limit @/home/pezhore/vimplaybook.yaml.retry

localhost                  : ok=1    changed=0    unreachable=0    failed=1
```

Well... crap. Looks like I'm not a root account, but I do have sudoer's privs.

Take two:

```language-yml
- hosts: localhost
    remote_user: pezhore
    tasks:
    - name: ensure vim is installed
      apt: pkg=vim state=latest
      sudo: yes
```

Now that I have specified `sudo: yes`, I'll need to run ansible-playbook with --ask-sudo-pass

```
pezhore@ubuntu:~⟫ ansible-playbook vimplaybook.yaml --ask-sudo-pass
sudo password:

PLAY [localhost] ***************************************************

GATHERING FACTS ***************************************
ok: [localhost]

TASK: [ensure vim is installed] ******************************************
changed: [localhost]
```

Much better.

# Part Two: .vimrc

My .vimrc file contains one line I care about... the execute pathogen command. (Pathogen is an autoloader for vim plugins.)

First, I check to see if the file exists, and if it doesn't I use the file/touch command to create a blank. At this point, with the file existing, I check to make sure a particular line exists. Cool stuff here? The `when: not vimrc.stat.exists` \- the `exists` portion is part of the stat functionality and the `when:` directive is a useful introduction to conditional tasks. Also, the `inlinefile` is something that was rather difficult to achieve with Puppet - having a regex match for a single line and altering said line is hugely helpful.

```language-yml
- stat: path=/home/pezhore/.vimrc
  register: vimrc
- name: create vimrc file
  file: path=/home/pezhore/.vimrc state=touch
  when: not vimrc.stat.exists
- name: ensure pathogen is enabled in .vimrc
  lineinfile: dest=/home/pezhore/.vimrc regexp=^execute pathogen line="execute pathogen#infect()"
```

# Part Three: Git All the Things!

As I stated before, I use pathogen for autoloading my various modules (sleuth, tabular, sensible). All of these modules are available through git. Cloning is rather easy with Ansible.

```language-yml
- name: git clone pathogen
git: repo=https://github.com/tpope/vim-pathogen.git
     dest=/home/pezhore/.vim
- name: create bundle directory
file: path=/home/pezhore/.vim/bundle/ state=directory
- name: git clone sleuth
git: repo=https://github.com/tpope/vim-sleuth.git
     dest=/home/pezhore/.vim/bundle/sleuth
- name: git clone tabluar
git: repo=https://github.com/godlygeek/tabular.git
     dest=/home/pezhore/.vim/bundle/tabular
- name: git clone sensible
git: repo=https://github.com/tpope/vim-sensible.git
     dest=/home/pezhore/.vim/bundle/sensible
```

This at first run appears to be working perfectly (unnecessary portions are removed):

```
TASK: [git clone pathogen] ****************************************************
changed: [localhost]

TASK: [create bundle directory] ****************************************************
changed: [localhost]

TASK: [git clone sleuth] ****************************************************
changed: [localhost]

TASK: [git clone tabluar] ****************************************************
changed: [localhost]

TASK: [git clone sensible] ****************************************************
changed: [localhost]

PLAY RECAP ****************************************************
localhost                  : ok=9    changed=5    unreachable=0    failed=0
```

And we're done! Or are we? (Hint: We're not.)

# Part Four: Idempotency!

Ansible (like Puppet/Chef) should be idempotent - you should be able to run the same playbook repeatedly and have the system state be the same after every run. But when running this exactly same playbook immediately afterwards, we see a problem with the pathogen git section.

```
TASK: [git clone pathogen] ***********************************************
changed: [localhost]

TASK: [create bundle directory] ***********************************************
ok: [localhost]

TASK: [git clone sleuth] ***********************************************
ok: [localhost]

TASK: [git clone tabluar] ***********************************************
ok: [localhost]

TASK: [git clone sensible] ***********************************************
ok: [localhost]

PLAY RECAP ************************************
localhost                  : ok=9    changed=1    unreachable=0    failed=0
```

Checking `git status` in the /home/pezhore/.vim folder gives us a hint as to why this is the case:

```
pezhore@ubuntu:~/.vim⟫ git status
On branch master
Your branch is up-to-date with 'origin/master'.

Untracked files:
    (use "git add ..." to include in what will be committed)

        bundle/

nothing added to commit but untracked files present (use "git add" to track)
```

So now we have a problem. By creating the bundle directory (required for the other vim modules), we have changed git. And at every run of the ansible playbook, it will see that the local git is not the same as the remote - requiring a fresh pull.

I did a rather messy work around, frankly I'd be open for alternatives:

```language-yml
- name: git clone pathogen
git: repo=https://github.com/tpope/vim-pathogen.git
     dest=/home/pezhore/.vim/.pathogen
- file: src=/home/pezhore/.vim/autoload dest=/home/pezhore/.vim/.pathogen/autoload
```

This section above clones git into a hidden directory `/home/pezhore/.vim/.pathogen`, then creates a symlink from `.pathogen/autoload` (the necessary directory for vim to find the `execute pathogen#infect()` directive) to .vim/autoload.

Finally, we have our ideal idempotency.

```
TASK: [git clone pathogen] ****************************************************
ok: [localhost]

TASK: [file src=/home/pezhore/.vim/autoload dest=/home/pezhore/.vim/.pathogen/autoload] ***
ok: [localhost]

TASK: [create bundle directory] ***************************************************
ok: [localhost]

TASK: [git clone sleuth] ***************************************************
ok: [localhost]

TASK: [git clone tabluar] ***************************************************
ok: [localhost]

TASK: [git clone sensible] ****************************************************
ok: [localhost]

PLAY RECAP ****************************************************
localhost                  : ok=10   changed=0    unreachable=0    failed=0
```

The final playbook is below:

```language-yml
---
- hosts: localhost
    remote_user: pezhore
    tasks:
    - name: ensure vim is installed
    apt: pkg=vim state=latest
    sudo: yes
    - stat: path=/home/pezhore/.vimrc
    register: vimrc
    - debug: msg="vimrc does not exist"
    when: not vimrc.stat.exists
    - name: create vimrc file
    file: path=/home/pezhore/.vimrc state=touch
    when: not vimrc.stat.exists
    - name: ensure pathogen is enabled in .vimrc
    lineinfile: dest=/home/pezhore/.vimrc regexp=^execute pathogen line="execute pathogen#infect()"
    - name: git clone pathogen
    git: repo=https://github.com/tpope/vim-pathogen.git
         dest=/home/pezhore/.vim/.pathogen
    - file: src=/home/pezhore/.vim/autoload dest=/home/pezhore/.vim/.pathogen/autoload
    - name: create bundle directory
    file: path=/home/pezhore/.vim/bundle/ state=directory
    - name: git clone sleuth
    git: repo=https://github.com/tpope/vim-sleuth.git
         dest=/home/pezhore/.vim/bundle/sleuth
    - name: git clone tabluar
    git: repo=https://github.com/godlygeek/tabular.git
         dest=/home/pezhore/.vim/bundle/tabular
    - name: git clone sensible
    git: repo=https://github.com/tpope/vim-sensible.git
         dest=/home/pezhore/.vim/bundle/sensible
```

# Going forward

I'd like to use variables for the repeated `/home/pezhore/` and `/home/pezhore/.vim/` folders, as well as write something that will attempt to perform the vim customization for all users with bash terminals.