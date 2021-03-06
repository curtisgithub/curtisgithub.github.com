---
layout: post
title: First look at PackStack
categories:
---

# {{ page.title }}

I am a big fan, and user, of [OpenStack](http://openstack.org). I also really like the [Ansible](http://ansible.cc) configuration management and orchestration system. In fact, I use Ansible to deploy OpenStack, and all kinds of [other things](https://github.com/ccollicutt/ansible_playbooks) as well.

Recently on the Ansible mailing list the lead developer (and now CTO of [Ansibleworks](http://ansibleworks.com)) [suggested that it was time to bring together](https://groups.google.com/forum/?fromgroups=#!topic/ansible-project/eNlPwjIHGGs) everyone who is working, or wants to work with, both Ansible and OpenStack.

One of the suggestions was to follow what [PackStack](https://github.com/stackforge/packstack) has done--it's based on puppet--and port it over to Ansible. (NOTE: I'm not even sure that's the right git repo, things are moving so fast!) 

While I had heard of PackStack before, I had never used it, so I decided it was time I took a look so that I can perhaps help with the OpenStack + Ansible project. Frankly, it was on my list of technology to check out because I always need to have some virtual machines running OpenStack, and it would be nice if there was an easy way to quickly build a multi-host OpenStack install (especially if I would like to contribute code back to the community at some point).

Also--I'm sure Ansible will soon be one of Vagrants supported deployment systems, and when that happens it will be very easy to deploy OpenStack with Vagrant.

So I spent a couple hours creating an [Ansible playbook](https://github.com/ccollicutt/ansible_playbooks/tree/master/packstack) that would simply fire off the PackStack command with a generic answer file. So I haven't ported anything from PackStack to Ansible--I'm simply using Ansible and Vagrant to create an environment for PackStack to do it's work.

## RedHat/CentOS

One big note--PackStack currently only supports RedHat/CentOS. I'm using CentOS 6.

## Host organization

That repository also contains a Vagrant file, and an Ansible hosts file. I have four virtual machines making up a small OpenStack cluster:

<pre>
<code>$ cat ansible_hosts 
[openstack]
apis ansible_ssh_host=192.168.100.130
scheduler ansible_ssh_host=192.168.100.131
compute01 ansible_ssh_host=192.168.100.132
compute02 ansible_ssh_host=192.168.100.133
</code>
</pre>

And all of those IPs and names are reflected in the `PackStack.yml`, `files/packstack.cfg`, and `Vagrantfile` files. I don't know if they are the most descriptive names, but this is how I've currently organized it.

<pre>
<code>$ grep "100\.13" Vagrantfile 
apis_config.vm.network :hostonly, "192.168.100.130" # nic3
scheduler_config.vm.network :hostonly, "192.168.100.131" # nic3
compute01_config.vm.network :hostonly, "192.168.100.132" # nic3
compute02_config.vm.network :hostonly, "192.168.100.133" # nic3
</code>
</pre>

The `apis` host is going to run horizon (not that I need it) and the front-facing APIs, and the scheduler will run the mysql server and nova-scheduler...and some cinder related services as well (which likely need more investigation as I have really only have experience thus far with OpenStack Essex--but at least I can experiment with other versions of OpenStack now, thanks to PackStack). 

For example, here is what nova services are running where:

<pre>
<code>[root@apis packstack(keystone_admin)]$ nova-manage service list
Binary           Host       Zone  Status     State Updated_At
nova-scheduler   scheduler  nova  enabled    :-)   2013-03-13 21:15:11
nova-cert        apis       nova  enabled    :-)   2013-03-13 21:15:15
nova-consoleauth apis       nova  enabled    :-)   2013-03-13 21:15:15
nova-compute     compute01  nova  enabled    :-)   2013-03-13 21:15:13
nova-compute     compute02  nova  enabled    :-)   2013-03-13 21:15:12
</code>
</pre>

Smiley faces are good. :-)

## Deploying PackStack

Deploying this specific example only requires a couple of commands (I'm running OSX, and using Virtualbox).

First, tell Vagrant to build the servers. 

*NOTE:* You could have IP collisions if you have other virtual machines running--the IPs used here are hard-coded.

<pre>
<code>$ vagrant up
[apis] Importing base box 'centos6'...
[apis] The guest additions on this VM do not match the install version of
VirtualBox! This may cause things such as forwarded ports, shared
folders, and more to not work properly. If any of those things fail on
this machine, please update the guest additions and repackage the
box.

Guest Additions Version: 4.1.6
VirtualBox Version: 4.2.6
[apis] Matching MAC address for NAT networking...
[apis] Clearing any previously set forwarded ports...
[apis] Fixed port collision for 22 => 2222. Now on port 2200.
SNIP!
</code>
</pre>

Once those are built you should have four vms running:

<pre>
<code>$ vagrant status
Current VM states:

apis                     running
scheduler                running
compute01                running
compute02                running

This environment represents multiple VMs. The VMs are all listed
above with their current state. For more information about a specific
VM, run `vagrant status NAME`.
</code>
</pre>

Once that's done, you can simply run the `packstack.yml` playbook.

<pre>
<code>$ ansible-playbook -k -u root PackStack.yml 
SSH password: #enter the vagrant password of 'vagrant', just have to do this once

PLAY [openstack] ********************* 

GATHERING FACTS ********************* 
ok: [compute01]
ok: [scheduler]
ok: [compute02]
ok: [apis]

SNIP! #Tons of stuff happens here, mostly done by PackStack

TASK: [run PackStack] ********************* 
skipping: [compute01]
skipping: [compute02]
skipping: [scheduler]
changed: [apis]

PLAY RECAP ********************* 
apis                           : ok=13   changed=11   unreachable=0    failed=0    
compute01                      : ok=7    changed=5    unreachable=0    failed=0    
compute02                      : ok=7    changed=5    unreachable=0    failed=0    
scheduler                      : ok=7    changed=5    unreachable=0    failed=0  
</code>
</pre>

When it completes, there should be a multi-server OpenStack cluster running!

Login to the `apis` server:

<pre>
<code>$ ssh root@192.168.100.130
root@192.168.100.130's password: #vagrant password again
Last login: Wed Mar 13 16:34:03 2013 from 192.168.100.1            
[root@apis ~]# source keystonerc_admin 
[root@apis ~(keystone_admin)]$ nova list
# nothing will appear here, but at least you should get no error messages
</code>
</pre>

Because the Ansible playbook downloaded and installed the Cirros image, @nova image-list@ should give some output:

<pre>
<code>[root@apis ~(keystone_admin)]$ nova image-list
+--------------------------------------+--------+--------+--------+
| ID                                   | Name   | Status | Server |
+--------------------------------------+--------+--------+--------+
| 84eebd30-953f-4ffe-b9f9-afb7099f1e75 | cirros | ACTIVE |        |
+--------------------------------------+--------+--------+--------+
</code>
</pre>

At any rate, it's a good start.

Once Vagrant supports Ansible and VMWare Fusion, it will be crazy easy to create a working OpenStack cluster on my laptop.

## Acknowledgments

Nothing I ever do is novel, and this blog post is no different. I based this work off the following: [Installing a 4 node Fedora 18 OpenStack Folsom cluster with PackStack](https://www.berrange.com/tags/packstack/).

## Issues/Caveats/Questions

- I had at least one problem getting this going...PackStack doesn't seem to want to install puppet on CentOS 6. I had to make sure the Ansible playbook setup the EPEL repository so that the puppet RPMs were available.
- AFAIK, Virtualbox doesn't support [nested virtualization](https://www.virtualbox.org/ticket/4032), so you won't be able to boot any [Inception](http://www.imdb.com/title/tt1375666/) styled OpenStack instances, ie. vms within vms. Though, again AFAIK, VMWare Fusion on OSX does supported nested virtualization, which is why I'm excited about Vagrant supporting Fusion. I'd prefer to use KVM, but I need to run OSX for work.
- Don't think [Cinder](https://wiki.openstack.org/wiki/Cinder) is working.
- It can take a good 20 to 30 minutes for this entire build process to complete, mostly because there are a ton of puppet modules to download.
- I'm not even sure what version of OpenStack this installs...something to look into. Might have to jump to a more recent version of Fedora to get something like OpenStack Grizzly.

<pre>
<code>[root@apis ~(keystone_admin)]$ yum info openstack-nova-common
Loaded plugins: fastestmirror, priorities
Loading mirror speeds from cached hostfile
 * base: www.muug.mb.ca
 * epel: www.muug.mb.ca
 * extras: mirror.its.sfu.ca
 * updates: www.muug.mb.ca
Installed Packages
Name       : openstack-nova-common
Arch       : noarch
Version    : 2012.2.2
Release    : 1.el6
Size       : 115 k
Repo       : installed
From repo  : epel
Summary    : Components common to all OpenStack Nova services
URL        : http://openstack.org/projects/compute/
License    : ASL 2.0
Description: OpenStack Compute (codename Nova) is open source software designed to
           : provision and manage large networks of virtual machines, creating a
           : redundant and scalable cloud computing platform. It gives you the
           : software, control panels, and APIs required to orchestrate a cloud,
           : including running instances, managing networks, and controlling access
           : through users and projects. OpenStack Compute strives to be both
           : hardware and hypervisor agnostic, currently supporting a variety of
           : standard hardware configurations and seven major hypervisors.
           : 
           : This package contains scripts, config and dependencies shared
           : between all the OpenStack nova services.
</code>
</pre>


