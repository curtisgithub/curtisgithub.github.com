---
layout: post
title: Installing chef on Centos 5
categories:
- serverascode
- chef
---

# {{ page.title }}

## Mirroring the FrameOS RPM repository

Installing Chef is pretty easy given that FrameOS has created all of the RPMs for us. See this [blog post](http://blog.frameos.org/2011/04/14/announcing-rbel-frameos-org/) for more information.

I mirror their repository on a local, centralized server using mrepo. Below is an example of my configuration.

<pre>
<code>[root@repos etc]# cd /etc/mrepo.conf.d/
[root@repos mrepo.conf.d]# cat chef.conf 
[chef]
# FRAMEOS builds RPMS for chefhere: 
# http://blog.frameos.org/2011/04/14/announcing-rbel-frameos-org/
#
# This might also be a good repo to use:
# - http://download.elff.bravenet.com/5/x86_64/
name = FrameOS rbel Chef RPMs $release ($arch)
release = 5
#arch = x86_64 i386
arch = x86_64
metadata = repomd repoview yum

### Additional repositories
chef = http://rbel.frameos.org/stable/el$release/$arch/
</code>
</pre>

This repo is then available to my servers at `http://repos/mrepo/chef-x86_64/RPMS.chef/`.

## Installing chef-server

Then I created a CentOS 5 virtual machine called `chef-server`. I enabled the repo I mention above on that server, and then ran:

<notextile>
<pre>
<code>[root@chef-server yum.repos.d]# yum install rubygem-chef-server
Loaded plugins: fastestmirror
Loading mirror speeds from cached hostfile
 * extras: ftp.telus.net
 * updates: repos
Setting up Install Process
Resolving Dependencies
--> Running transaction check
SNIP!
</code>
</pre>
<br />

I added these iptables rules to /etc/sysconfig/iptables.

<pre>
<code># Chef
# -- web interface
-A RH-Firewall-1-INPUT -m state --state NEW -m tcp -p tcp --dport 4040 -j ACCEPT
# -- chef-server
-A RH-Firewall-1-INPUT -m state --state NEW -m tcp -p tcp --dport 4000 -j ACCEPT
# -- amqp server
-A RH-Firewall-1-INPUT -m state --state NEW -m multiport -p tcp --dport 5672,4369,50229 -j ACCEPT
# -- search indexes (solr)
-A RH-Firewall-1-INPUT -m state --state NEW -m tcp -p tcp --dport 8983 -j ACCEPT
# data store (couchdb)
-A RH-Firewall-1-INPUT -m state --state NEW -m tcp -p tcp --dport 5984 -j ACCEPT
</code>
</pre>
<br />

And then ran the setup script (after taking a look at what it does :) ).

<pre>
<code>[root@chef-server sbin]# setup-chef-server.sh
Checking RabbitMQ...
RabbitMQ not running. Starting...
Starting rabbitmq-server: SUCCESS
rabbitmq-server.
Configuring RabbitMQ default Chef user...

Starting CouchDB...

Starting couchdb:                                          [  OK  ]
Enabling Chef Services...

Starting Chef Services...

Starting chef-server:                                      [  OK  ]
Starting chef-server-webui:                                [  OK  ]
Starting chef-solr:                                        [  OK  ]
Starting chef-expander:                                    [  OK  ]
</code>
</pre>
</notextile>

At which point I had a chef-server running, to which I can add clients and nodes.

## Installing chef-client

Installing chef-client is also easily done with the provided rpms. I created another CentOS 5 virtual machine, called chef-client. (Actually I created many of them. It's fun once you get it automated. :) )

<pre>
<code>[root@chef-client ~]# yum install chef-client
</code>
</pre>

(Note that this may not be the preferred way to bootstrap a chef-client, but it has been working for me.)

Then create a `client.rb` file in `/etc/chef`, where chef-server.example.com is the fqdn of your chef-server and is accessible on port 4000 from your chef-client.

<pre>
<code>[root@chef-client2 chef]# cat client.rb 
log_level        :info
    log_location     STDOUT
    chef_server_url  'http://chef-server.example.com:4000'
</code>
</pre>

Next, copy the `validation.pem` file from the chef-server to `/etc/chef` on the chef-client, likely using scp (or, do it when the server is built in a kickstart file :) ).

<pre>
<code>[root@chef-client2 chef]# ls
client.rb  validation.pem
</code>
</pre>

Then start `chef-client`.

<pre>
<code>[root@chef-client2 chef]# service chef-client start
</code>
</pre>

*But* in the /var/log/chef/client.log you will see an error that says client.pem is not present. This is good--chef-client will create the client.pem file.

<pre>
<code># Logfile created on [Date] 1 by logger.rb/22285
 INFO: Daemonizing..
 INFO: Forked, in 1762. Priveleges: 0 0
 INFO: *** Chef 0.10.0 ***
 INFO: Client key /etc/chef/client.pem is not present - registering
 WARN: Failed to read the private key /etc/chef/validation.pem: #<Errno::ENOENT: No such file or directory - /etc/chef/validation.pem>
 ERROR: Chef::Exceptions::PrivateKeyMissing: I cannot read /etc/chef/validation.pem, which you told me to use to sign requests!
 FATAL: Stacktrace dumped to /var/chef/cache/chef-stacktrace.out
 ERROR: Sleeping for 1800 seconds before trying again
 FATAL: SIGTERM received, stopping
</code>
</pre>

Now, restart chef-client so that new client.pem file can be used in conjuncation with the validation.pem file to register the node/client with the chef-server.

<pre>
<code>[root@chef-client2 chef]# service chef-client restart
</code>
</pre>

As long as the client.pem is there, the validation.pem is there, and the networking is OK, you should connect:

<pre>
<code> INFO: *** Chef 0.10.0 ***
 INFO: Run List is []
 INFO: Run List expands to []
 INFO: Starting Chef Run for chef-client2.example.com
 INFO: Loading cookbooks []
 WARN: Node chef-client2.example.com has an empty run list.
 INFO: Chef Run complete in 6.815418 seconds
 INFO: Running report handlers
 INFO: Report handlers complete
 FATAL: SIGTERM received, stopping
 INFO: Daemonizing..
 INFO: Forked, in 2032. Priveleges: 0 0
</code>
</pre>

And now the client should appear in the node and client lists. (Note that I have not detailed how to add a user/client to the chef system, you'll have to do that to use knife.)

<pre>
<code>[someuser@chef-server ~]$ knife node list
  chef-client2.example.com
SNIP!
[someuser@chef-server ~]$ knife client list
  chef-client2.example.com
SNIP!
</code>
</pre>

(The SNIP!s mean I've removed some items for brevity.)

Finally, run @chkconfig chef-client on@ on the chef-client to ensure the service starts at boot.

## Installing chef-client from a kickstart file

When building new vms I install chef-client from a kickstart file. This is also easily done!

The first important option in the kickstart file is the `repo` option.

<pre>
<code>repo --name=chef --baseurl=http://your_repo_server/mrepo/chef-x86_64/RPMS.chef/
</code>
</pre>

and then, in the `%packages` section simply add:

<pre>
<code>rubygem-chef
</code>
</pre>

which will be installed from the chef repo configured in the repo option.

Also, enable the service:

<pre>
<code>services --enabled chef-client
</code>
</pre>

Finally, in the `%post` section I add the below. Note the [PASTE THE CONTENTS OF YOUR VALIDATION.PEM HERE!!!] portion--that means put the results of @cat /etc/chef/validation.pem@ there, not that actual phrase. :)

<pre>
<code>%post
# chef-client

if [ ! -e /etc/chef ]; then
        mkdir /etc/chef
fi

cat > /etc/chef/client.rb << EOCLRB
log_level        :info
    log_location     STDOUT
    chef_server_url  'http://chef-server.example.com:4000'
EOCLRB
chmod 600 /etc/chef/client.rb

cat > /etc/chef/validation.pem << EOVALPEM
[PASTE THE CONTENTS OF YOUR VALIDATION.PEM HERE!!!]
EOVALPEM
chmod 600 /etc/chef/validation.pem
</code>
</pre>

When the server built from this kickstart boots chef-client will startup. However, it will fail the first time it starts up because the client.pem had to be generated. But, the next time it starts up it will connect to the chef-server and register. If you want it to register right away, then ssh into the server and run @service chef-client restart@ and it should register.





