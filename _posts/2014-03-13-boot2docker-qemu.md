---
layout: post
title: boot2docker and libvirt
categories: 
---

# {{ page.title }}

[Docker](https://www.docker.io) is so hot right now. Well, containers in general are. LXC just hit version 1.0 and the developers have declared it production ready. 

Here is part of the LXC 1.0 release announcement:

> LXC 1.0 is the first production ready release of LXC and it comes with a commitment from upstream to maintain it until at least Ubuntu 14.04 LTS reaches end of life in April 2019. That's slightly over 5 years of support!

So what is docker? From the website:

> Docker is an open-source project to easily create lightweight, portable, self-sufficient containers from any application. The same container that a developer builds and tests on a laptop can run at scale, in production, on VMs, bare metal, OpenStack clusters, public clouds and more.

## boot2docker

The idea behind [boot2docker](https://github.com/boot2docker/boot2docker) is to be able to use docker quickly, mostly on OSX. But OSX doesn't support containers (yet, maybe someday), so running docker natively isn't possible. 

To use docker on OSX it has to be done inside a Linux virtual machine running in a hypervisor (like Virtualbox or VMWare Fusion) on top of OSX. This is what boot2docker does--provides a small Linux vm with docker installed, and helps get it all configured and provides a command line interface.

From the github repo for boot2docker:

> boot2docker is a lightweight Linux distribution based on Tiny Core Linux made specifically to run Docker containers. It runs completely from RAM, weighs ~24MB and boots in ~5s (YMMV). 

boot2docker comes with helpful commands and setup to get this running easily and quickly on OSX, but I'm not going to use it on OSX...I'm going to use it with libvirt and KVM.

## Use boot2docker with libvirt

First I downloaded the boot2docker iso from github.

<pre>
<code>root# wget \
https://github.com/boot2docker/boot2docker/releases/download/v0.7.0/boot2docker.iso
</code>
</pre>

Then I created a qemu disk image from that iso.

<pre>
<code>root# qemu-img convert -O qcow2 boot2docker.iso /var/lib/libvirt/images/boot2docker.img
root# file /var/lib/libvirt/images/boot2docker.img: QEMU QCOW Image (v2), 25165824 bytes
</code>
</pre>

## boot2docker boot script

Now that I have a base backing file made from the boot2docker ISO file, I can boot virtual machines off it.

I wrote a script that I am still in the process of refining (ie. this is still pretty ugly) but using it I can start several boot2docker based virtual machines from libvirt.

I've also added a second drive for each vm and in the script the drive image gets partitioned and ext4 formatted with a label that boot2docker recognizes and mounts automatically.

I'm using sfdisk to partition a second file, and the partitioning may not be setup properly. I haven't done enough testing, but it's working so far. :)

<pre>
<code>root# cat boot2docker.sh 
#!/bin/bash

vmtype=boot2docker
num_vms=4
backing_image=boot2docker.img

for ((i=1; i<=num_vms; i++)); do

  virsh destroy ${vmtype}$i > /dev/null
  virsh undefine ${vmtype}$i > /dev/null
  rm -f ./${vmtype}0${i}.xml > /dev/null
  rm -f /var/lib/libvirt/images/${vmtype}$i.img > /dev/null
  rm -f /var/lib/libvirt/images/${vmtype}$i-persist.img > /dev/null

#
# Setup partitions for image
#

cat <<-SFDISKOUT > /var/tmp/sfdisk.out.${vmtype}${i}
# partition table of boot2docker1-persist.img
unit: sectors

${vmtype}${i}-persist.img1 : start=     2048, size= 10483712, Id=83
${vmtype}${i}-persist.img2 : start=        0, size=        0, Id= 0
${vmtype}${i}-persist.img3 : start=        0, size=        0, Id= 0
${vmtype}${i}-persist.img4 : start=        0, size=        0, Id= 0
SFDISKOUT

  #
  # Create images
  # 

  pushd /var/lib/libvirt/images > /dev/null
    qemu-img create -f qcow2 -b ${backing_image} ${vmtype}${i}.img > /dev/null
    qemu-img create -f raw ${vmtype}${i}-persist.img 5G
    sfdisk --force ${vmtype}${i}-persist.img < /var/tmp/sfdisk.out.${vmtype}${i}
    losetup --offset 1048576 /dev/loop0 ${vmtype}${i}-persist.img
    mkfs.ext4 -F -L boot2docker-data /dev/loop0
    losetup -d /dev/loop0
  popd > /dev/null

  rm -f /var/tmp/sfdisk.out.${vmtype}${i}

  chown libvirt-qemu:kvm /var/lib/libvirt/images/*.img

  vm_uuid=`uuid`

  #
  # Build the libvirt xml file
  #

 cat <<-LIBVIRTXML > ${vmtype}${i}.xml
<domain type='kvm'>
    <uuid>${uuid}</uuid>
    <name>${vmtype}${i}</name>
    <memory>4194304</memory>
    <os>
            <type>hvm</type>
            <boot dev="hd" />
    </os>
    <features>
        <acpi/>
    </features>
    <vcpu>1</vcpu>
    <devices>
        <disk type='file' device='disk'>
            <driver type='qcow2' cache='none'/>
            <source file='/var/lib/libvirt/images/${vmtype}${i}.img'/>
            <target dev='vda' bus='virtio'/>
        </disk>
        <disk type='file' device='disk'>
      	    <driver type='raw' cache='none'/>
            <source file='/var/lib/libvirt/images/${vmtype}${i}-persist.img'/>
            <target dev='vdb' bus='virtio'/>
        </disk>

  <interface type='network'>
     <source network='default'/>
            <model type='virtio'/>
            <mac address='fa:16:3e:18:89:0${i}'/>
  </interface>


    </devices>
</domain>
LIBVIRTXML

  #
  # Define and start the vm
  #

  virsh define ${vmtype}${i}.xml > /dev/null
  if virsh start ${vmtype}${i} > /dev/null; then
	echo "${vmtype}${i} started"
  fi
  sleep 1

done

virsh list --all
exit 0
</code>
</pre>

## Create some boot2docker virtual machines

If I run that script, which is a little noisy, I end up with four virtual machines running the boot2docker OS (which is based on Tiny Linux).

The script ends off by listing all the running vms on the box.

<pre>
<code>root# ./boot2docker.sh 
SNIP!
boot2docker4 started
 Id Name                 State
----------------------------------
127 boot2docker1         running
128 boot2docker2         running
129 boot2docker3         running
130 boot2docker4         running
</code>
</pre>

The vms are getting IPs from dnsmasq which is configured by default by libvirt.

<pre>
<code>root# cat /var/lib/libvirt/dnsmasq/default.leases 
1394830266 fa:16:3e:18:89:04 192.168.122.246 * 01:fa:16:3e:18:89:04
1394830262 fa:16:3e:18:89:03 192.168.122.245 * 01:fa:16:3e:18:89:03
1394830263 fa:16:3e:18:89:02 192.168.122.244 * 01:fa:16:3e:18:89:02
1394830255 fa:16:3e:18:89:01 192.168.122.243 * 01:fa:16:3e:18:89:01
</code>
</pre>

I set the mac adresses in the script to be fa:16:3e:18:89:0X.

Knowing the IPs the vms received from libvirt/dnsmasq, I can ssh into them. (The default user/pass is docker/tcuser.)

<pre>
<code>root# ssh docker@192.168.122.243
Warning: Permanently added '192.168.122.243' (ECDSA) to the list of known hosts.
docker@192.168.122.243's password: 
                        ##        .
                  ## ## ##       ==
               ## ## ## ##      ===
           /""""""""""""""""\___/ ===
      ~~~ {~~ ~~~~ ~~~ ~~~~ ~~ ~ /  ===- ~~~
           \______ o          __/
             \    \        __/
              \____\______/
 _                 _   ____     _            _
| |__   ___   ___ | |_|___ \ __| | ___   ___| | _____ _ __
| '_ \ / _ \ / _ \| __| __) / _` |/ _ \ / __| |/ / _ \ '__|
| |_) | (_) | (_) | |_ / __/ (_| | (_) | (__|   <  __/ |
|_.__/ \___/ \___/ \__|_____\__,_|\___/ \___|_|\_\___|_|
boot2docker: 0.7.0
</code>
</pre>

And we can see that the second device, which is /dev/vdb, has indeed been mounted.

<pre>
<code>docker@boot2docker:~$ df -h
Filesystem                Size      Used Available Use% Mounted on
rootfs                    3.5G    223.4M      3.3G   6% /
tmpfs                     1.9G         0      1.9G   0% /dev/shm
/dev/vdb1                 4.8G     25.3M      4.5G   1% /mnt/vdb1
cgroup                    1.9G         0      1.9G   0% /sys/fs/cgroup
</code>
</pre>

## Use docker

We can run docker version to see if it works.

<pre>
<code>docker@boot2docker:~$ docker version
Client version: 0.9.0
Go version (client): go1.2.1
Git commit (client): 2b3fdf2
Server version: 0.9.0
Git commit (server): 2b3fdf2
Go version (server): go1.2.1
Last stable version: 0.9.0
</code>
</pre>

And also run a docker command. The first time we run a container type it'll have to be downloaded.

<pre>
<code>docker@boot2docker:~$ docker run ubuntu /bin/echo hello world
Unable to find image 'ubuntu' locally
Pulling repository ubuntu
9f676bd305a4: Download complete 
9cd978db300e: Download complete 
eb601b8965b8: Download complete 
5ac751e8d623: Download complete 
9cc9ea5ea540: Download complete 
511136ea3c5a: Download complete 
6170bb7b0ad1: Download complete 
1c7f181e78b9: Download complete 
f323cf34fd77: Download complete 
321f7f4200f4: Download complete 
7a4f87241845: Download complete 
hello world
</code>
</pre>

With boot2docker we can have many small vms that quickly boot and are ready to run docker right away. Not sure how practical this is, but it's interesting none-the-less.

Now I need to fix up the script a bit, and also figure out how to setup ssh keys so that I don't have to enter a password to login.
