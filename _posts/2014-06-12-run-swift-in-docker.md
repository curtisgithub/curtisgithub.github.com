---
layout: post
title: Swift OnlyOne - Run OpenStack Swift in Docker
categories: 
header_image: /img/met/DT4626.jpg
header_permalink: http://www.metmuseum.org/collection/the-collection-online/search/438417
---

# {{ page.title }}

First, let met say that this is not about how to run a cluster of OpenStack Swift servers in Docker, rather it's about running a single container that has a version of [OpenStack Swift all-in-one](http://docs.openstack.org/developer/swift/development_saio.html) deployed, and specifically that version only has one storage device (a docker volume) and is configured to store one replica on that device.

## Why Swift OnlyOne?

I'm calling it Swift OnlyOne because it just has one server, one device, and is configured to do one replica. Swift All-in-one, on the other hand, sets up four servers and devices, though within one virtual machine.

Most people will use Swift in a much larger context as typically Swift is used to store huge amounts (petabytes!) of files. But in this case I just want a small deployment that can provide object storage to several other containers, or perhaps to do development against.

As an example--I think that OpenStack Swift is a perfect partner for Docker, and I don't just mean deploying Swift with Docker, I mean having Docker containers use Swift as a datastore. 

If you have multiple containers on multiple hosts, and some of the containers run an application which needs access to the same files, then you need a way to share those files. Usually this is done with a distributed file system (DFS), but that has all kinds of complexity associated, and, I think, makes Docker hard to use in an idiomatic way. What if, instead of using a DFS you used OpenStack Swift, and thus, instead of relying on a file system your application used object storage? I think you would be better off in the long run.

## Use Swift OnlyOne

First, because *Swift requires the filesystem have xattr*, Docker must be setup with either btrfs or XFS (or some other xattr supporting file system). I have only used it with btrfs. So /var/lib/docker is a btrfs volume and the docker daemon is run with "-s btrfs".

_NOTE: I have Vagrant file and Ansible playbook on [github](https://github.com/ccollicutt/vagrant-docker-btrfs) that will setup a Vagrant-based virtual machine with docker and btrfs configured. So all you would have to do is clone that repo and run "vagrant up" to get docker + btrfs._

<pre>
<code>vagrant@host1:~$ sudo btrfs fi show /var/lib/docker
Label: none  uuid: 732ee044-4b3a-4391-8b53-fd7da224c008
	Total devices 1 FS bytes used 1.99GiB
	devid    1 size 20.00GiB used 4.04GiB path /dev/sdb

Btrfs v3.12
vagrant@host1:~$ ps ax | grep [d]ocker
  997 ?        Sl     6:40 /usr/bin/docker.io -d -s btrfs
</code>
</pre>

There is a [github repository](https://github.com/ccollicutt/docker-swift-onlyone) that has the Dockefile and Swift configuration files used for this example, or you can pull it from the *Docker repository*.

<pre>
<code>vagrant@host1:~$ docker pull serverascode/swift-onlyone
Pulling repository serverascode/swift-onlyone
7e8283467cba: Download complete 
SNIP!
d7279e38d8cd: Download complete 
</code>
</pre>

Now we can run an interactive Swift OnlyOne container.

<pre>
<code>vagrant@host1:/vagrant/dockerfiles/swift-onlyone$ docker run -i -t serverascode/swift-onlyone /bin/bash
root@f2f8ccb82c0e:/# /usr/local/bin/startmain.sh 
Device d0r1z1-127.0.0.1:6010R127.0.0.1:6010/sdb1_"" with 1.0 weight got id 0
Reassigned 128 (100.00%) partitions. Balance is now 0.00.
Device d0r1z1-127.0.0.1:6011R127.0.0.1:6011/sdb1_"" with 1.0 weight got id 0
Reassigned 128 (100.00%) partitions. Balance is now 0.00.
Device d0r1z1-127.0.0.1:6012R127.0.0.1:6012/sdb1_"" with 1.0 weight got id 0
Reassigned 128 (100.00%) partitions. Balance is now 0.00.
Starting to tail /var/log/syslog...(hit ctrl-c if you are starting the container in a bash shell)
^C
</code>
</pre>

Note that I hit "ctrl-c" when it says "Starting to tail..." because when running this container in non-interactive mode, I tail /var/log/syslog to be able to do "docker logs $ID" and get the Swift logs.

We can see what processes are running, and there are quite a few, including rsyslog and memcached. Usually Docker models processes, but in this case I am taking a role-based approach to using Docker.

<pre>
<code>root@f2f8ccb82c0e:/# ps ax
  PID TTY      STAT   TIME COMMAND
    1 ?        Ss     0:00 /bin/bash
   43 ?        Ss     0:00 /usr/bin/python /usr/bin/supervisord -c /etc/supervisor/conf.d/supervisord.conf
   45 ?        S      0:00 /usr/bin/python /usr/bin/swift-container-server /etc/swift/container-server.conf
   46 ?        S      0:00 /usr/bin/python /usr/bin/swift-account-reaper /etc/swift/account-server.conf
   47 ?        S      0:00 /usr/bin/python /usr/bin/swift-object-replicator /etc/swift/object-server.conf
   48 ?        S      0:00 /usr/bin/python /usr/bin/swift-account-auditor /etc/swift/account-server.conf
   49 ?        S      0:00 /usr/bin/python /usr/bin/swift-object-server /etc/swift/object-server.conf
   50 ?        S      0:00 /usr/bin/python /usr/bin/swift-container-sync /etc/swift/container-server.conf
   51 ?        S      0:00 /usr/bin/python /usr/bin/swift-account-replicator /etc/swift/account-server.conf
   52 ?        S      0:00 /usr/bin/python /usr/bin/swift-account-server /etc/swift/account-server.conf
   53 ?        S      0:00 /bin/bash -c source /etc/default/rsyslog && /usr/sbin/rsyslogd -n -c3
   54 ?        S      0:00 /usr/bin/python /usr/bin/swift-proxy-server /etc/swift/proxy-server.conf
   55 ?        S      0:00 /usr/bin/python /usr/bin/swift-object-updater /etc/swift/object-server.conf
   56 ?        Sl     0:00 /usr/bin/memcached -u memcache
   57 ?        Sl     0:00 /usr/sbin/rsyslogd -n -c3
   86 ?        S      0:00 /usr/bin/python /usr/bin/swift-object-server /etc/swift/object-server.conf
   87 ?        S      0:00 /usr/bin/python /usr/bin/swift-container-server /etc/swift/container-server.conf
   88 ?        S      0:00 /usr/bin/python /usr/bin/swift-account-server /etc/swift/account-server.conf
   89 ?        S      0:00 /usr/bin/python /usr/bin/swift-proxy-server /etc/swift/proxy-server.conf
   91 ?        R+     0:00 ps ax
</code>
</pre>

Now we can use Swift.

<pre>
<code>root@f2f8ccb82c0e:/# swift -A http://127.0.0.1:8080/auth/v1.0 -U test:tester -K testing stat
       Account: AUTH_test
    Containers: 0
       Objects: 0
         Bytes: 0
  Content-Type: text/plain; charset=utf-8
   X-Timestamp: 1402590362.23352
    X-Trans-Id: tx1c32b455aa7c4178a4add-005399d49a
X-Put-Timestamp: 1402590362.23352
root@f2f8ccb82c0e:/# swift -A http://127.0.0.1:8080/auth/v1.0 -U test:tester -K testing upload etc_swift /etc/swift
etc/swift/dispersion.conf
etc/swift/account-server.conf
etc/swift/backups/1402588704.container.ring.gz
etc/swift/backups/1402588704.object.ring.gz
etc/swift/backups/1402588704.container.builder
etc/swift/proxy-server.conf
etc/swift/object-server.conf
etc/swift/swift.conf
etc/swift/backups/1402588704.object.builder
etc/swift/container-server.conf
etc/swift/backups/1402588704.account.builder
etc/swift/backups/1402588705.account.ring.gz
etc/swift/object.builder
etc/swift/backups/1402588705.account.builder
etc/swift/object.ring.gz
etc/swift/container.builder
etc/swift/account.ring.gz
etc/swift/supervisord.log
etc/swift/container.ring.gz
etc/swift/account.builder
etc/swift/supervisord.pid
root@f2f8ccb82c0e:/# 
</code>
</pre>

We can also start the container in non-interactive mode, and add a port mapping.

<pre>
<code>vagrant@host1:~$ ID=$(docker run -d -p 8080 -t serverascode/swift-onlyone)
</code>
</pre>

Now that container should be running with a port mapped to 8080.

<pre>
<code>vagrant@host1:~$ docker ps
CONTAINER ID        IMAGE                               COMMAND                CREATED             STATUS              PORTS                     NAMES
0c57f60e1de6        serverascode/swift-onlyone:latest   /bin/sh -c /usr/loca   3 seconds ago       Up 3 seconds        0.0.0.0:49162->8080/tcp   loving_hawking      
</code>
</pre>

Above we can see that port 49162 on the container host is mapped to 8080 on the container.

We can also check the logs.

<pre>
<code>     vagrant@host1:~$ docker logs $ID
Device d0r1z1-127.0.0.1:6010R127.0.0.1:6010/sdb1_"" with 1.0 weight got id 0
Reassigned 128 (100.00%) partitions. Balance is now 0.00.
Device d0r1z1-127.0.0.1:6011R127.0.0.1:6011/sdb1_"" with 1.0 weight got id 0
Reassigned 128 (100.00%) partitions. Balance is now 0.00.
Device d0r1z1-127.0.0.1:6012R127.0.0.1:6012/sdb1_"" with 1.0 weight got id 0
Reassigned 128 (100.00%) partitions. Balance is now 0.00.
Starting to tail /var/log/syslog...(hit ctrl-c if you are starting the container in a bash shell)
</code>
</pre>

Let's run stat again, but this time from the container host, not the container.

Note that it's not port 8080 any more for the auth url, it's 49162.

<pre>
<code>vagrant@host1:~$ swift -A http://127.0.0.1:49162/auth/v1.0 -U test:tester -K testing stat
       Account: AUTH_test
    Containers: 0
       Objects: 0
         Bytes: 0
  Content-Type: text/plain; charset=utf-8
   X-Timestamp: 1402590701.15270
    X-Trans-Id: txfebf58919cbf4c61ac73c-005399d5ed
X-Put-Timestamp: 1402590701.15270        
vagrant@host1:~$ swift -A http://127.0.0.1:49162/auth/v1.0 -U test:tester -K testing upload test swift.txt 
swift.txt
vagrant@host1:~$ swift -A http://127.0.0.1:49162/auth/v1.0 -U test:tester -K testing list test
swift.txt
</code>
</pre>

Check the logs again:

<pre>
<code>vagrant@host1:~$ docker logs $ID | tail
Jun 12 16:33:16 0c57f60e1de6 account-replicator: Replication run OVER
Jun 12 16:33:16 0c57f60e1de6 account-replicator: Attempted to replicate 1 dbs in 0.00341 seconds (292.99204/s)
Jun 12 16:33:16 0c57f60e1de6 account-replicator: Removed 0 dbs
Jun 12 16:33:16 0c57f60e1de6 account-replicator: 0 successes, 0 failures
Jun 12 16:33:16 0c57f60e1de6 account-replicator: no_change:0 ts_repl:0 diff:0 rsync:0 diff_capped:0 hashmatch:0 empty:0
Jun 12 16:33:16 0c57f60e1de6 object-replicator: Starting object replication pass.
Jun 12 16:33:16 0c57f60e1de6 object-replicator: 1/1 (100.00%) partitions replicated in 0.00s (767.48/sec, 0s remaining)
Jun 12 16:33:16 0c57f60e1de6 object-replicator: 1 suffixes checked - 0.00% hashed, 0.00% synced
Jun 12 16:33:16 0c57f60e1de6 object-replicator: Partition times: max 0.0003s, min 0.0003s, med 0.0003s
Jun 12 16:33:16 0c57f60e1de6 object-replicator: Object replication complete. (0.00 minutes)
</code>
</pre>

At this point we have a nice little OpenStack Swift install that we could use by linking with other containers.

## Using a data only container

It would be best to run with a data only container as well, and use that volume on /srv. 

So first create a data only container.

<pre>
<code>vagrant@host1:~$ docker run -v /srv --name SWIFT_DATA busybox
vagrant@host1:~$ docker ps --all  |grep SWIFT_DATA
6c13b4e27320        busybox:buildroot-2014.02           /bin/sh                7 seconds ago       Exit 0                          SWIFT_DATA   
</code>
</pre>

Now we can create a docker container with "--volumes-from" the SWIFT_DATA container.

<pre>
<code>vagrant@host1:~$ ID=$(docker run -d -p 8080 --volumes-from SWIFT_DATA -t serverascode/swift-onlyone)
</code>
</pre>

And if we inspect that container we can see where the container's volume is. Sorry that the lines will probably not wrap very nicely.

<pre>
<code>vagrant@host1:~$ docker inspect $ID | grep VolumesFrom
        "VolumesFrom": "SWIFT_DATA",
vagrant@host1:~$ docker inspect $ID | grep "\/srv"
        "/srv": "/var/lib/docker/vfs/dir/8d437b57f36a2d849cece0752c8316c6916c31ec12fd9049d4203662806d3fe2"
        "/srv": true
vagrant@host1:~$ sudo ls /var/lib/docker/vfs/dir/8d437b57f36a2d849cece0752c8316c6916c31ec12fd9049d4203662806d3fe2
sdb1
</code>
</pre>

I think the data only container is an ideomatic way to use Docker.

## Conclusion

I'm sure there is a lot that can be done to improve this Dockerfile, so please let me know if you see any issues or have any ideas. I have not done a lot of testing with it yet. But if it does work, then I think this is a nice way to quickly get access to a Swift instance, even if it is a purposely limited one, and hopefully help people move away from the complexity of a DFS when using containers.

## Issues

- At this time the proxy is not using SSL, nor is there an SSL terminator in front of the proxy, so it's all plain text. You wouldn't want to do this in production.
