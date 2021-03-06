---
layout: post
title: A year with OpenStack Essex
categories:
header_image: http://c179631.r31.cf0.rackcdn.com/openstack.jpg
---

# {{ page.title }}

I've been using an OpenStack Essex installation (cluster? I never know what to call it) for about a year now. OpenStack has gone through several new versions, Essex to Folsom to Grizzly and now Havana was released only a few days ago. But here I am back on Essex. I believe it has been possible to upgrade OpenStack since Folsom, but because I'm running Essex I think upgrading would mean forklift style. My current workplace has a few OpenStack clouds running, and one of them has gone from Folsom to Grizzy and will eventually go to Havana and beyond, but for this one, we're stuck on Essex.

The system is made up of eight Dell C6220s with one "cloud controller" (ie. not highly available) and seven compute nodes. As of right now we have had almost 3500 virtual machines booted. That's not bad considering this OpenStack cloud is only used by one application (Apache VCL).

<pre>
<code>mysql> select id from instances order by id DESC limit 1;
+------+
| id   |
+------+
| 3355 |
+------+
1 row in set (0.01 sec)
</code>
</pre>
<p></p>

## Striped solid state drives

Because we use OpenStack to essentially provide a VDI-lite service, one in which the VMs don't store any state, and Windows instances are heavy IOPS users, we moved to using striped solid state drives. We've been running the compute nodes that way for about three months and so far so good (prior to that they were running on spinning disks).

<pre>
<code>curtis@c2:~$ cat /proc/mdstat 
Personalities : [linear] [multipath] [raid0] [raid1] [raid6] [raid5] [raid4] [raid10] 
md127 : active raid0 sda5[0] sdc5[2] sdb5[1]
      2597588736 blocks super 1.2 256k chunks
      
md0 : active raid1 sda1[0] sdb1[1]
      524224 blocks [2/2] [UU]
      
md1 : active raid0 sdb3[1] sda3[0] sdc3[2]
      106953984 blocks super 1.2 256k chunks
unused devices: <none>
curtis@c2:~$ sudo smartctl -i /dev/sda
smartctl 5.41 2011-06-09 r3365 [x86_64-linux-3.2.0-51-generic] (local build)
Copyright (C) 2002-11 by Bruce Allen, http://smartmontools.sourceforge.net

=== START OF INFORMATION SECTION ===
Device Model:     Crucial_CT960M500SSD1
Serial Number:    1324093FD9AB
LU WWN Device Id: 5 00a075 1093fd9ab
Firmware Version: MU02
User Capacity:    960,197,124,096 bytes [960 GB]
Sector Sizes:     512 bytes logical, 4096 bytes physical
Device is:        Not in smartctl database [for details use: -P showall]
ATA Version is:   8
ATA Standard is:  ATA-8-ACS revision 6
Local Time is:    Mon Oct 21 13:57:51 2013 MDT
SMART support is: Available - device has SMART capability.
SMART support is: Enabled
</code>
</pre>
<p></p>

## Issues

So far we have only had a couple issues.

The first is there was a security patch added to the nova package that double checked the virtual size of an image. I updated OpenStack's packages and we couldn't boot Windows images, so I had to roll back to a previous version. I haven't had time to figure out what the issue is, so we have frozen our OpenStack version while I find time to determine the cause. I'm quite sure that this is not an OpenStack issue, rather a configuration issue in terms of the Windows image size.

I also have a problem in which it seems the Windows disk images are mounted via ndb, I assume for some kind of file injection, but then not released. This can cause deleted image files to be hung onto by the file system, so disk space can fill up, but du will not be able to explain why (had to use lsof and look for deleted files).

<pre>
<code>curtis@c2:~$ mount | grep nbd
/dev/mapper/nbd15p1 on /tmp/openstack-disk-mount-tmp0lvNeR type fuseblk 
(rw,nosuid,nodev,allow_other,blksize=4096)
/dev/mapper/nbd14p1 on /tmp/openstack-disk-mount-tmpMrUDJF type fuseblk 
(rw,nosuid,nodev,allow_other,blksize=4096)
/dev/mapper/nbd13p1 on /tmp/openstack-disk-mount-tmp5p_Jcy type fuseblk 
(rw,nosuid,nodev,allow_other,blksize=4096)
/dev/mapper/nbd12p1 on /tmp/openstack-disk-mount-tmpOv9v9X type fuseblk 
(rw,nosuid,nodev,allow_other,blksize=4096)
SNIP!
</code>
</pre>

While this does seem like some kind of bug, the held nbd mounts/images are limited to 16 by a configuration option, and while this shouldn't be happening it doesn't seem to be affecting normal operation, at least in our limited use case. This is another issue that needs more investigation. Maybe someone will read this post and let me know what is going on. I did email the OpenStack list about the issue but no one replied. Not sure how many people are using OpenStack Essex with Windows 7.

## Essex has worked great

While I've got a couple issues to look into, OpenStack Essex has worked great. Certainly no major issues to report, but I'm still looking forward to eventually upgrading this system.
