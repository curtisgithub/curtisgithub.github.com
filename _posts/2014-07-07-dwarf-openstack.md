---
layout: post
title: Fake OpenStack with Dwarf
categories:

---
 
# {{ page.title }}

[Dwarf](https://github.com/juergh/dwarf) is this really cool little project by [Juerg Haefliger](https://github.com/juergh) that provides a subset of the OpenStack APIs to use libvirt on a single host. For some context, here's the [original email](http://www.gossamer-threads.com/lists/openstack/dev/36420?search_string=dwarf;#36420) that was sent to the OpenStack list. What it does is allow you to use manage a single libvirt host as though it were OpenStack, ie. use nova, glance, and keystone commands to manage libvirt virtual machines.

## Why?

For some reason I find faking APIs really interesting. I guess a better word than faking would be "compatability" but really what is going on is APIs are being faked. For example, OpenStack has always, as far as I know, provided some Amazon Web Services (AWS) compatibility. OpenStack Swift also can provide Amazon S3 API compatibility. Another example is [Cloudscaling providing a GCE compatabile API for OpenStack](http://www.cloudscaling.com/blog/openstack/gce-api-available-now-on-openstack-stackforge/). 

I think fake APIs also suggests that a certain application or service is becoming popular, and so having a little fake OpenStack subset API using Dwarf is a compliment to OpenStack. Also it can help in terms of understanding how OpenStack works. Do you wonder what the keystone catalog does? Well, now you can mess around with it in Dwarf and find out.

## Caveats

As Haefliger says in the README for Dwarf:

- No authentication!
- Just for one host
- A subset of OpenStack commands
- Serialized and blocking

## Install Dwarf

First install the Dwarf PPA. I'm running this on Ubuntu Precise 12.04, which is itself a virtual machine, and thus we'll be doing nested virtualization, which may need to be turned on in some hosts.

<pre>
<code>ubuntu@dwarf:~$ sudo apt-add-repository ppa:juergh/dwarf 
</code>
</pre>

Then install Dwarf.

<pre>
<code>ubuntu@dwarf:~$ sudo apt-get install dwarf
</code>
</pre>

Now we have a dwarf command.

<pre>
<code>ubuntu@dwarf:~$ which dwarf
/usr/bin/dwarf
</code>
</pre>

We can start dwarf with service.

<pre>
<code>root@dwarf:~# service dwarf start
</code>
</pre>

That starts a few processes listening, openstack identity, openstack compute, and openstack images, ie. keystone, nova, and glance respectively.

<pre>
<code>root@dwarf:/etc# netstat -ant
Active Internet connections (servers and established)
Proto Recv-Q Send-Q Local Address           Foreign Address         State      
tcp        0      0 0.0.0.0:22              0.0.0.0:*               LISTEN     
tcp        0      0 127.0.0.1:35357         0.0.0.0:*               LISTEN     
tcp        0      0 127.0.0.1:8774          0.0.0.0:*               LISTEN     
tcp        0      0 127.0.0.1:9292          0.0.0.0:*               LISTEN     
tcp        0      0 192.168.122.77:22       192.168.122.1:38835     ESTABLISHED
tcp6       0      0 :::22                   :::*                    LISTEN 
</code>
</pre>

Need python-novaclient, python-keystoneclient too. I'm going to use pip to get more recent versions of these commands. I guess glance comes from glance-client? Weird.

<pre>
<code>root@dwarf:~# apt-get install python-pip
root@dwarf:~# pip install python-novaclient
root@dwarf:~# nova --version
2.17.0
root@dwarf:~# glance --version
0.12.0
root@dwarf:~# keystone --version
0.9.0
</code>
</pre>

## Use Dwarf

Create a default openstack rc file and source it. Again note there is no real authentication.

<pre>
<code>root@dwarf:~# cat dwarfrc 
export OS_AUTH_URL=http://127.0.0.1:35357/v2.0/
export OS_COMPUTE_API_VERSION=1.1
export OS_REGION_NAME=dwarf-region
export OS_TENANT_NAME=dwarf-tenant
export OS_USERNAME=dwarf-user
export OS_PASSWORD=dwarf-password
root@dwarf:~# . dwarfrc
</code>
</pre>

And finally we can run some nova commands.

<pre>
<code>root@dwarf:~# nova list
+----+------+--------+----------+
| ID | Name | Status | Networks |
+----+------+--------+----------+
+----+------+--------+----------+
root@dwarf:~# nova flavor-list
+-----+-----------------+-----------+------+-----------+------+-------+-------------+
|  ID |       Name      | Memory_MB | Disk | Ephemeral | Swap | VCPUs | RXTX_Factor |
+-----+-----------------+-----------+------+-----------+------+-------+-------------+
| 100 | standard.xsmall | 512       | 10   | N/A       |      | 1     |             |
| 101 | standard.small  | 768       | 30   | N/A       |      | 1     |             |
| 102 | standard.medium | 1024      | 30   | N/A       |      | 1     |             |
+-----+-----------------+-----------+------+-----------+------+-------+-------------+
root@dwarf:~# keystone catalog
WARNING:keystoneclient.httpclient:Failed to retrieve management_url from token
Service: image
+-------------+----------------------------+
|   Property  |           Value            |
+-------------+----------------------------+
|  publicURL  | http://127.0.0.1:9292/v1.0 |
|    region   |        dwarf-region        |
|   tenantId  |            1000            |
|  versionId  |            1.0             |
| versionInfo | http://127.0.0.1:9292/v1.0 |
| versionList |   http://127.0.0.1:9292    |
+-------------+----------------------------+
Service: compute
+-------------+---------------------------------+
|   Property  |              Value              |
+-------------+---------------------------------+
|  publicURL  | http://127.0.0.1:8774/v1.1/1000 |
|    region   |           dwarf-region          |
|   tenantId  |               1000              |
|  versionId  |               1.1               |
| versionInfo |    http://127.0.0.1:8774/v1.1   |
| versionList |      http://127.0.0.1:8774      |
+-------------+---------------------------------+
</code>
</pre>

For some reason the latest python-glanceclient was broken in my install. Not sure if it's something I did wrong or what, but I ended up using 0.12.0. Note the glance that comes with 12.04 does not have the image-create command.

<pre>
<code>root@dwarf:/tmp# pip install python-glanceclient==0.12.0
root@dwarf:/tmp# which glance
/usr/local/bin/glance
root@dwarf:/tmp# glance --version
0.12.0
</code>
</pre>


<pre>
<code>root@dwarf:~# glance index
No handlers could be found for logger "keystoneclient.httpclient"
ID                                   Name                           Disk Format          Container Format     Size          
------------------------------------ ------------------------------ -------------------- -------------------- --------------
96b8b4cc-bf45-4dc2-add0-c6d0fc96aec4                                                                                        
d0a3d8f7-d336-40d1-b548-fbb5e5e01d8f                                                                                        
ced791fc-bd11-4d91-9eb6-3fe892dd2a6d                                                                                        
40bee026-03a4-4020-88bc-bc0acf9465a6                                                                                        
0a29e5fc-fb0b-487d-b957-f4fd296d71b1                                                                                        
f51efef5-fe73-4957-94c0-bf94038a2685 
</code>
</pre>

Interestingly running glance image-create with no options with dwarf creates empty images. I deleted all those and also added a cirros image.

Download the Cirros image. I'm using Cirros because I'm running Dwarf inside a virtual machine, so have limited resources.

<pre>
<code>root@dwarf:~# wget http://download.cirros-cloud.net/0.3.2/cirros-0.3.2-x86_64-disk.img
</code>
</pre>

Add that image to Dwarf using glance.

<pre>
<code>root@dwarf:~# glance image-create --name "Cirros" --file cirros-0.3.2-x86_64-disk.img 
</code>
</pre>

Now it's in glance.

<pre>
<code>root@dwarf:~# glance image-list
No handlers could be found for logger "keystoneclient.httpclient"
+--------------------------------------+--------+-------------+------------------+----------+--------+
| ID                                   | Name   | Disk Format | Container Format | Size     | Status |
+--------------------------------------+--------+-------------+------------------+----------+--------+
| 56105cdc-00d8-4e69-beae-fbe20abcbe36 | Cirros |             |                  | 13167616 | ACTIVE |
+--------------------------------------+--------+-------------+------------------+----------+--------+
</code>
</pre>

Add a keypair. (Though cirros won't use it.)

<pre>
<code>
root@dwarf:~# nova keypair-add --pub-key ~/.ssh/id_rsa.pub root
root@dwarf:~# nova keypair-list
+------+-------------------------------------------------+
| Name | Fingerprint                                     |
+------+-------------------------------------------------+
| root | 7f:21:e1:9b:ee:3d:84:89:a5:bc:c1:3e:79:20:e5:c0 |
+------+-------------------------------------------------+
</code>
</pre>

If you are nesting, ie. a vm inside a vm, before going further edit the default libvirt network. Change 192.168.122.0/24 to some other network, such as 10.0.0.0/24. 192.168.122.0/24 will likely already be in use and the default network won't start, and neither will libvirt based vms.

<pre>
<code>root@dwarf:~# virsh net-edit default
# Edit to look like this:
<code><network>
  <name>default</name>
  <uuid>7fd26ceb-ed87-7887-198e-d9cbc4759b70</uuid>
  <forward mode='nat'/>
  <bridge name='virbr0' stp='on' delay='0' />
  <ip address='10.0.0.1' netmask='255.255.255.0'>
    <dhcp>
      <range start='10.0.0.2' end='10.0.0.254' />
    </dhcp>
  </ip>
</network>
</code>
</pre>

Start that network.

<pre>
<code>root@dwarf:~# virsh net-start default
setlocale: No such file or directory
Network default started
root@dwarf:~# brctl show
bridge name bridge id   STP enabled interfaces
virbr0    8000.525400a6a92a yes   virbr0-nic
</code>
</pre>

Looks good.

Next: boot a vm. 

<pre>
<code>root@dwarf:~# nova boot --flavor 100 --image 56105cdc-00d8-4e69-beae-fbe20abcbe36 --key_name root test1
ERROR: 'NoneType' object has no attribute 'get'
</code>
</pre>

I see there is an error reported, but the vm does indeed get started up. Something to look into, might have to do with the version of nova client being used. I certainly had some trouble with the glance client.

That vm is now running:

<pre>
<code>root@dwarf:~# virsh list
setlocale: No such file or directory
 Id Name                 State
----------------------------------
  2 dwarf-00000003       running

root@dwarf:~# nova list
+--------------------------------------+-------+--------+------------+-------------+-------------------+
| ID                                   | Name  | Status | Task State | Power State | Networks          |
+--------------------------------------+-------+--------+------------+-------------+-------------------+
| 0e8a17bf-4c99-455d-87d6-4eb8d35af1d7 | test1 | ACTIVE | N/A        | N/A         | private=10.0.0.28 |
+--------------------------------------+-------+--------+------------+-------------+-------------------+
root@dwarf:~# cat /var/lib/libvirt/dnsmasq/default.leases 
1404773235 52:54:00:2c:94:14 10.0.0.28 * 01:52:54:00:2c:94:14
root@dwarf:~# ping -c 1 -w 1 10.0.0.28
PING 10.0.0.28 (10.0.0.28) 56(84) bytes of data.
64 bytes from 10.0.0.28: icmp_req=1 ttl=64 time=0.694 ms

--- 10.0.0.28 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 0.694/0.694/0.694/0.000 ms
</code>
</pre>

ssh into the vm...login "cirros", password "cubswin:)"

<pre>
<code>root@dwarf:~# ssh cirros@10.0.0.28
The authenticity of host '10.0.0.28 (10.0.0.28)' can't be established.
RSA key fingerprint is 44:8a:7a:ce:25:d6:f6:aa:2f:98:bb:c3:ec:a2:e8:2a.
Are you sure you want to continue connecting (yes/no)? yes
Warning: Permanently added '10.0.0.28' (RSA) to the list of known hosts.
cirros@10.0.0.28's password: 
$ ifconfig eth0
eth0      Link encap:Ethernet  HWaddr 52:54:00:2C:94:14  
          inet addr:10.0.0.28  Bcast:10.0.0.255  Mask:255.255.255.0
          inet6 addr: fe80::5054:ff:fe2c:9414/64 Scope:Link
          UP BROADCAST RUNNING MULTICAST  MTU:1500  Metric:1
          RX packets:92 errors:0 dropped:0 overruns:0 frame:0
          TX packets:70 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:1000 
          RX bytes:9758 (9.5 KiB)  TX bytes:7502 (7.3 KiB)
$ cat /proc/meminfo | head -1
MemTotal:         503476 kB
</code>
</pre>

In fact Dwarf does at least one nice thing for us in that it'll determine the virtual machines IP address automatically, which libvirt doesn't do.

## Conclusion

Using Dwarf we can boot instances using libvirt and qemu. The [code is out there on github](https://github.com/juergh/dwarf) ready to be hacked on and improved or forked for your own purposes. Once you get your host configured properly and the right nova and glance clients installed it seems to work well.

Thanks [Juerg Haefliger](https://github.com/juergh) for Dwarf. :)
