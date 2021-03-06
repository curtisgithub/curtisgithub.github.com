---
layout: post
title: apt-cacher-ng
categories: 
---

# {{ page.title }}

apt-cacher-ng is an package caching system for apt packages. And I suppose it must be the "next generation" version. :)

I find it to be an indespensible system--especially when I am creating complex multi-virtual machine systems on my laptop--because it allows me to only have to download each package once from the Internet, and every vm can just grab the package from apt-cacher-ng. So if you have one, five or ten vms needing the same package, it's still only downloaded once from the Internet but many times from your local cache server.

Configuring it is easy!

<pre>
<code>package_cache_srv$ sudo apt-get install apt-cacher-ng
package_cache_srv$ sudo vi /etc/apt-cacher-ng/acng.conf
# edit config, change the bind address to: BindAddress: 0.0.0.0
package_cache_srv$ sudo service apt-cacher-ng restart
</code>
</pre>

Then on each system that you want to use the apt-cache server and a proxy server configuration file for apt:

<pre>
<code>vm_1$ cat /etc/apt/apt.conf.d/01proxy
Acquire::http { Proxy "http://<package_cache_srv IP address>:3142"; };
</code>
</pre>

Now each vm configured with the proxy file will use the apt-cache-ng server to obtain packages. 
