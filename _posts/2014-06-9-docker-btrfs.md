---
layout: post
title: Docker and btrfs
categories:
header_image: /img/met/DP813769.jpg
header_permalink: http://www.metmuseum.org/collection/the-collection-online/search/337702 
---
 
# {{ page.title }}

As I write this the first [Docker convention](https://twitter.com/hashtag/dockercon?src=hash) is going on in San Francisco (sounds funny to write it was "docker convention" instead of "dockercon"...a convention sounds like a scene from Fear and Loathing in Las Vegas), and Docker has hit 1.0 and been declared [production](https://twitter.com/solomonstre/status/476036477973831682) worthy.

Docker has had, for a few versions, the ability to use btrfs as it's backing driver, instead of aufs.

## Install btrfs on Ubuntu 14.04/Trusty

Before installing docker I'm going to setup btfrs.

In this example I have a disk device called /dev/vdb to use with btrfs. I've removed some of the output from the commands.

<pre>
<code>$ sudo su
$ apt-get install btrfs-tools
$ mkfs.btrfs /dev/sdb
$ mkdir /var/lib/docker
$ mount /dev/sdb /var/lib/docker
$ mount | grep btrfs
/dev/sdb on /var/lib/docker type btrfs (rw)
</code>
</pre>

Now that btrfs is installed and a device is formatted and mounted under /var/lib/docker we can install docker.

## Install docker

Next we need to install docker. It's important that we also configure docker to startup with the "-s btrfs" option as well.

<pre>
<code>$ apt-get install docker.io
$ vi /etc/default/docker.io
</code>
</pre>

We need to change the DOCKER_OPTS entry to look like this:

<pre>
<code>$ grep DOCKER_OPTS docker.io 
# Use DOCKER_OPTS to modify the daemon startup options.
DOCKER_OPTS="-s btrfs"
</code>
</pre>

Now restart docker.

<pre>
<code>$ service docker.io restart
</code>
</pre>

We should see that docker is running with "-s btrfs":

<pre>
<code>$ ps ax  |grep [d]ocker
10088 ?        Sl     0:02 /usr/bin/docker.io -d -s btrfs
</code>
</pre>

If it's not running with "-s btrfs" then ensure it's set to do so in /etc/default/docker.io and has been restarted.

I've already created several containers and pulled images, actually just the base busybox image which is quite small, so if I run "btrfs subvolume list /var/lib/docker" I should see some subvolumes have been created.

<pre>
<code>vagrant@host1:~$ sudo btrfs subvolume list /var/lib/docker | head -5
ID 258 gen 11 top level 5 path btrfs/subvolumes/511136ea3c5a64f264b78b5433614aec563103b4d4702f3ba7d4d2698e22c158
ID 259 gen 16 top level 5 path btrfs/subvolumes/42eed7f1bf2ac3f1610c5e616d2ab1ee9c7290234240388d6297bc0f32c34229
ID 260 gen 13 top level 5 path btrfs/subvolumes/c120b7cab0b0509fd4de20a57d0f5c17106f3451200dfbfd8c6ab1ccb9391938
ID 261 gen 13 top level 5 path btrfs/subvolumes/d200959a3e91d88e6da9a0ce458e3cdefd3a8a19f8f5e6a1e7f10f268aea5594
ID 262 gen 15 top level 5 path btrfs/subvolumes/120e218dd395ec314e7b6249f39d2853911b3d6def6ea164ae05722649f34b16
</code>
</pre>

And we can see the size of the btrfs file system.

<pre>
<code>$ btrfs filesystem df /var/lib/docker
Data, single: total=1.01GiB, used=337.56MiB
System, DUP: total=8.00MiB, used=16.00KiB
System, single: total=4.00MiB, used=0.00
Metadata, DUP: total=1.00GiB, used=2.72MiB
Metadata, single: total=8.00MiB, used=0.00
$
$ df -h
Filesystem      Size  Used Avail Use% Mounted on
/dev/sda1        40G  1.2G   37G   4% /
none            4.0K     0  4.0K   0% /sys/fs/cgroup
udev            745M   12K  745M   1% /dev
tmpfs           150M  368K  150M   1% /run
none            5.0M     0  5.0M   0% /run/lock
none            750M     0  750M   0% /run/shm
none            100M     0  100M   0% /run/user
vagrant         233G  185G   49G  80% /vagrant
/dev/sdb         20G  344M   18G   2% /var/lib/docker
$
$ sudo btrfs filesystem show /var/lib/docker
Label: none  uuid: c8f11393-9268-475a-82de-cbd697ab3847
  Total devices 1 FS bytes used 340.30MiB
  devid    1 size 20.00GiB used 3.04GiB path /dev/sdb

Btrfs v3.12
</code>
</pre>

Each image and container has a subvolume.

And, as far as I know at this point, that's it--we're now running docker with btrfs.

## Ansible playbook

I've setup an [Ansible playbook](https://github.com/ccollicutt/vagrant-docker-btrfs) with a [Vagrant](http://vagrantup.com) file that will setup a virtual machine with docker and btrfs configured in the same way that I describe in this blog post. Vagrant will automatically provision the virtual machine using Ansible.

<pre>
<code>$ git clone https://github.com/ccollicutt/vagrant-docker-btrfs
$ cd vagrant-docker-btrfs
$ vagrant up --provider virtualbox
Bringing machine 'host1' up with 'virtualbox' provider...
==> host1: Importing base box 'trusty64'...
SNIP!
TASK: [add alias for vagrant user docker = docker.io] ************************* 
changed: [host1]

NOTIFIED: [restart docker] **************************************************** 
changed: [host1]

PLAY RECAP ******************************************************************** 
host1                      : ok=17   changed=16   unreachable=0    failed=0 
$  
</code>
</pre>

Now that that is booted up and configured, we can login and check a couple things.

<pre>
<code>$ vagrant ssh
SNIP!
vagrant@host1:~$ sudo btrfs subvolume list /var/lib/docker
vagrant@host1:~$# none because we have no images or containers 
vagrant@host1:~$ mount  |grep btrfs
/dev/sdb on /var/lib/docker type btrfs (rw)
vagrant@host1:~$ ps ax  |grep [d]ocker
10041 ?        Sl     0:00 /usr/bin/docker.io -d -s btrfs
vagrant@host1:~$ docker pull busybox
Pulling repository busybox
a9eb17255234: Download complete 
d200959a3e91: Download complete 
fd5373b3d938: Download complete 
37fca75d01ff: Download complete 
511136ea3c5a: Download complete 
42eed7f1bf2a: Download complete 
f06b02872d52: Download complete 
120e218dd395: Download complete 
c120b7cab0b0: Download complete 
1f5049b3536e: Download complete 
vagrant@host1:~$ docker images
REPOSITORY          TAG                   IMAGE ID            CREATED             VIRTUAL SIZE
busybox             buildroot-2013.08.1   d200959a3e91        4 days ago          2.489 MB
busybox             ubuntu-14.04          37fca75d01ff        4 days ago          5.609 MB
busybox             ubuntu-12.04          fd5373b3d938        4 days ago          5.455 MB
busybox             buildroot-2014.02     a9eb17255234        4 days ago          2.433 MB
busybox             latest                a9eb17255234        4 days ago          2.433 MB
vagrant@host1:~$# now we should have some subvolumes
vagrant@host1:~$ sudo btrfs subvolume list /var/lib/docker
ID 258 gen 12 top level 5 path btrfs/subvolumes/511136ea3c5a64f264b78b5433614aec563103b4d4702f3ba7d4d2698e22c158
ID 259 gen 17 top level 5 path btrfs/subvolumes/42eed7f1bf2ac3f1610c5e616d2ab1ee9c7290234240388d6297bc0f32c34229
ID 260 gen 14 top level 5 path btrfs/subvolumes/f06b02872d5253f5123284edcf49749b352400a1c5880b5ebf2864f5afddeb22
ID 261 gen 14 top level 5 path btrfs/subvolumes/37fca75d01ffc49df7b99aacdbcd4a0ebae39de299787b8f77bb5b6698414308
ID 262 gen 16 top level 5 path btrfs/subvolumes/120e218dd395ec314e7b6249f39d2853911b3d6def6ea164ae05722649f34b16
ID 263 gen 16 top level 5 path btrfs/subvolumes/a9eb172552348a9a49180694790b33a1097f546456d041b6e82e4d7716ddb721
ID 264 gen 19 top level 5 path btrfs/subvolumes/1f5049b3536eb73e7a660a672976ae9c19e8460bf57c2528f9c1e4b2c4bf309f
ID 265 gen 19 top level 5 path btrfs/subvolumes/c120b7cab0b0509fd4de20a57d0f5c17106f3451200dfbfd8c6ab1ccb9391938
ID 266 gen 19 top level 5 path btrfs/subvolumes/d200959a3e91d88e6da9a0ce458e3cdefd3a8a19f8f5e6a1e7f10f268aea5594
ID 267 gen 19 top level 5 path btrfs/subvolumes/fd5373b3d93820744a327e609ee86166e5984d7377987f0fde78daeaa345705d
</code>
</pre>

## Conclusion

I haven't explored much more in terms of using btrfs with docker yet, but this is a good start.


