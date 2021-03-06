---
layout: post
title: Deploying a boundary.com meter with ansible
categories:
---

# {{ page.title }}

Lately we have started using [Boundary](http://boundary.com) at work. While [puppet](https://forge.puppetlabs.com/puppetlabs/boundary)  and [chef](https://github.com/boundary/bprobe_cookbook) recipes and a [shell script](https://github.com/boundary/boundary_scripts) already exist to deploy a boundary meter onto a node/server, there was not a way to easily deploy one using [ansible](http://ansibleworks.com), which is my preferred configuration management and orchestration tool. 

Creating a boundary meter on a server is not complicated, but it was not really possible to do it with ansible without simply using ansible to copy a shell script up to the server. And, as ansible's documentation suggests, if you're pushing a script up to the server to run it with ansible, then it might be time to turn that bash script into an ansible module.

So, I have written a basic boundary meter module for ansible, and it is currently sitting in the [pull request](https://github.com/ansible/ansible/pull/3272) queue waiting to be reviewed, and hopefully added to the many [ansible modules](http://www.ansibleworks.com/docs/modules.html) in core. 

_NOTE: Both boundary and ansible move pretty fast, so it's likely that things will have changed even by the time I post this. :)_

## Obtaining the ca.pem file

_NOTE: It seems this file may be in the ubuntu package now, and is installed in /etc/bprobe/ca.pem.dkpg-dist_

One file that the bprobe package and my ansible module do not provide is the ca.pem file that is necessary for bprobe to contact boundary's api server.

The easiest way to get that file is to grab it from the bprobe_cookbook github repo.

<pre>
<code>$ wget https://raw.github.com/boundary/bprobe_cookbook/master/files/default/ca.pem
$ md5sum ca.pem
11f809a92ed1cc029c3ac86b42460a10  ca.pem
</code>
</pre>

or I believe it is also available in the official tar.gz release.

That file needs to go into /etc/bprobe and would likely be put there using a configuration management system of some kind, be it chef, ansible, puppet, saltstack, etc... :)

## Getting the bprobe client

While you don't need the bprobe client to create a meter, you do need bprobe in order to send data. So let's start by installing bprobe.

I'm installing the bprobe client on ubuntu 12.04. Below I'll simply show the parts of my ansible playbook that setup the official boundary repository and install bprobe. Even if you don't use ansible it's pretty straightforward to understand what's happening.

<pre>
<code>#Snippet of an ansible playbook

- name: ensure boundary repository is installed
  action: apt_repository repo='deb https://apt.boundary.com/ubuntu/ precise universe'
  
- name: ensure boundary repository gpg key is installed
  action: apt_key url=https://apt.boundary.com/APT-GPG-KEY-Boundary state=present

- name: install boundary bprobe
  action: apt pkg=bprobe state=installed force=yes
</code>
</pre>

Once those three actions have run, bprobe will be installed.

<pre>
<code>$ which bprobe
/usr/local/bin/bprobe
</code>
</pre>

## Creating a meter

In order to start sending data to boundary's api, that's all done with a restful api, we need to register a meter. In order to do that with ansible, you'll need the [boundary_meter module](https://github.com/ccollicutt/ansible/blob/devel/library/monitoring/boundary_meter) I (initially) wrote, which by now, if I'm lucky, will be in ansible's core set of modules.

Once that module is available to ansible, using it to create a meter looks like the below:

<pre>
<code>- name: register boundary meter
  action: boundary_meter apikey=AAAAAAA apiid=BBBBBB state=present \
  name=${ inventory_hostname }
  notify: restart bprobe

handlers:
  - name: restart bprobe
    action: service name=bprobe state=restarted
</code>
</pre>

where AAAAAA and BBBBBB are your organizations api id and api key for boundary.

## Monitoring

Once the meter is registered and bprobe up and running connecting back to boundary's api server, data should be flowing, and you can login to the boundary web gui and check out the node.

Thanks, and as always, if there are questions, suggestions, comments or criticisms, do let me know. :)
