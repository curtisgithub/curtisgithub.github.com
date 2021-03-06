---
layout: post
title: More over committing with kvm
categories:
---

# {{ page.title }}

Previously I wrote about [overcommiting with kvm](http://serverascode.com/2013/02/20/overcommitting-with-kvm.html).

In this post I'm still doing the exact same thing, but now I'm keep track of load and iops.

## Basic environment

What we have is:

- a single Dell C6220 node
- 32 threads
- 128GB of memory
- two Intel 520 SSDs in a stripe
- ubuntu precise64 cloud image
- qcow2 image snapshots
- open vSsitch
- dnsmasq providing ip addresses
- "ansible":ansible.cc for running stress tests

What we're going to do is boot 300 2GB instances based off the same image.

## Get setup

First thing we do is reboot, just to start fresh. It doesn't really matter, but I like to start with no swap being used, just to show exactly what happens when we boot hundreds of vms.

First set ksm to do more scanning faster.

<pre>
<code>root# echo "20000" > /sys/kernel/mm/ksm/pages_to_scan
root# echo "20" > /sys/kernel/mm/ksm/sleep_millisecs
</code>
</pre>

Next--re-setup vswitch, as I have a kind of [rigged-up configuration](http://serverascode.com/2013/02/21/openvswitch.html) going, which needs some care after a reboot, so obviously this is just for testing, not production. :)

<pre>
<code>$ sudo ifconfig br-int
$ sudo ifconfig br-int up
$ sudo ifconfig br-int 192.168.100.10 netmask 255.255.255.0
</code>
</pre>

That ip is what dnsmasq is set to listen on.

I was running into an error where some taps existed already, and the boot script would start failing. I'm not sure why, and I just ended up deleting all the ports in the switch for that particular bridge, br-int.

<pre>
<code>#
# Show/count all the ports
# 

$ sudo ovs-vsctl list-ports br-int | wc -l
300

#
# Delete some ports
#

$ sudo for i in $(seq 1 300); do ovs-vsctl del-port br-int tap$i; done
# This takes a while...
</code>
</pre>

Now restart to make sure dnsmasq is listening on 102.168.100.10.

<pre>
<code>$ sudo service dnsmasq stop  
 * Stopping DNS forwarder and DHCP server dnsmasq
 [ OK ] 

#
# Now with the right br-int config
#

$ sudo dnsmasq start
</code>
</pre>

Next--time to start hundreds of vms!

## Start virtual machines

Ok, now that we have networking (hopefully) all set up, we're going to boot 300 vms, 10 seconds apart. 

Thankfully these are linux vms so they don't really cause a boot storm, unlike windows 7. 

If I booted 30 (note: 30, not 300) windows 7 vms the server's load would get so high that the system would grind to a halt. I know because I've tried it. Even though this is an ubuntu cloud image, which hopefully is specifically setup to use less iops, and knowing that is perhaps an unfair advantage, I still have no problem saying that windows images use more resources than linux images.

Start instances!

<pre>
<code>root# ./kvm_ubuntu_openvswitch.sh 
</code>
</pre>

After that completes there are ~300 vms running.

<pre>
<code>$ ps ax | grep "kvm -drive" | wc -l
301
</code>
</pre>

We can run tests on those vms.

_NOTE: I haven't taken the time to tell dnsmasq to send dhcp information for more than a /24, so we only have 240 vms with an ip address._

<pre>
<code>$ wc -l /var/lib/misc/dnsmasq.leases 
240 /var/lib/misc/dnsmasq.leases
</code>
</pre>

So when I run the stress test with ansible, we can only run it across 240 vms, even though 300 are running, it's just that 60 of them don't have ips.

## Stress test

In my previous post a commenter suggested running _stress_, so that's what I'm doing.

Using ansible, I'll run this stress command:

<pre>
<code>shell stress --cpu 1 --io 1 --vm 1 --vm-bytes 1024M --timeout 10s
</code>
</pre>

across 20 vms at a time, running over all the vms that are reported by an inventory script that looks at the ips in the dnsmasq.leases file.

Here's an example of running ansible's ping module across all those hosts.

<pre>
<code>$ ansible all -c ssh -i ./inventory.py -m ping -u ubuntu
 SNIP!
 192.168.100.98 | success >> {
    "changed": false, 
    "ping": "pong"
}

192.168.100.99 | success >> {
    "changed": false, 
    "ping": "pong"
}

192.168.100.97 | success >> {
    "changed": false, 
    "ping": "pong"
}
</code>
</pre>

Note that the ips aren't in order, so .97 is the last host in this run. Suffice it to say that all 240 hosts "ponged" back. :)

Here's the simple ansible playbook I'll be running. These vms don't have access to the internet, and don't have stress installed, so I'm just copying over the package and installing it "manually", and then running the stress command.

<pre>
<code>$ cat load.yml 
---
- hosts: all
  user: ubuntu
  sudo: yes
  tasks:
  - name: check if stress is already installed
    action: shell which stress
    register: stress_installed
    ignore_errors: True
  - name: copy stress deb to server
    action: copy src=files/stress_1.0.1-1build1_amd64.deb \
    dest=/tmp/stress_1.0.1-1build1_amd64.deb
    only_if: ${stress_installed.rc} > 0
  - name: install stress
    action: shell dpkg -i /tmp/stress_1.0.1-1build1_amd64.deb
    only_if: ${stress_installed.rc} > 0
  - name: run stress
    action: shell stress --cpu 1 --io 1 --vm 1 --vm-bytes 1024M --timeout 10s
</code>
</pre>

Let's run it across 20 vms at a time and see what happens.

<pre>
<code>$ ansible-playbook -u ubuntu -c ssh -i ./inventory.py -f 20 ./load.yml

PLAY [all] ********************* 

GATHERING FACTS ********************* 
ok: [192.168.100.110]
ok: [192.168.100.113]
ok: [192.168.100.117]
SNIP!    
192.168.100.97                 : ok=3    changed=2    unreachable=0    failed=0    
192.168.100.98                 : ok=3    changed=2    unreachable=0    failed=0    
192.168.100.99                 : ok=3    changed=2    unreachable=0    failed=0  

# Done!
</code>
</pre>

Ansible can be fun. :)

## Graphs

Below are a rather poor set of graphs. Forgive me as I'm a newbie with gnuplot.

As soon as the load starts going up, that is when the test starts, and as soon as it's on its way down, that's when it ends. :)

First run:

!https://raw.github.com/ccollicutt/ccollicutt.github.com/master/img/kvm_overcommitting_load_1.png! 

Second run:

!https://raw.github.com/ccollicutt/ccollicutt.github.com/master/img/kvm_overcommitting_load_2.png! 

Last load run:

!https://raw.github.com/ccollicutt/ccollicutt.github.com/master/img/kvm_overcommitting_load_3.png! 

So with three runs we see a load of about 30, where a load of 32 would be Ok with me, given we have 32 threads in this server.

Also, let's watch some iops.

I gathered iops data using _iostat_.

!https://raw.github.com/ccollicutt/ccollicutt.github.com/master/img/kvm_overcommitting_io_1.png! 

Ooof, that's a misleading graph, isn't it? An increase of one iop is absolutely nothing. A rounding error perhaps.

So--the iops don't change during the test run, I guess because stress isn't running any io test, even though we are running with _--io 1_. That said, I'm not sure what an io setting of 1 with stress does, something to look into. Perhaps running some tests with [fio](http://freecode.com/projects/fio) is something to do in the future.

But that graph sure _looks_ like something's happening, even though the iops only increase by one. I probably shouldn't include that graph here, but part of what I'm doing is learning about how to display the results of a performance test.

## Conclusion

First off, I'm not a scientist, didn't take statistics, etc. So I'm not sure what kind of conclusions can be made here. All I can say for sure is that fours runs of the stress command across 20 virtual machines in parallel, over a total of 240 vms all running on the same KVM-based host, seems to bring load up to somewhere around 30 to 35, which is acceptable to me.

Mostly this has generated more questions--such as what exactly is stress doing? How do we know when the vms are too unresponsive? What kind of overcommitting numbers do we want? What would happen if we used fio instead of _--io 1_ with stress? Do red lines in a graph make things seem worse? :)

As usual, if anyone has any suggestions, questions, or critiques let me know in the comments!
