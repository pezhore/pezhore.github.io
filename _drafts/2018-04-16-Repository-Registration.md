---
layout: single
title:  "Automatic NuGet Repository Registration"
date:   2018-04-16
categories: ['Source Control']
tag: ['Automation']
---
In a previous post, I covered how git branches, automated testing, and a rudimentary pipeline can be used to promote code in a controlled manner. This post is dedicated to the process for preparing an environment (assuming separate dev, qa, and prod) with the appropriate versions of PowerShell modules. The general idea is to have a "first run" script to prepare the environment for execution.

It performs the following steps:

1. Based on a provided environment (dev, qa, prod) ensure the corresponding Artifactory repository is registered.
2. For each provided module
    1. Check the dev/qa/prod repository for each of the provided modules - comparing the latest available repository module version with the locally installed module versions.
    2. If the remote has a newer version, install it locally.
    3. Load the latest version of the specified module
3. Exit/cleanup

After the environment prep script runs, the main automation is executed leveraging the imported modules.


