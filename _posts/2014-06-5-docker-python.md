---
layout: post
title: Using Docker with Python and iPython
categories: 
---
 
# {{ page.title }}

Right now [Docker](http://docker.io) is one of the hottest projects on the planet, so that means some people aren't going to like it simply based on that fact alone.

Having said that, I really enjoy the paradigm shift in terms of working with containers, service discovery, and all the interesting new ideas and areas being created. (If you feel like having your mind warped, just read [Jeff Linday's](https://twitter.com/progrium) twitter feed.)

In this post I thought I would take a quick look at using the [docker-py](https://github.com/dotcloud/docker-py) module to use Docker containers via Python and, one of my favorite programming applications, [iPython](http://ipython.org/).

## Install docker-py

First, you need docker-py. Note that in the examples show here I am using Ubuntu Trusty/14.04.

<pre>
<code>$ pip install docker-py
</code>
</pre>

## ipython

I really like [iPython](http://ipython.org/) for exploring Python. It's kind of an advanced Python shell, but also does much more.

<pre>
<code>$ sudo apt-get install ipython
SNIP!
$ ipython
Python 2.7.6 (default, Mar 22 2014, 22:59:56) 
Type "copyright", "credits" or "license" for more information.

IPython 1.2.1 -- An enhanced Interactive Python.
?         -> Introduction and overview of IPython's features.
%quickref -> Quick reference.
help      -> Python's own help system.
object?   -> Details about 'object', use 'object??' for extra details.

In [1]:
</code>
</pre>

## Install docker

If docker isn't already installed, then go ahead and install it.

<pre>
<code>$ sudo apt-get install docker.io
</code>
</pre>

I also alias docker.io to docker.

<pre>
<code>$ alias docker='docker.io'
$ docker version
Client version: 0.9.1
Go version (client): go1.2.1
Git commit (client): 3600720
Server version: 0.9.1
Git commit (server): 3600720
Go version (server): go1.2.1
Last stable version: 0.11.1, please update docker
</code>
</pre>

Docker should now have a socket open that we can connect to.

<pre>
<code>$ ls /var/run/docker.sock 
/var/run/docker.sock
</code>
</pre>

## Pull an image

Let's download the busybox image.

<pre>
<code>$ docker pull busybox
Pulling repository busybox
71e18d715071: Download complete 
98b9fdab1cb6: Download complete 
1277aa3f93b3: Download complete 
6e0a2595b580: Download complete 
511136ea3c5a: Download complete 
b6c0d171b362: Download complete 
8464f9ac64e8: Download complete 
9798716626f6: Download complete 
fc1343e2fca0: Download complete 
f3c823ac7aa6: Download complete 
</code>
</pre>

Now we are ready to use docker-py.

## Working with docker-py

Now that we have docker-py, iPython, Docker, and the busybox image, we can start some containers!

If you're not familiar with iPython, have a look at the [tutorial](http://ipython.org/ipython-doc/stable/interactive/tutorial.html). iPython is quite powerful.

First, fire up ipython and import docker.

<pre>
<code>$ ipython
Python 2.7.6 (default, Mar 22 2014, 22:59:56) 
Type "copyright", "credits" or "license" for more information.

IPython 1.2.1 -- An enhanced Interactive Python.
?         -> Introduction and overview of IPython's features.
%quickref -> Quick reference.
help      -> Python's own help system.
object?   -> Details about 'object', use 'object??' for extra details.

In [1]: import docker
</code>
</pre>

Next we make a connection to Docker.

<pre>
<code>In [2]: c = docker.Client(base_url='unix://var/run/docker.sock',
   ...:                   version='1.9',
   ...:                   timeout=10)
</code>
</pre>

Now we have a connection to Docker.

iPython offers tab completion. If I type "c." and then hit the TAB key, ipython will show me what the Docker connection object has to offer. 

<pre>
<code>In [3]: c.
c.adapters                      c.headers                       c.pull
c.attach                        c.history                       c.push
c.attach_socket                 c.hooks                         c.put
c.auth                          c.images                        c.remove_container
c.base_url                      c.import_image                  c.remove_image
c.build                         c.info                          c.request
c.cert                          c.insert                        c.resolve_redirects
c.close                         c.inspect_container             c.restart
c.commit                        c.inspect_image                 c.search
c.containers                    c.kill                          c.send
c.cookies                       c.login                         c.start
c.copy                          c.logs                          c.stop
c.create_container              c.max_redirects                 c.stream
c.create_container_from_config  c.mount                         c.tag
c.delete                        c.options                       c.top
c.diff                          c.params                        c.trust_env
c.events                        c.patch                         c.verify
c.export                        c.port                          c.version
c.get                           c.post                          c.wait
c.get_adapter                   c.prepare_request               
c.head                          c.proxies   
</code>
</pre>

Let's look at c.images. I I put a "?" after the object, ipython will provide details about the object.

<pre>
<code>In [5]: c.images?
Type:       instancemethod
String Form:<bound method Client.images of <docker.client.Client object at 0x7f3acc731790>>
File:       /usr/local/lib/python2.7/dist-packages/docker/client.py
Definition: c.images(self, name=None, quiet=False, all=False, viz=False)
Docstring:  <no docstring>
</code>
</pre>

And grab the busybox image.

<pre>
<code>In [6]: c.images(name="busybox")
Out[6]: 
[{u'Created': 1401402591,
  u'Id': u'71e18d715071d6ba89a041d1e696b3d201e82a7525fbd35e2763b8e066a3e4de',
  u'ParentId': u'8464f9ac64e87252a91be3fbb99cee20cda3188de5365bec7975881f389be343',
  u'RepoTags': [u'busybox:buildroot-2013.08.1'],
  u'Size': 0,
  u'VirtualSize': 2489301},
 {u'Created': 1401402590,
  u'Id': u'1277aa3f93b3da774690bc4f0d8bf257ff372e23310b4a5d3803c180c0d64cd5',
  u'ParentId': u'f3c823ac7aa6ef78d83f19167d5e2592d2c7f208058bc70bf5629d4bb4ab996c',
  u'RepoTags': [u'busybox:ubuntu-14.04'],
  u'Size': 0,
  u'VirtualSize': 5609404},
 {u'Created': 1401402589,
  u'Id': u'6e0a2595b5807b4f8c109f3c6c5c3d59c9873a5650b51a4480b61428427ab5d8',
  u'ParentId': u'fc1343e2fca04a455f803ba66d1865739e0243aca6c9d5fd55f4f73f1e28456e',
  u'RepoTags': [u'busybox:ubuntu-12.04'],
  u'Size': 0,
  u'VirtualSize': 5454693},
 {u'Created': 1401402587,
  u'Id': u'98b9fdab1cb6e25411eea5c44241561326c336d3e0efae86e0239a1fe56fbfd4',
  u'ParentId': u'9798716626f6ae4e6b7f28451c0a1a603dc534fe5d9dd3900150114f89386216',
  u'RepoTags': [u'busybox:buildroot-2014.02', u'busybox:latest'],
  u'Size': 0,
  u'VirtualSize': 2433303}]
</code>
</pre>

Create a container. Note that I'm adding a command to be run, in this example the "env" command.

<pre>
<code>In [8]: c.create_container(image="busybox", command="env")
Out[8]: 
{u'Id': u'584459a09e6d4180757cb5c10ac354ca46a32bf8e122fa3fb71566108f330c87',
 u'Warnings': None}
</code>
</pre>

Start the container using the Id.

<pre>
<code>In [9]: c.start(container="584459a09e6d4180757cb5c10ac354ca46a32bf8e122fa3fb71566108f330c87")
</code>
</pre>

And we can check the logs, should see the output of the command "env" that we configured when the container was created.

<pre>
<code>In [11]: c.logs(container="584459a09e6d4180757cb5c10ac354ca46a32bf8e122fa3fb71566108f330c87")
Out[11]: 'HOME=/\nPATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin\nHOSTNAME=584459a09e6d\n'
</code>
</pre>

If I run a container with the same options using the dokcer command line, I should see something similar.

<pre>
<code>$ docker run busybox env
HOME=/
PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
HOSTNAME=ce3ad38a52bf
</code>
</pre>

As far as I can tell, docker-py does not have a run option, instead we have to create a container and then start it.

Here's another example:

<pre>
<code>In [17]: busybox = c.create_container(image="busybox", command="echo hi")

In [18]: busybox?
Type:       dict
String Form:{u'Id': u'34ede853ee0e95887ea333523d559efae7dcbe6ae7147aa971c544133a72e254', u'Warnings': None}
Length:     2
Docstring:
dict() -> new empty dictionary
dict(mapping) -> new dictionary initialized from a mapping object's
    (key, value) pairs
dict(iterable) -> new dictionary initialized as if via:
    d = {}
    for k, v in iterable:
        d[k] = v
dict(**kwargs) -> new dictionary initialized with the name=value pairs
    in the keyword argument list.  For example:  dict(one=1, two=2)

In [19]: c.start(busybox.get("Id"))

In [20]: c.logs(busybox.get("Id"))
Out[20]: 'hi\n'
</code>
</pre>

If you haven't used busybox images with docker yet, I definitely suggest it. I also suggest the debian:jessie image which is only 120MB, quite a bit smaller than, say, the Ubuntu images.

## Conclusion

Docker is a fascinating new system and it's going to be used to build interesting new technologies, especially around cloud services. Using iPython we've explored how to programmatically create docker containers using the docker-py module. Now using python we can create those next generation ideas using docker and containers. 






