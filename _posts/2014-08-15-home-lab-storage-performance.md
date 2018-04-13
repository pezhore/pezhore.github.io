---
layout: single
author_profile: true
classes: wide
title:  "Home Lab storage performance"
date:   2014-08-15
categories: ['Home Lab']
tags: archived
---
I have been noticing my Zabbix server had been alerting on high I/O and CPU wait times on several VMs when multiple servers were powered on or other concurrent high I/O tasks took place. I had a feeling it may have been due to two things:

* The relative lack of RAM compared to the size of my storage (8GB RAM for 3.6TB Storage).
* Synchronous writes (ZFS & VMFS)

FreeNAS configuration specs are hard to come by, but it seems that 8GB is the minimum requirement for my zfs pool size. While researching performance issues with VMware, I came upon this helpful post in the [FreeNAS forums][1] that lead me to believe synchronous writes may be a contributing factor.

Time to figure out a solution...

# More RAM

As my FreeNAS motherboard can take two more sticks of RAM, I grabbed another 2x 4GB sticks of RAM, bringing my total to 16GB. Unfortunately, due to the limitations of my motherboard, I couldn't get ECC RAM - something which slightly concerns me as any memory corruption will be flushed to disk. The cost of a server class motherboard, new processor, and 8GB of ECC RAM is more than I'd like to spend right now - and frankly, at that price, a Synology or QNAP NAS device starts to look pretty good.

# Temporarily Disabling Sync

While waiting for the new RAM to arrive, I opted for one of the four solutions for VMware slowness in the FreeNAS forum post - disabling sync. Unfortunately, it appears that disabling sync will also disable the synchronous writes for the ZFS metadata - risking the entire pool in the event of sudden server shutdown. FreeNAS is currently on a UPS, so power surge/power loss is less of a concern - but overheating, or USB drive failure is much harder to prevent. Luckily, I finally have a reasonable backup solution in place - a combination of rsync for flat files and Veeam backups for VMs, both to my Iomega Storcenter.

With the backups and the UPS, I feel somewhat confident in temporarily disabling sync for my ZFS volume. I realize that I'm playing with fire, but at least with these two precautions, I feel like I'm wearing asbestos clothes (actually, that's a very apropos metaphor).

# Future Plan(s)

Going forward if I want to do this correctly, I should really implement one of the following:

## Mirrored SSDs for ZIL Cache

Getting two small SSDs (say 2x [30GB mSATA][2] and 2x [mSATA/SATA adapters][3]) mirrored for ZIL should handle the bulk of my synchronous write problems. My motherboard has two SATA ports free, and price-wise it looks to be a solution for between $50 to $100. But between the ever present concerns of memory corruption/non ECC RAM and the increased costs for more RAM, I'm wondering if I'm throwing too much money into a lab environment.

## iSCSI

This solution is the recommended one from the FreeNAS forum. Enabling sync for ZFS and relying on the async for iSCSI should lead to a performance improvement without the increased danger of losing ZFS metadata. I've used iSCSI in the past at work, but my attempts to get it running at home have always been met with trouble. There's also something nice about having my VM files written to an easily exportable/browsed file system - something only found with NFS.

## Synology/QNAP/Something else

Another alternative is to just give up and migrate to an appliance that is commercially available/supported. Several past coworkers have used Synology products without any issues, and I like the various plugins and community support for Synology/QNAP. This does leave me with a rather difficult question of what to do with my FreeNAS box.

I could always take the 4x 2TB drives from FreeNAS, move them to Synology, and back-fill the FreeNAS box with cheaper drives. This could give me a better performing backup destination (the mirrored Iomega tends to chug under simultaneous rsync/Veeam backups). The Iomega could then pivot into a NFS share (VMware ISOs?) and a general laptop/PC backup destination.

## Doing nothing

This is probably the most likely first step for going forward. Without any additional financial costs, I can benchmark how well the system performs with the increased RAM - both with sync enabled and disabled. If I feel really ambitious, I'll even pull the additional RAM and run the tests again with only 8GB of RAM.

If it looks like performance has improved enough with the extra RAM, I will probably take the hit and go back to synchronous writes. Otherwise, I'll have to look at iSCSI or the mSATA drives as a viable alternative to a whole new system.

[1]: http://forums.freenas.org/index.php?threads/sync-writes-or-why-is-my-esxi-nfs-so-slow-and-why-is-iscsi-faster.12506/
[2]: http://www.amazon.com/MyDigitalSSD-Bullet-Proof-mSATA-Solid/dp/B00B3X72U4
[3]: http://www.amazon.com/gp/product/B00BGEVUO4/