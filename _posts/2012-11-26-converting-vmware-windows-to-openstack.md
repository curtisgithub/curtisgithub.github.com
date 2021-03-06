---
layout: post
title: Converting VMWare Windows images to OpenStack with virt-v2v 
categories:
---

# {{ page.title }}

As I've mentioned in previous posts, I'm currently working on a project that uses Apache VCL to provide university students with the ability to remotely login to a virtual machine and use specialized software to complete their classwork--a virtual computer lab if you will.

Because we are moving off our current backend to OpenStack, we need to convert Windows images from VMWare to something that will work in OpenStack/KVM.

## Note about Windows...

Let me note that I am not a "windows guy"--I haven't really used windows in the last 10 years. Because I have usually been employed as a Unix/Linux admin, I've always either ran OpenBSD, Linux, or OSX on my desktop, and literally never had to work with Windows. Until now of course. ;)

## Moving the backend to OpenStack

Currently we are in the process of moving the backend of our VCL system from a VMWare ESXi based cluster to OpenStack (Essex to be precise.) There are a lot of reasons for this move, and I won't get into them here.

Unfortunately you can't just take an ESXi image and drop it into OpenStack and have it work. Certainly we can actually import the image into glance, as vmdk images are supported, but the OS on the image will not have the virtio drivers for disk and network installed, drivers that OpenStack uses by default.

Neither is it as easy as just installing the drivers into the OS image while it's running in ESXi. While I believe it's possible to do this in Windows Server 2008, I don't believe you can install drivers into Windows 7 without having the actual "hardware" accessible to the image (please correct me if I'm wrong--I'd love to hear that I am), not without some registry and other hacks outside of my purview.

This means we needed to find a way to convert the images from ones that work in VMWare ESXi, to ones that will work in OpenStack + KVM, preferably a way that doesn't require me to learn anything about the Windows registry. ;)

## virt-v2v to the rescue

Fortunately RedHat provides a system called virt-v2v. This allows the cross-conversion of images for several different types of hypervisors.

The main things I found out about RedHat and virt-v2v are:

# Some of the packages are not available in CentOS, or at least I couldn't find them, so you need a RedHat license. If you are converting images from VMWare ESXi to OpenStack, then you probably have enough money in the project to buy a RedHat license. ;)
# It doesn't work in a virtual machine--it expects hardware virtualization, so you need a hardware server. You don't need much, just enough to support hardware virtualization with KVM.
# Currently, and this might change, you have to download the image from the ESXi server each time you try a conversion, so if it takes a long time to download, then the conversion will also take a long time to finish.

I'm sure most of the above could be changed with a bit of Perl hacking, but as far as I can tell, without changing any of the RedHat code you do need to meet those requirements.

Note--RedHat has pretty good [documentation](https://access.redhat.com/knowledge/docs/en-US/Red_Hat_Enterprise_Virtualization_for_Desktops/2.2/html/Administration_Guide/virt-v2v-scripts.html) on the subject, a few quick google searches will tell you as much as I know. :)

## Installing virt-v2v

First, as I've mentioned, you need a hardware server with RedHat Enterprise 6.x on it.

<pre>
<code>[root@localhost ~]# cat /etc/redhat-release 
Red Hat Enterprise Linux Server release 6.3 (Santiago)
</code>
</pre>

It needs to be registered with the RedHat Network and have the "RHEL Server Supplementary" and "RHEL V2VWIN" channels enabled, as per the below image.

![](https://raw.github.com/ccollicutt/ccollicutt.github.com/master/img/rhn_entitlements_virt-v2v.png)

<pre>
<code>[root@localhost ~]# yum repolist
Loaded plugins: product-id, rhnplugin, subscription-manager
Updating certificate-based repositories.
Unable to read consumer identity
repo id       repo name     status
rhel-x86_64-server-6 Red Hat Enterprise Linux Server (v. 6 for 64-bit x86_64) 8,712
rhel-x86_64-server-supplementary-6 RHEL Server Supplementary (v. 6 64-bit x86_64) 311
rhel-x86_64-server-v2vwin-6 RHEL V2VWIN (v. 6 for 64-bit x86_64) 2
repolist: 9,025
</code>
</pre>

Once those are configured, install the following packages.

<pre>
<code>[root@localhost ~]# yum install virt-v2v virtio-win libguestfs-winsupport libvirt
</code>
</pre>

And then reboot.

Once the server has rebooted, start libvirtd if it isn't already.

<pre>
<code>[root@localhost ~]# service libvirtd start
</code>
</pre>

Next we need to add a storage pool to libvirt. In this example, just because of the way RHEL defaulted my temporary install, we have a huge /home/images directory in which to put the converted images.

<pre>
<code>[root@localhost ~]# df -h /home
Filesystem            Size  Used Avail Use% Mounted on
/dev/mapper/VolGroup-lv_home
                      5.4T  235G  4.9T   5% /home
[root@localhost ~]# cat pool.xml 
  <pool type="dir">
        <name>virtimages</name>
        <target>
          <path>/home/images</path>
        </target>
      </pool>
</code>
</pre>

Using that pool.xml file, we'll configure the pool.

<pre>
<code>[root@localhost ~]# virsh pool-create --file pool.xml
[root@localhost ~]# virsh pool-list
Name                 State      Autostart 
-----------------------------------------
virtimages           active     no        

</code>
</pre>

Finally, we configure a .netrc file, obviously entering the proper ESXi host, login, and password.

<pre>
<code>[root@localhost ~]# cat .netrc
machine <esxi host> login <login> password <password>
</code>
</pre>

Now we can run virt-v2v!

<pre>
<code>[root@localhost images]# virt-v2v -ic esx://<esxi host>/?no_verify=1 -o libvirt
 -os virtimages <exsi vm name>
<esxi vm name>: 100% [========================]D 0h51m16s
virt-v2v: WARNING: No mapping found for bridge interface public in config file. 
The converted guest may not start until its network interface is updated.
virt-v2v: <esxi vm name> configured with virtio drivers.
</code>
</pre>

I don't worry about the "No mapping" message.

## Bring that image into OpenStack

Now that we have an image in the libvirt pool we configured, it's time to:

# Convert the raw/vmdk image to qcow2 using "qemu-img".
# Import the converted qcow2 image into glance.
# Boot the image once and manually login to finish off virt-v2v's "firstboot" scripts. This means you need an admin login to the image.
# "Set" the hardware, make any changes to the instance such as updating and the like.
# Shut down the instance.
# Create a new OpenStack image from that instance using _nova image-create_.
# Then delete the original image, and instance booted from it, if you want. It's not much use except for archival purposes.
# Boot a new instance from the new image, the one created with _nova image-create_, just to make sure it all works out Ok.

At this point you should, hopefully, have a working Windows image, one that was converted from VMWare ESXi!

