---
layout: post
title: Install ZFS on Ubuntu Trusty 14.04
categories:
# v3

---

# {{ page.title }}

In this blog post I'll install [ZFS-on-Linux](http://zfsonlinux.org/) (ZoL) on trusty old Ubuntu Trusty 14.04.

ZFS is an amazing file system that is now also usable on Linux. One of ZFS' best features is that it can "self heal" as it is a checksumming file system. Also it can use SSDs in a couple of different ways, such as the ZIL drive and the L2ARC cache.

There are other interesting file systems and ways to cache with solid state drives. [btrfs](https://btrfs.wiki.kernel.org/index.php/Main_Page) is continually getting better (I use it with [Docker](http://serverascode.com/2014/06/09/docker-btrfs.html)) and recently the Linux kernel gained a few ways to do SSD caching: dmcache, flashcache, and bcache.

In my situation I have various media files from short films I've made that I need to backup and protect from bitrot. To do that I've decided to use ZFS on Linux. I worked with ZFS + FreeBSD a bit, but I also want the ability to mount many different types of file systems, and surprisingly FreeBSD doesn't support that many of them. I'm also a big fan of XFS, which I believe FreeBSD only supports in read-only mode. So Linux it is.

## ZoL PPA

The easiest way to get ZoL is to use the ZFS-native PPA. The software-properties-common package is required for the add-apt-repository command.

<pre>
<code>curtis@storage:~$ sudo apt-get install software-properties-common
curtis@storage:~$ sudo add-apt-repository ppa:zfs-native/stable
</code>
</pre>

Now we can install ZoL. Installing will also compile a kernel module.

_NOTE: I'm removing a lot of the output below for brevity; I usually mark that with SNIP!._

<pre>
<code>curtis@storage:~$ sudo apt-get update
curtis@storage:~$ sudo apt-get install -y ubuntu-zfs
SNIP!
zfs.ko:
Running module version sanity check.
 - Original module
   - No original module exists within this kernel
 - Installation
   - Installing to /lib/modules/3.13.0-24-generic/updates/dkms/

depmod....

SNIP!
Setting up ubuntu-zfs (8~trusty) ...
Processing triggers for libc-bin (2.19-0ubuntu6) ...
</code>
</pre>

Now load the module.

<pre>
<code>curtis@storage:~$ modprobe zfs
curtis@storage:~$ lsmod | grep zfs
zfs                  1185541  0
zunicode              331251  1 zfs
zavl                   15010  1 zfs
zcommon                51321  1 zfs
znvpair                89166  2 zfs,zcommon
spl                   175436  5 zfs,zavl,zunicode,zcommon,znvpair
</code>
</pre>

## Configure zpool

I have an older computer that I am using as the zfs backup server. In this example it has two 1.5TB drives that I want to use in a zfs mirror (ie. RAID1). I'll add more storage later but for this example just the two 1.5TB drives, sdb and sdd. They were previously used elsewhere and need to be reformatted for zfs.

<pre>
<code>curtis@storage:~$ lsblk
NAME   MAJ:MIN RM   SIZE RO TYPE MOUNTPOINT
sda      8:0    0 119.2G  0 disk
├─sda1   8:1    0 113.3G  0 part /
├─sda2   8:2    0     1K  0 part
└─sda5   8:5    0     6G  0 part [SWAP]
sdb      8:16   0   1.4T  0 disk
├─sdb1   8:17   0   200M  0 part
├─sdb2   8:18   0   1.4T  0 part
└─sdb3   8:19   0   128M  0 part
sdc      8:32   0 465.8G  0 disk
├─sdc1   8:33   0    64K  0 part
├─sdc2   8:34   0   462G  0 part
└─sdc3   8:35   0   3.8G  0 part
sdd      8:48   0   1.4T  0 disk
├─sdd1   8:49   0   200M  0 part
├─sdd2   8:50   0   1.4T  0 part
└─sdd3   8:51   0   128M  0 part
sr0     11:0    1  1024M  0 rom
</code>
</pre>

We'll create a zpool mirror callled tank.

<pre>
<code>curtis@storage:~$ sudo zpool create tank mirror sdb sdd
curtis@storage:~$ zfs list
NAME   USED  AVAIL  REFER  MOUNTPOINT
tank  91.5K  1.34T    30K  /tank
</code>
</pre>

Interestingly zfs didn't warn me about reformatting.

There is now a /tank directory of about ~1.4TB.

<pre>
<code>curtis@storage:~$ df -h | grep tank
tank            1.4T     0  1.4T   0% /tank
</code>
</pre>

Now create another file system on tank. Note the casesensitivity=mixed for use with Windows.

<pre>
<code>curtis@storage:/tank$ zfs create -o casesensitivity=mixed tank/bup
</code>
</pre>

## Samba and ZFS

As stated previously, I want to use this as a backup server. I do a lot of work with video and audio files and that is all, unfortunately, done from a windows workstation. So I want to be able to backup from Windows to the ZoL backup server. I'll use samba (SMB) to do that.

Please note that I haven't used samba in years, so I'm not quite sure this is the right way to go about it. But it is working for me. :)

First, install samba.

<pre>
<code>curtis@storage:~$ sudo apt-get install samba
</code>
</pre>

Now we can create a file system in /tank and share that via SMB.

<pre>
<code>curtis@storage:~$ sudo zfs set sharesmb=on tank/bup
curtis@storage:~$ sudo chown curtis:curtis /tank/bup
</code>
</pre>

Check what zfs thinks about the share status with regards to samba and nfs.

<pre>
<code>root@storage:/var/log/samba# sudo zfs get sharesmb,sharenfs
NAME      PROPERTY  VALUE     SOURCE
tank      sharesmb  on        local
tank      sharenfs  off       default
tank/bup  sharesmb  on        local
tank/bup  sharenfs  off       default
</code>
</pre>

Based on [this blog post](http://unix.stackexchange.com/questions/97812/fedora-19-zfsonlinux-how-to-configure-cifs-share) I added the below to /etc/samba/smb.conf and restarted smbd and nmbd. These settings may or may not be appropriate for your use case.

<pre>
<code>usershare path = /var/lib/samba/usershares
usershare max shares = 100
usershare allow guests = yes
usershare owner only = n
</code>
</pre>

Next, add a samba user.

<pre>
<code>root@storage:/var/log/samba# sudo smbpasswd -a curtis
New SMB password:
Retype new SMB password:
Added user curtis.
</code>
</pre>

Finally I can connect to that server with \\storage\tank_bup or the server and share should be browsable from the Windows workstation, assuming they are on the same network, and in this case they are.

## Conclusion

In this post I've done a couple things:

- Install ZFS on Linux
- Create a pool with mirrored drives
- Configure a samba share to access from Windows

So far the performance has been fine. I get about 111MB/s write which is basically as fast as a 1GB network can go.

Soon I'll add an SSD caching device which will get me more IOPS but I've hit the limit on the network.

## Updates

- Commenter Ofer B says _apt-get update_ is necessary, so I added that.
