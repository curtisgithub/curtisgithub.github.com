---
layout: post
title: Basic infrastructure to support production linux servers
categories:
- infrastructure
---

# {{ page.title }}

Every IT group providing Linux servers will require some infrastructure services. What I mean by that is that there are services that Linux sysadmins need that help them to run their servers in a efficient, scalable way. 

Production services need support from __infrastructure services__.

This post lists a basic collection of services that a Linux sysadmin might require. For me, this would be the *minimum* services required to run a group of Linux servers, be it 10 or 1000. 

Notes:
# I mention specific solutions, but there are many different ways to obtain the same basic infrastructure.
# Most of what I discuss below is geared towards Redhat servers, but is also completely applicable to any Linux disto that has a packaging system, which is pretty much all of them.

*1. Write documentation with mediawiki*

One of the most important things a sysadmin does is document what they did so that they can take a relaxing, rejuvenating vacation. By this I mean that they have documented their systems so that the other admins taking over can use it to fix things, to understand what is installed, where it is, what it does, who owns it, ect, when the primary sysadmin is unavailable because he/she is in downtown Tokyo and is having a hard time connecting to the local wireless because they can't read Japanese.

Documentation is also important so you can remember what you did six months ago to get iscsi working, or how to configure a xen dom0 to use a bridge that comes from a vlan, or that command to dd a logical volume from one server to another over ssh without having to figure it out all over again. I suppose you could [blog](http://serverascode.com) about it too. *:)*

Regardless of what is being documented, usually the system I will use is a [Mediawiki](http://www.mediawiki.org) instance. Mediawiki supports searching (also full text searching if you use [sphinx](http://sphinxsearch.com/) and the [sphinx search plugin](http://www.mediawiki.org/wiki/Extension:SphinxSearch) so that you can search in pre tags too), file uploads, categories, and many, many other features, especially via the diverse plugin/extension community. 

When people don't want to use mediawiki I ask them what they want to do, how they want to work, and inevitably mediawiki can do it.

*2. Serve packages using mrepo*

Most Linux distros use _some_ form of package management to install software. For example, in Debian/Ubuntu a deb file and in Redhat it's an rpm. 

These packages, somewhat akin to an advanced zip file, contain all the files, requirements, metadata, ect, for a particular piece of software or service. The @./configure; make; make install@ dance is only done on the build servers.

In most of the environments I work in, we strive to *package all code*. This means that all software and applications are installed on a server in a package. Configuration can be done by hand, or via a centralized configuration management system, but the code is actually installed via a package. This goes for Perl CPAN modules too. :) _EVERYTHING_.

The *package server* provides a central spot that the servers obtain packages from, eg. a [mrepo](http://dag.wieers.com/home-made/mrepo/) server. 

The packages server will do three major things:

** Download RPM packages from external repositories, eg. the official Redhat repositories, EPEL, RPMForge, ect.
Also it will allow us to store our own *custom packages*.
This means our servers don't go to the internet to get updates, they go to the packages server.

** Serve those packages to all of our Linux servers.

** Allow for the ability to  “freeze” the repositories so that we can install software updates in test environments, test them, and then install the exact same version of the packages/software in production, thus being a _somewhat_ more sure that everything is going to work OK after the update.

*3. Centralize syslogging with rsyslog*

There should be a central syslog server somewhere in your infrastucture, and all production servers should send syslog packets to that server. 

This will allow for a central spot for gathering syslog messages from all servers. We can then run scripts/processes to analyze these logs looking for issues.

Also, should a server get *hacked* the first thing _malicious users_ usually do is (try) to delete logs. But, if the logs have been sent to a central log server then they (probably) can't do that.

I usually replace, when possible, syslog with [rsyslog](http://www.rsyslog.com/) and at minimum use TCP delivery instead of UDP.

*4. Centralize root email with sendmail or postfix and dovecot*

There are many scripts on servers that will send email to root if something breaks, eg. cron or logwatch. So all of these emails should be sent to a central email address that sysadmins have access to.

I'm not a huge fan of what is essentially logging over email, but it's nearly impossible to avoid.

Currently I configure each server's root alias to email a central address which is delivered to a Maildir and serve that up over IMAPS with [dovecot](http://www.dovecot.org). Dovecot is pretty great.

(Note that while sendmail is default on RHEL5, postfix is default on RHEL6!)

*5. Serve kickstarts over http*

The kickstart server is simply a plain http server that serves kickstart files, and in conjunction with the packaging server allows for rapid, repeatable installation of Redhat/CentOS. Or you could do it over nfs, or pop it on the USB key. I always use a web server to serve up the kickstarts.

`ks=http://example.com/ks/newserver.ks` if you know what I mean. :)

*6. Centralize configuration management*

This is a central server that can be used to access all the Linux/Unix servers via ssh and to try automate with custom scripts or things like [pdsh](https://code.google.com/p/pdsh/), [chef](http://www.opscode.com/chef/), [puppet](http://www.puppetlabs.com/), [fabric](http://www.fabfile.org), ect.

Also stores a copy of every Linux servers RPM database so that if a server gets hacked you can check what files have changed, if any, on the hacked server. (Though that should be in backups too.)

*7. Build packages with rpmbuild*

To create custom RPM packages a build server for each OS and architecture is required.  So if you run RHEL5 and RHEL 6 on x86_64 then you'll need two build servers, one for each OS and arch.

This is where @rpmbuild -ba some.spec@ will be run to build a RPM. Then the RPM will be copied to the packaging server.

*8. Login from anywhere securely with ssh*

(Of course with ssh!)

Linux/Unix admins should be able to access a ssh gateway from the Internet to be able to access servers and possibly their workstations or just the central management server.

Only public key authentication would be allowed to this server (meaning no password based authentication) which makes it very secure in terms of auth.

Some workplaces will want to put this behind a commercial VPN. Try to avoid this...IMHO ssh is one of the, if *not the*, most secure network applications on the planet--certainly better than some million [SLOC](https://secure.wikimedia.org/wikipedia/en/wiki/Source_lines_of_code) ssl "vpn". 

PS. Did you know you can create a ssh-based vpn with something like [sshuttle](https://github.com/apenwarr/sshuttle?) If you did sysadmin 2pts for you! *:)*

*9. Monitor uptime with nagios*

Essentially a [Nagios](http://www.nagios.org) server, or similar, that monitors production services and servers and will let you know when they go down. I have also used hobbit, which is apparently called [Xymon](http://sourceforge.net/projects/xymon/) now.

(But they won't go down, right? In fact, the monitoring system will go down more than the production serivces, won't it. *:)* I've always thought that was the hard part of uptime monitoring.)

*10. Manage code with version control systems*

Every IT workplace should have a central code repository. Sure, hg and git are distributed, but I still think it's nice to have a central location. "Github":github.com seems to be successful at centralizing distributed revision control, so I think centralizing a local git or hg instance (with hgweb.cgi for example) will work for most workplaces as well. :)

I have a github account and have used svn and hg at work. Use whatever you want--just do it now!

*11. Backup with rdiff-backup*

This would be a server used to rsync snapshot-type backups over ssh to disk for rapid restores. I like [rdiff-backup](http://www.nongnu.org/rdiff-backup/)

It would likely compliment that huge Netbackup, or Commvault (calmvault?), or other backup system you have that doesn't work half the time and requires the installation of a gigantic root-running tarball or monolithic 600MB RPM which doubles the size of your 500MB minimal Redhat OS install. Good times in commercial backup land.

*12. Test with jmeter*

If you have dev, test, production environments then you should also automate testing. This would be the service that helps you do that, running Jmeter and/or Selenium for example; do everything from one location, perhaps when a RPM/package is built then it's automatically tested from here. Wouldn't that be nice!?

*13. Monitor performance with cacti*

Everyone wants performance. Or do they? How does one know how much performance one needs? You have to do performance testing, and you have to monitor performance. You can't just buy performance, you have to put some work in.

That said, for monitoring performance I usually use either [Nagios](http://www.nagios.org) or something like [Cacti](http://www.cacti.net) using snmp with some custom scripts.

*14. Secure remote bios level access via a remote KVM or IPMI*

By this I mean the secure network you attach your remote KVM to the hardware dom0 servers, and/or to the IPMI interfaces that most teir 1 servers come with (not that I endorse only tier 1 vendors) so that you don't have to hang out in that cold, loud, unfriendly server room.
