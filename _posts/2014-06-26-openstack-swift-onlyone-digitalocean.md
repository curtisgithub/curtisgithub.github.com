---
layout: post
title: Deploy OpenStack Swift OnlyOne to Digital Ocean
categories:
---
 
# {{ page.title }}

In this blog post I want to show how to get your very own internet available object storage system using [OpenStack Swift](http://docs.openstack.org/developer/swift/) and [Docker](https://docker.com). Also it will be terminated by SSL (though with a self-signed certificate).

It's important to note that this is a special case OpenStack Swift setup--it only has one storage device and will only make one replica, which I call OpenStack Swift OnlyOne. Normally Swift installations are huge! But his one is small, which I think is cool. Or fun. But not fun and cool. That's too much.

This is what we are going to do:

- Boot a virtual machine on Digital Ocean
- Configure it to be able to use xattr
- Pull a few docker images
- Start three docker containers
- Create swift containers and upload files into them
- Set a container to serve an index.html page

## Get a docker virtual machine

Handily Digital Ocean has an image that comes with Docker 1.0 already. I'm going to use the tugboat CLI.

<pre>
<code>curtis$ tugboat images --global | grep Docker
Docker 1.0 on Ubuntu 14.04 (id: 4296335, distro: Ubuntu)
Dokku v0.2.3 on Ubuntu 14.04 (w/ Docker 1.0) (id: 4381169, distro: Ubuntu)
</code>
</pre>

Let's boot it. 66 is the 512MB image. 

_NOTE: If you really plan on using this for work instead of just testing Swift, a larger droplet size will likely be necessary. I did get some out of memory errors with the 512MB size._

<pre>
<code>curtis$ tugboat create swifty-onlyone -i 4296335 -s 66 -k 118429
Queueing creation of droplet 'swifty-onlyone'...done
curtis$ tugboat droplets
swifty-onlyone (ip: <droplet IP>, status: new, region: 4, id: 1945827)
</code>
</pre>

Wait until it's active, then ssh in.

<pre>
<code>curtis$ tugboat droplets
swifty-onlyone (ip: <droplet IP>, status: active, region: 4, id: 1945827)
curtis$ ssh root@<droplet IP>
SNIP!
root@swifty-onlyone:~# 
</code>
</pre>

Add xattr attribute to fstab for root and remount. 

_NOTE: Swift requires the file system support xattr. I'm not sure if it's enabled by default or not._

<pre>
<code>root@swift-onlyone:~# # vi /etc/fstab and add user_xattr
root@swift-onlyone:~# grep xattr /etc/fstab
UUID=050e1e34-39e6-4072-a03e-ae0bf90ba13a /               ext4    errors=remount-ro,user_xattr 0       1
root@swift-onlyone:~# mount -o remount /
</code>
</pre>

## Get docker images

Pull some docker images:

- busybox
- serverascode/swift-onlyone
- serverascode/pound

<pre>
<code>root@swift-onlyone:~# docker pull busybox; docker pull serverascode/swift-onlyone; docker pull serverascode/pound
</code>
</pre>

Now we have all those images locally.

<pre>
<code>root@swifty-onlyone:~# docker images
REPOSITORY                   TAG                   IMAGE ID            CREATED             VIRTUAL SIZE
serverascode/swift-onlyone   latest                1b562d4e3975        3 hours ago         349.2 MB
serverascode/pound           latest                2bfef1fdc39d        3 hours ago         285.2 MB
busybox                      buildroot-2013.08.1   d200959a3e91        3 weeks ago         2.489 MB
busybox                      ubuntu-14.04          37fca75d01ff        3 weeks ago         5.609 MB
busybox                      ubuntu-12.04          fd5373b3d938        3 weeks ago         5.455 MB
busybox                      buildroot-2014.02     a9eb17255234        3 weeks ago         2.433 MB
busybox                      latest                a9eb17255234        3 weeks ago         2.433 MB
</code>
</pre>


## Create the containers

We're going to create three containers:

1. SWIFT_DATA: A volume only container
2. SWIFT: Has OnlyOne installed, volume from SWIFT_DATA
3. A pound ssl termination container, linked to SWIFT

First, create a volume only container.

<pre>
<code>root@swift-onlyone:~# docker run -v /srv --name SWIFT_DATA busybox
root@swift-onlyone:~# docker ps -a | grep DATA
838c68ce031b        busybox:buildroot-2014.02   /bin/sh             15 seconds ago      Exited (0) 14 seconds ago                       SWIFT_DATA    
</code>
</pre>

Should see a volume in /var/lib/docker/volumes now.

<pre>
<code>root@swift-onlyone:~# ls /var/lib/docker/volumes/
1b6e87f07e2e5c0e49362bfa51f22fb8a32bca691a12d5c5872db0b90baf5241  _tmp
</code>
</pre>

Now create the OnlyOne container using a volume from SWIFT_DATA. Make sure to call it SWIFT.

Please note a couple of environment variables being set:

- *SWIFT_STORAGE_URL_SCHEME=https* Tells Swift Proxy to use https for the storage url
- *SWIFT_SET_PASSWORDS=yes* The startmain.sh script for Swift OnlyOne will change the default password for each use to one password.

<pre>
<code>root@swift-onlyone:~# docker run -d -e SWIFT_SET_PASSWORDS=yes -e SWIFT_STORAGE_URL_SCHEME=https --volumes-from SWIFT_DATA --name SWIFT -t serverascode/swift-onlyone
</code>
</pre>

If SWIFT_SET_PASSWORDS=yes was set, then the password will be echoed to the container log.

As an example, below it's been set to: laibiibooghu.

<pre>
<code>root@swift-onlyone:~# docker logs 6807caaaaf3b | head
Ring files already exist in /srv, copying them to /etc/swift...
Setting default_storage_scheme to https in proxy-server.conf...
storage_url_scheme = https
Setting passwords in /etc/swift/proxy-server.conf
user_test_tester = laibiibooghu .admin
user_test2_tester2 = laibiibooghu .admin
user_test_tester3 = laibiibooghu
Starting supervisord...
Starting to tail /var/log/syslog...(hit ctrl-c if you are starting the container in a bash shell)
Jun 27 16:46:24 6807caaaaf3b object-replicator: Starting object replicator in daemon mode.
</code>
</pre>

Finally create a pound container. This will be the ssl termination point and will be available from the Internet.

This container will be linked to the SWIFT container.

<pre>
<code>root@swift-onlyone:~# docker run -d --link SWIFT:SWIFT -p 443:443 -t serverascode/pound
</code>
</pre>

Now we have three containers, two of them running, and the other being the volume only container.

<pre>
<code>root@swift-onlyone:~# docker ps -a
CONTAINER ID        IMAGE                               COMMAND                CREATED              STATUS                         PORTS                  NAMES
2f6dcdae1db2        serverascode/pound:latest           /bin/sh -c /usr/loca   15 seconds ago       Up 14 seconds                  0.0.0.0:443->443/tcp   naughty_turing               
76d27dafa403        serverascode/swift-onlyone:latest   /bin/sh -c /usr/loca   About a minute ago   Up About a minute              8080/tcp               SWIFT,naughty_turing/SWIFT   
838c68ce031b        busybox:buildroot-2014.02           /bin/sh                About an hour ago    Exited (0) About an hour ago                          SWIFT_DATA
</code>
</pre>

Now from my laptop I can run the swift command line.

<pre>
<code>curtis$ alias sw='swift --insecure -A https://<droplet IP>/auth/v1.0 -U test:tester -K <password>'
curtis$ sw stat
       Account: AUTH_test
    Containers: 0
       Objects: 0
         Bytes: 0
  Content-Type: text/plain; charset=utf-8
   X-Timestamp: 1403882745.61961
    X-Trans-Id: tx28102150d50b484a92f3a-0053ad8cf9
X-Put-Timestamp: 1403882745.61961
</code>
</pre>

And upload a directory with a file in it.

<pre>
<code>curtis$ echo "hi" > index.html
curtis$ sw upload www index.html
index.html
</code>
</pre>

Set permissions so that anyone can read the files in the www container, ie. they are public.

<pre>
<code>curtis$ sw post --read-acl='.r:*,.rlistings' www
curtis$ sw stat www
       Account: AUTH_test
     Container: www
       Objects: 1
         Bytes: 3
      Read ACL: .r:*,.rlistings
     Write ACL:
       Sync To:
      Sync Key:
 Accept-Ranges: bytes
   X-Timestamp: 1403883848.54012
    X-Trans-Id: txd858295e7d294d39bdf3e-0053ad921d
  Content-Type: text/plain; charset=utf-8
</code>
</pre>

Make index.html the default web index.

<pre>
<code>curtis$ sw post -m 'web-index:index.html' www
</code>
</pre>

Now we can access that page in a web browser, and get the index.html. 

<pre>
<code>curtis$ wget --no-check-certificate https://<droplet IP>/v1/AUTH_test/www/
--2014-06-27 11:48:20--  https://<droplet IP>/v1/AUTH_test/www/
Connecting to <droplet IP>:443... connected.
WARNING: cannot verify <droplet IP>'s certificate, issued by '/C=US/ST=Oregon/L=Portland/O=IT/CN=172.17.0.13':
  Self-signed certificate encountered.
    WARNING: certificate common name '172.17.0.13' doesn't match requested host name '<droplet IP>'.
HTTP request sent, awaiting response... 200 OK
Length: 3 [text/html]
Saving to: 'index.html'

100%[====================================================================================================================================================>] 3           --.-K/s   in 0s      

2014-06-27 11:48:20 (109 KB/s) - 'index.html' saved [3/3]

curtis$ cat index.html 
hi</code>
</pre>

Note that I just wanted to use that as a demonstration, not the actual use case for Swift. Swift stores unstructured data, which we, as a planet, have a lot of. It doesn't have to serve web pages.

## Conclusion

Now for $5 a month you have a little swift install. The storage on that instance is pretty limited, at 20GB, but at any rate you can put all kinds of [DevOps reactions](http://devopsreactions.tumblr.com/) gifs there if you want. Or, perhaps use it to create interesting, proof-of-concept scalable web systems.

I should note as well that you could deploy OnlyOne in the same fashion on any Docker host, which is one of Docker's most interesting features.
