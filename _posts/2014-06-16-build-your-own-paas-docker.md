---
layout: post
title: Build your own platform as a service with Docker
categories: 
header_image: /img/met/DT95.jpg
header_permalink: http://www.metmuseum.org/collection/the-collection-online/search/10464
---

# {{ page.title }}

First off, let me be clear--[Docker](http://www.docker.com/) is not a "platform as a service" (PaaS) by itself. However, I do think it's an important component, one that makes deploying a PaaS much simpler.

Second, for the most part I'm discussing the concept of building "your own PaaS" from my personal perspective, which is that I have a few blogs and Wordpress sites that I run for friends (ie. not a business venture or anything) and I have a single hardware server out there on the Internet that I use to do this. I thought it would be fun to use that server to think about how I one could provide a Wordpress PaaS that could potentially scale out to multiple hosts, even though I could never afford to or need to do that.

In a way, what I'm doing is creating a proof of concept (PoC) wordpress as a service, mostly for a single host which once in place could provide almost any application, and could potentially scale out to multiple container hosts.

## Why not just use Dokku?

[Dokku](https://github.com/progrium/dokku) is a piece of software created by [Jeff Lindsay](https://twitter.com/progrium) that essentially creates a single-host PaaS.

> Docker powered mini-Heroku. The smallest PaaS implementation you've ever seen.

But I'm not going to use Dokku, even though that is probably the best way to go about this. Instead I'm going to mostly layout the minimum systems and components needed to create a low-fi PaaS. Also [Deis](http://deis.io/), Flynn and [other](http://stackoverflow.com/questions/18285212/how-to-scale-docker-containers-in-production) container management systems, in various states with differing features, exist.

## Components

These are the minimum components I think you would need to create your own PaaS.

1. Wildcard DNS entry
2. "Web router" (such as [Hipache](https://github.com/dotcloud/hipache))
3. Docker - Application server/container provisioning and image creation
3. Application source code
5. Environment variables
6. Datastores, such as MySQL, NoSQL, object storage, etc
7. Something to tie them all together

## Wildcard DNS entry

The first thing you need is a wildcard DNS entry for a domain name. This [gist](https://gist.github.com/ngoldman/7287753) describes how to configure one using Namecheap, which happens to also be my registrar of choice (like a few other registrars, but not all, they support two factor authentication). So I bought a domain name like "somedomainapp.com" to run my "apps" and then configured a wildcard entry using Namecheap's DNS service. 

Obviously in a larger production environment you'd either manage your own nameservers or perhaps use something like [Google's DNS as a service](https://gist.github.com/ngoldman/7287753), which I would love to try out, and some loadbalancers or similar.

At this point, at minimum, you have a wildcard DNS entry pointing to an IP on your server which apps will be able to use, such as *.yourdomainapp.com.

## Web router

![](https://raw.githubusercontent.com/ccollicutt/ccollicutt.github.com/master/img/somedomain.png)
(_Above: hipache non-existent domain page_)

I'm not sure what to call this layer. Heroku calls this [HTTP routing](https://devcenter.heroku.com/articles/http-routing). Web routing works for me.

Essentially what his does is route incoming requests for apps to the right webserver, which in our case will be a docker container, or several docker containers. A request for someapp.somedomainapp.com would go to 127.0.0.1:49899 or 172.17.0.3:80 and such, which are docker containers.

In my situation I am using [hipache](https://github.com/dotcloud/hipache) which is backed by redis. This means you can add routes to hipache by entering them into redis, and hipache isn't required to restart because it will query redis for domain configuration. By default hipache allows the use of wildcard domains, so it'll route any configured entry, or send a default page if it doesn't exist.

My PoC python script, which is called "wpd" (more on that later), can dump the keys setup in redis for hipache. The below output means hipache will randomly balance requests for someapp.yourdomainapp.com to the two containers listed in redis.

<pre>
<code>$ wpd listkeys
someapp.yourdomainapp.com
===> http://127.0.0.1:49156
===> http://127.0.0.1:49157
$ redis-cli lrange someapp.yourdomainapp.com 0 -1
1) "someapp.yourdomainapp.com"
2) "http://127.0.0.1:49156"
3) "http://127.0.0.1:49157"
</code>
</pre>

There are many other ways to do "web routing." Dokku uses nginx. There is also [vulcand](https://github.com/mailgun/vulcand) which is backed by [etcd](https://github.com/coreos/etcd) and though it's new it sounds exciting. Hipache does support SSL, but as of a few weeks ago Vulcand did not, though I think it's on the roadmap (but I am a golang fanboy, so am biased).

## Docker!

Again comparing Heroku to what we are doing here, I think that Docker would fill the roles of [buildpack](https://devcenter.heroku.com/articles/buildpacks) and [dyno](https://devcenter.heroku.com/articles/dynos) though perhaps in terms of the buildpack part not containing a "slug" of the application code, rather only the environment the application would run in. Perhaps it's better to consider a Dockerfile a type of buildpack.

Using my wordpress example, the Dockerfile would create the image which docker then runs to create a wordpress application container, for example using apache2 + php.

Docker manages the container and provides networking and network address translation to expose the apache2's port to the web router.

So Docker is doing quite a bit of the work for us. Without Docker, we would probably need a way to create virtual machines images programmatically, and a way to start up and instance and get it networked, which could be as simple as something like [packer](http://www.packer.io/) and libvirtd (perhaps with kvm or lxc) or other combinations such as packer and openstack. Certainly that would require a few more resources. (Interestingly packer can build docker images as well.)

## Application source code

In Dokku code is pushed to a git repository which kicks off other processes. This is also how Heroku works. Those processes somehow get the code into the application container.

However, in my wordpress example, the wordpress code is downloaded in the startup script. Once the container is started from the wordpress image, the startup script runs something like:

<pre>
<code>if [ ! -e /app/wp-settings.php ]; then
        cd /app
        curl -O http://wordpress.org/latest.tar.gz
        tar zxvf latest.tar.gz
        mv wordpress/* /app
        rm -f /app/wordpress
        chown -R www-data:www-data /app
        rm -f latest.tar.gz
fi
</code>
</pre>

to grab the code. The url used to download could be taken from an environment variable instead of just being static as in the example.

The git push/recieve style probably makes more sense in a PaaS, but I haven't looked into what it takes to do that. Again Jeff Lindsay has [gitrecieve](https://github.com/progrium/gitreceive) and Flynn (a Jeff Lindsay project) has [gitrecieved](https://github.com/flynn/gitreceived). Also he has [execd](https://github.com/progrium/execd) and more. Busy guy!

Obviously there are a lot of ways to get code into the container to run it. If there is any one important thing a PaaS does, it's run your code.

## Environment variables

I think Docker images should be fairly generic. Also you don't want to commit configuration information to the image, such as passwords, so they should come from environment variables, and those variables need to get injected into the containers environment somehow.

For my wordpress example, I set environment variables with Docker. The Docker run command has the "-e" option which allows setting environment variables which will be exposed in the container. Below is an example.

<pre>
<code>$ docker run -e FOO=bar -e BAR=foo busybox env
HOME=/
PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
HOSTNAME=6cf2d6e8acb3
FOO=bar
BAR=foo
</code>
</pre>

My wordpress startup script checks for a few environment variables

<pre>
<code>DB_NAME=${DB_NAME:-"wordpress"}
DB_USER=${DB_USER:-"wordpress"}
DB_PASSWORD=${DB_PASSWORD:-"wordpress"}
DB_HOST=${DB_HOST:-$1}
</code>
</pre>

and uses them to create the wordpress configuration file with the right database settings.

Later on I'll talk about "tying the whole room together" which I do with a python script, and in it I set the environment variables for the container.

Below is a snippet of python code used to start a container with environment variables.

<pre>
<code>env = {
  'DB_HOST': MYSQL_HOST,
  'DB_NAME': dbname,
  'DB_USER': dbname,
  'DB_PASSWORD': dbpass,
}
container = dockerConn.create_container(image, detach=True, environment=env)
</code>
</pre>

Docker and python go well together using [docker-py](https://github.com/dotcloud/docker-py).

Another method is using a shared configuration system. Previously I mentioned etcd.

> [etcd is] a highly-available key value store for shared configuration and service discovery. 

etcd can store configuration information and [confd](https://github.com/kelseyhightower/confd) is a configuration management agent which can query etcd for information and generate configuration files that applications can use, and also restart the services that use those configuration files.

Given that I'm suggesting environment/configuration variables are a key part to PaaS, things like etcd, confd, and [consul](https://github.com/hashicorp/consul) are going to be important projects. But again, with my limited wordpress PaaS I'm just working on a very simplified PoC, where environment variables come at container runtime. However, I would imagine most larger PaaS or PaaS-like systems will use something like consul or etcd.

## Datastores

If your application needs to persist data, then it's got to put that data somewhere and the application container itself is not a good place. In general, I think there are two approaches: 

1. Another container 
2. A separate service

In terms of "a separate service" I'm talking about something like [Amazon RDS](http://aws.amazon.com/rds/) or [OpenStack Trove](https://wiki.openstack.org/wiki/Trove) (both of those being Database as a service) or object storage like [OpenStack Swift](http://docs.openstack.org/developer/swift/). In short I mean a service that is managed by someone else, perhaps the same provider that either runs Docker or the server that Docker is running on.

The other option is another container. Again using the wordpress example, instead of running one application container, perhaps I would run a second container that has a MySQL server running (or both services could even be in a single container). Maybe that MySQL service is a container, maybe it's a hardware server configured with Ansible. Who knows. Docker also recommends the [volumes from](https://docs.docker.com/userguide/dockervolumes/) approach, which works great when the data doesn't have to be distributed across multiple containers, which would be the case in a MySQL or [OpenStack Swift](http://serverascode.com/2014/06/12/run-swift-in-docker.html) container.

My feeling is that either is Ok, but I prefer to run a separate service. So in my wordpress example there is a single MySQL server that all wordpress apps will connect to, each having their own database. Perhaps that separate service is using Docker too. 

## Something to tie them all together

I'm working on a python script called "wpd" that ties all of this together, and what it does is:

1. Creates a site entry in the wpd MySQL database
2. Creates a database datastore that the site can use for wordpress
3. Creates couple of wordpress containers
4. Provides those containers with environment variables regarding how to connect to a worpress database
5. Adds that site to redis so that hipache can route/loadbalance requests to those containers

<pre>
<code>$ wpd -h
usage: wpd [-h] {listkeys,addsite,listsites,addimage,deploysite,dumpsite} ...

positional arguments:
  {listkeys,addsite,listsites,addimage,deploysite,dumpsite}
    listkeys            list all the keys in redis
    addsite             add a site to the database
    listsites           list all the sites in the database
    addimage            add a docker image to the database
    deploysite          startup a sites containers
    dumpsite            show all information about a site

optional arguments:
  -h, --help            show this help message and exit
</code>
</pre>

As you can see it has a few options, such as "addsite" and "deploysite". It's not complete yet. Adding a site just enters it into the wpd database, and deploying it means starting up containers and adding the information to redis so that hipache can route http requests to them.

What this would look like in a larger system...I'm not sure. It seems like a user management system more than anything else--users have sites, sites have names, containers, images and datastores.

## Issues

There are a few issues that I'd like to mention (though I'm probably not covering them all).

_Logging_ - Getting logs out of docker is still a bit of an problem. At this point likely you'll need to configure syslog in the container to ship logs to a centralized system. I expect that eventually docker will have more advanced ways of dealing with logs, if they don't already.

_File systems_ - Wordpress is a good example of a web application that is difficult to scale because it relies a lot on the file system for data persistence, such as media uploaded by users. In order to scale out a file system across multiple docker hosts you'll need a distributed file system, which is a huge pain and can also increase the size of your failure domain dramatically. I suggest not using file systems to store files and instead use object storage such as OpenStack Swift, which isn't as hard to deploy as some might think. What's more Swift doesn't think it can do [consistency and availability](http://en.wikipedia.org/wiki/CAP_theorem) at the same time.

_Datastore credentials_ - I'm not sure what the best way to securely store these is. Credentials and other important configuration information will need to be injected into the container somehow, and thus will need to be stored in a database or etcd or similar. Something to look into.

## Conclusion

In the end, I think that docker is a great system to use as a component in PaaS and that, to me, it simplifies rolling your own small platform as a service host. All you need is a web router, a Dockerfile and a docker host, a way to get the app into the container, and *pow* now you're cookin' with PaaS. Just remember to make your containers fairly generic, pipe in environment variables for configuration (or get them from somewhere else), and avoid file systems if possible.

## The Future!
 
Obviously I'm leaving a lot out. There's a huge difference between toying around with a PoC like this and running a production PaaS. I haven't even mentioned the terms "service discovery" or "container scheduling", which theoretically things like etcd and [libswarm](https://github.com/docker/libswarm) could take care of respectively, though I'm not sure libswarm will turn into a container scheduler. Recently Google released [Kubernetes](https://news.ycombinator.com/item?id=7873897) which is as docker cluster manager of some kind, though it currently only runs on GCE. Mesosphere is also working on its [Deimos](https://github.com/mesosphere/deimos) project. Further, CoreOS has [fleet](https://github.com/coreos/fleet) and Spotify [helios](https://github.com/spotify/helios). [Shipyard](http://serverascode.com/2014/05/25/docker-shipyard-multihost.html) can also control multiple docker hosts. Also there is [Havok](https://github.com/ehazlett/havok) which is a way to monitor docker containers and add them to vulcand via etcd. Neither have I discussed limiting resources, such as container memory and CPU and a about a billion other things, but I look forward to learning more.

## Updates

- Changed Apache Mesos to Mesosphere with regards to Deimos
- Added Shipyard and Havok

