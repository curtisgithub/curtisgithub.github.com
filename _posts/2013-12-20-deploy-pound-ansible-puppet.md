---
layout: post
title: Deploy Pound with Ansible or Puppet
categories: 
---

# {{ page.title }}

I just thought I would mention that I've put up [Puppet](http://forge.puppetlabs.com/serverascode) and [Ansible](https://galaxy.ansibleworks.com/list#/users/10) modules for deploying the Pound proxy and load-balancer. 

I've been working on learning Puppet lately, so that is the reason for that module, and then just yesterday AnsibleWorks released their "playbook" site, called [Galaxy](https://galaxy.ansibleworks.com), which performs a similar function to the Puppet Forge, so I thought I would try putting together a playbook/module and uploading it.

So if you use puppet you can download my Pound module with:

<pre>
<code>$ puppet module install serverascode/pound
</code>
</pre>

or if you use ansible you can do:

<pre>
<code>$ ansible-galaxy install serverascode.pound
</code>
</pre>

Next up is looking at SaltStack and then onto figuring out how to do testing of the modules. I'm thinking that testing will be the same with every module, so it should be easy to add that to each one, even if I end up supporting Ansible, Puppet, Chef, and SaltStack.
