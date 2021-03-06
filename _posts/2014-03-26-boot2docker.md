---
layout: post
title: boot2docker on OSX
categories: 
---
 
# {{ page.title }}

Containers are cool. There I said it. LXC is production ready, Docker is a new and very popular project. Containers are hip and that's Ok with me. (Note that when I say containers I pretty much just mean LXC and systems based on LXC, there are other container systems out there, and have been for some time.)

In this post I'll look at using boot2docker on OSX. In a previous post I was using [boot2docker via libvirt and kvm](http://serverascode.com/2014/03/13/boot2docker-qemu.html), but now I'll use it on OSX, mostly because I have an Apple macbook and really need to learn more about docker.

## So...what is boot2docker?

From the [README.md](https://github.com/boot2docker/boot2docker) file:

> boot2docker is a lightweight Linux distribution based on Tiny Core Linux made specifically to run Docker containers. It runs completely from RAM, weighs ~24MB and boots in ~5s (YMMV).

boot2docker will use virtualbox on OSX to boot a [tiny core linux](http://distro.ibiblio.org/tinycorelinux/) based operating system that has docker already installed. Using tiny core to provide access to docker is a really interesting model, and is necessary because OSX doesn't have a "container"-like system (yet), so in order to use containers locally on an OSX laptop or workstation you need to boot a Linux virtual machine.

Docker is:

> ...an open-source project to easily create lightweight, portable, self-sufficient containers from any application. The same container that a developer builds and tests on a laptop can run at scale, in production, on VMs, bare metal, OpenStack clusters, public clouds and more. -- [Docker](https://www.docker.io/)

## Install boot2docker on OSX

Thanks to [brew](http://brew.sh/), installation is easy.

<pre>
<code>curtis$ brew update
SNIP!
curtis$ brew install boot2docker
SNIP!
</code>
</pre>

These are the versions I have:

<pre>
<code>curtis$ brew list --versions | grep docker
boot2docker 0.7.0
docker 0.9.0
</code>
</pre>

Thanks brew!

## Use boot2docker

Now that boot2docker is installed, we can initialize it.

<pre>
<code>curtis$ boot2docker init
[2014-03-25 17:31:47] Creating VM boot2docker-vm
Virtual machine 'boot2docker-vm' is created and registered.
UUID: 1e60ddea-8794-4c32-b488-ef6b054628e8
Settings file: '/Users/curtis/VirtualBox VMs/boot2docker-vm/boot2docker-vm.vbox'
[2014-03-25 17:31:48] Apply interim patch to VM boot2docker-vm (https://www.virtualbox.org/ticket/12748)
[2014-03-25 17:31:48] Setting VM settings
[2014-03-25 17:31:48] Setting VM networking
[2014-03-25 17:31:48] boot2docker.iso not found.
[2014-03-25 17:31:49] Latest version is v0.7.0, downloading...
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100   342  100   342    0     0    508      0 --:--:-- --:--:-- --:--:--   508
100 24.0M  100 24.0M    0     0  1111k      0  0:00:22  0:00:22 --:--:-- 1332k
[2014-03-25 17:32:11] Done
[2014-03-25 17:32:11] Setting VM disks
[2014-03-25 17:32:11] Creating 40000 Meg hard drive...
Converting from raw image file="stdin" to file="/Users/curtis/.boot2docker/boot2docker-vm.vmdk"...
Creating dynamic image with size 41943040000 bytes (40000MB)...
[2014-03-25 17:32:11] Done.
[2014-03-25 17:32:11] You can now type boot2docker up and wait for the VM to start.
</code>
</pre>

Now we can start the boot2docker vm.

<pre>
<code>curtis$ boot2docker
Usage /usr/local/bin/boot2docker {init|start|up|save|pause|stop|restart|status|info|delete|ssh|download}
curtis$ boot2docker up
[2014-03-25 17:33:58] Starting boot2docker-vm...
[2014-03-25 17:34:18] Started.

To connect the docker client to the Docker daemon, please set:
export DOCKER_HOST=tcp://localhost:4243
</code>
</pre>

Now that the vm is up and running via boot2docker, we can ssh into the vm. For this image, at this time, the password to login is "tcuser."

<pre>
<code>curtis$ boot2docker ssh
docker@localhost's password: #enter "tcuser" as the password
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
docker@boot2docker:~$ ifconfig eth0
eth0      Link encap:Ethernet  HWaddr 08:00:27:B8:BB:5C  
          inet addr:10.0.2.15  Bcast:10.0.2.255  Mask:255.255.255.0
          inet6 addr: fe80::a00:27ff:feb8:bb5c/64 Scope:Link
          UP BROADCAST RUNNING MULTICAST  MTU:1500  Metric:1
          RX packets:88 errors:0 dropped:0 overruns:0 frame:0
          TX packets:60 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:1000 
          RX bytes:10303 (10.0 KiB)  TX bytes:8361 (8.1 KiB)
</code>
</pre>

Check it out, there's the cool docker whale guy.

We'll do as boot2docker suggests and export the DOCKER_HOST variable.

<pre>
<code>curtis$ export DOCKER_HOST=tcp://localhost:4243
# also put it in my .profile, and add an alias so I don't have to
# type docker all the time :)
curtis$ tail -2 ~/.profile 
export DOCKER_HOST=tcp://localhost:4243
alias d='docker'
</code>
</pre>

Now we can user the docker command. Note that I'm running this from OSX.

<pre>
<code># note I'm using my alias d=docker
curtis$ d info
Containers: 0
Images: 0
Driver: aufs
 Root Dir: /mnt/sda1/var/lib/docker/aufs
 Dirs: 0
Debug mode (server): true
Debug mode (client): false
Fds: 10
Goroutines: 13
Execution Driver: native-0.1
EventsListeners: 0
Kernel Version: 3.13.3-tinycore64
Init Path: /usr/local/bin/docker
</code>
</pre>

Ok, so here it is important to note that we are running the docker command on OSX, but the docker server is running inside a boot2docker based virtual machine created by Vagrant an the boot2docker scripts.

We can also check the status of the boot2docker vm. 

NOTE: I have noticed that when I move from home to work sometimes virtual machines network stops working. I haven't looked into this much, but a couple of times I had to restart the docker virtual machine with "boot2docker stop; boot2docker start."

<pre>
<code>curtis$ boot2docker status
[2014-03-26 12:51:09] boot2docker-vm is running.
</code>
</pre>

Lets download the busybox image to play with. This is part of the [hello world](http://docs.docker.io/en/latest/examples/hello_world/#hello-world) example on the docker documentation website.

<pre>
<code>curtis$ d pull busybox
Pulling repository busybox
769b9341d937: Download complete                                                 
511136ea3c5a: Download complete 
bf747efa0e2f: Download complete 
48e5f45168b9: Download complete 
</code>
</pre>

Unfortunately at this point I seem to be running into a bug. When I run a docker command I get a "Coudn't send EOF: use of a closed network connection" error.

<pre>
<code>curtis$ d run -i -t busybox echo "hi"
hi
# have to hit enter...
[error] client.go:2264 Couldn't send EOF: use of closed network connection
</code>
</pre>

I'll have to come back to this blog post once I find out more about this bug.

I did find it entered into a [github issue](https://github.com/boot2docker/boot2docker/issues/184#issuecomment-38740712) so it should be fixed soon, and once it is I'll come back and finish up this blog post.

I can still ssh into the boot2docker vm and run commands without that issue, of course.

<pre>
<code>curtis$ boot2docker ssh
SNIP!
docker@boot2docker:~$ docker run -i -t busybox echo "hi"
hi
</code>
</pre>

In reality we haven't come very far in this blog post, partially because there seems to be a bug in the particular versions of boot2docker and docker I am using, but at least we got boot2docker installed, and are ready to move onto actually using docker straight from OSX. Once I figure out that bug I'll come back and update this post with some more things I learned about Docker.
