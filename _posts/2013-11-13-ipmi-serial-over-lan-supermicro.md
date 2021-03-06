---
layout: post
title: MetalOps - IPMI serial-over-lan and Supermicro systems
categories: 
header_image: https://raw.github.com/ccollicutt/ccollicutt.github.com/master/img/ipmi_supermicro.png
---

# {{ page.title }}

IPMI access is important for people who admister bare metal...which is something that I do from time to time. I administer a small openenstack cluster of eight nodes, based on Dell C6220 chassis, and also a ten node openstack swift cluster based on Supermicro hardware. 

Usually people get a console, ie. bios access, via some Java applet. I find that really difficult to use because often the applet only runs on one OS, so it means installing a VM with that OS, firing up a browswer and downloading the applet.

IPMI serial-over-lan is much easier and works with both the Dell and the Supermicro hardware.

## Getting to the bios with SOL

Usually I open two terminals:

# Terminal to use access the SOL console with
# Terminal to control the power of the server

NOTE: Probably a good idea to change the ADMIN password. :)

First, establish an SOL connection. Below I'm assuming the server has already been setup with it's IPMI static IP of 10.0.0.10.

<pre>
<code>terminal_1$ ipmitool -I lanplus -H 10.0.0.10 -U ADMIN -P ADMIN sol activate
</code>
</pre>

Next, power down the server.

<pre>
<code>terminal_2$ ipmitool -I lanplus -H 10.0.0.10 -U ADMIN -P ADMIN chassis power off
</code>
</pre>

Then set it to boot into bios. I find this easier then trying to hit a key on startup.

<pre>
<code>terminal_2$ ipmitool -I lanplus -H 10.0.0.10-U ADMIN -P ADMIN chassis bootdev bios
</code>
</pre>

And finally start the server again.

<pre>
<code>terminal_2$ ipmitool -I lanplus -H 10.0.0.10 -U ADMIN -P ADMIN chassis power on
</code>
</pre>

In a few seconds, or a minute perhaps, takes a while for these systems to boot up, you should see the server start to come up in terminal_1, and eventually it will drop you into the text based bios, an example of which you can see in the picture at the very top of this blog post.

To exit, hit enter and then the "~" (tilde) key twice, and finally a ".". If you just enter one tilde then when you exit you will exit right out of your ssh session to the server that has access to the management network on which the IPMI interfaces are places, assuming that is, you have a secure management network.

Note that IPMI interfaces are notoriously insecure, and using them definitely requires some careful thought and resources, if available, to secure them.
