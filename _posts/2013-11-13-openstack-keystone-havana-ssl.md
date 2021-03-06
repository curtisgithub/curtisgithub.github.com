---
layout: post
title: OpenStack Keystone with SSL
categories: 
---

# {{ page.title }}

In this post I want to quickly go over getting SSL enabled in OpenStack Keystone, specifically the Havana release, and on Ubuntu 12.04. I am going to setup SSL with self-signed certificates for testing.

Havana Ubuntu cloud repo and packages:

<pre>
<code>$ cat /etc/apt/sources.list.d/ubuntu_cloud_archive_canonical_com_ubuntu.list 
deb http://ubuntu-cloud.archive.canonical.com/ubuntu precise-updates/havana main
$ dpkg --list | grep keystone
ii keystone 1:2013.2-0ubuntu1~cloud0   OpenStack identity service - Daemons
ii python-keystone 1:2013.2-0ubuntu1~cloud0   OpenStack identity service - Python library
ii python-keystoneclient 1:0.3.2-0ubuntu1~cloud0 Client library for OpenStack Identity API
</code>
</pre>

To create test SSL key file, run this command:

<pre>
<code>$ keystone-manage ssl_setup --keystone-user keystone --keystone-group keystone
</code>
</pre>

Then add an ssl section to the keystone.conf file:

<pre>
<code>[ssl]
enable = True
certfile = /etc/keystone/ssl/certs/keystone.pem
keyfile = /etc/keystone/ssl/private/keystonekey.pem
ca_certs = /etc/keystone/ssl/certs/ca.pem
ca_key = /etc/keystone/ssl/certs/cakey.pem
</code>
</pre>

Setup at least the keystone "Identity Service" publicurl endpoint to use https, eg:

<pre>
<code>https://$keystone_srv:5000/v2.0
</code>
</pre>

Finally, when using the keystone command line client, use the insecure option (again, this is for testing):

<pre>
<code>$ cat adminrc 
export OS_SERVICE_ENDPOINT=https://$keystone_srv:5000/v2.0
export OS_SERVICE_TOKEN=$your_admin_token
$ . adminrc
$ keystone --insecure user-list
+----------------------------------+--------+---------+-------+
|                id                |  name  | enabled | email |
+----------------------------------+--------+---------+-------+
| d83ff7c66b4a4086b498c960fa3096fe | admin  |   True  |       |
+----------------------------------+--------+---------+-------+
</code>
</pre>


The above assumes there is at least the admin user in keystone. Otherwise, with no users it will just complete with no results.

If you don't use the insecure option you will get this error:

<pre>
<code>$ keystone user-list
<attribute 'message' of 'exceptions.BaseException' objects> 
(HTTP Unable to establish connection to https://$keystone_srv/v2.0/users)
</code>
</pre>

Please do let me know if I've made any errors by commenting, but so far this is working for me as a basic test of keystone with ssl support.

Finally, note that in most production situations, I believe keystone would be fronted by a separate SSL termination system of some kind (eg. OpenBSD's [relayd](http://www.openbsd.org/cgi-bin/man.cgi?query=relayd&sektion=8&format=html)). So this is just for testing, and perhaps getting to know keystone a bit better.
