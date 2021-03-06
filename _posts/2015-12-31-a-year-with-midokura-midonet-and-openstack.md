---
layout: post
title: A Year with Midokura's Midonet and OpenStack
categories:
header_image: /img/buffalo.jpg
header_permalink: https://flic.kr/p/7iTCBV
---

# {{ page.title }}

One of the first things to do when deciding to deploy OpenStack is figure out what the Neutron network is going to look like. For that, one needs the project requirements, what your network administrators are willing to deploy (allow), and what the operators are comfortable using. Maybe those are all the same person, I don't know. :) There are all the general Neutron deployment options, as well as plugins like [Project Calico](http://www.projectcalico.org/), [Plumgrid](http://www.plumgrid.com/), and [many others](https://wiki.OpenStack.org/wiki/Neutron_Plugins_and_Drivers) to choose from. One might say you have a [plethora of choices](http://www.imdb.com/title/tt0092086/quotes?item=qt0400510).

Before I get too far, I should note that I don't have a lot of experience with non-Midonet deployments of Neutron, so I have "strong opinions, lightly held" in terms of how to deploy Neutron. 

My feeling when deciding to use Midonet was that the more generic Neutron deployment model, something like [HA with DVR](http://docs.openstack.org/networking-guide/scenario_dvr_ovs.html), sounded complicated and in some cases limited. Tales of debugging sounded awful. OpenStack operators love to anecdotally complain about standard Neutron deployment models, so I thought I'd just skip all that and implement Midonet. I needed a highly available system that also supported a [VPC](https://aws.amazon.com/vpc/) style of tenant networking. Also I like software defined networking (whatever that actually means). Midonet seemed like a perfect fit, and after a year in production, I feel it was a good choice. Now if I complain about networking in OpenStack I'd be really be complaining about Midonet as opposed to Neutron (which is essentially just the front-end API).

However, other networking models should be considered. I think it's safe to say I have experience operating Enterprise Midonet 1.8, but after that things get fuzzy. So complete whatever due-diligence seems necessary for your project.

## What is Midonet?

From my perspective as an OpenStack operator, Midonet is a Neutron plugin and software defined networking system. It falls into the category of "overlay" models. It creates a logical network topology over top of the physical one, essentially using VXLAN (or GRE) encapsulation. The network state is distributed and stored mostly in Zookeeper. The Midolman agent runs on API, gateway, and compute nodes. Midonet has good documentation, so I suggest taking at look at that to get a better understanding of how it all works.

Here's the standard Midonet diagram. It shows how Midonet "overlays" on top of the physical network.

![](/img/midonet_topology.png)


## Things I like about Midonet

There are many things to like about Midonet.

- _It is open source, though I'm not sure how many non-Midokura developers contribute_

Note: I have not deployed the open source version, only the commercial 1.8 series

- _BGP-based HA on the gateway nodes for north/south_

This means you can drop a gw node and only lose about 10 packets before the IP fails over. I really like this feature. Requires your network admin to do some work setting up BGP sessions. Midonet takes care of them on its side when BGP ports are configured. Also requires stateful port groups to be configured, but that is not hard. I'm not sure why anyone would deploy Midonet gateways without doing that.

- _Good support from Midokura_

I don't recommend many commercial systems, but I do recommend Enterprise Midonet. The support team has been fantastic.

- _Midokura is a modern, fast moving company with new ideas_

There are lots of interesting things that could/can be done with a system that enables SDN. Think about anycast with floating IPs across WAN-connected regions, cool stuff like that. An overlay is certainly not less complicated, but the features and capabilities it adds are worth it in my opinion.

- _Because it is software features can be added without requiring new hardware_
- _Has an API_
- _Gets rid of standard Neutron network nodes, though north/south goes through the gw nodes_

Some Neutron deployments require dedicated network nodes.

- _Your network admins can probably ignore it and just provide L3 connectivity_
- _Security groups are not iptables based_
- _Potential for service chaining, NFV, all that hyped up stuff_
- _Does have a web interface, but I have never used it_
- _Midokura is working hard on important OpenStack projects like [Kuryr](http://superuser.openStack.org/articles/project-kuryr-brings-container-networking-to-openStack-neutron_)
- _Layer-3 forwarding in the hypervisor_
- _LBaaS driver does work, though has some caveats_

## Things that aren't as much fun

- _Configuring ports, BGP sessions, and stateful port groups_

This is a bit painful for me, only because I didn't have time to write an Ansible module for it. This process would be easily automated, but if it isn't, then it is fairly complicated. In the end I wrote a terrible bash script to do it for me, which made me feel bad.

- _Does have the extra requirements of Zookeeper and Cassandra_

Many people dislike Zookeeper. I'm not sure why. I'm not a big fan of Java-based systems, but I can tell you that Zookeeper has been running fine for over a year with absolutely no issues. That said it's recommended to run the network service database (NSDB) nodes, ie. Zookeeper and Cassandra, on baremetal servers that are separate from the rest of the infrastructure. Nodes with 32GB or more is fine. Certainly these clusters require some care and feeding.

Then again, the network state has to go somewhere. Other solutions have to store it as well. Physical routers store network state too, just in config files or other local databases. Every SDN system has to store state, and usually that will be some kind of distributed database (perhaps they rolled their own).

- _Knowing that Midokura's exit plan likely involves being bought out_

I would imagine they will eventually be bought by a large conglomerate like Cisco. One of my other favorite tools, Ansible, was recently purchased by RedHat. Sometimes the exit turns out well, sometimes not.

## Issues I encountered

- _Memory resource sizing for midolman_

The Midolman agent can use a fair amount of memory. The package comes with some example sizing configurations. It is definitely a good idea to use the correct configuration file for the particular service and size of server that is being deployed, then make sure Nova knows to hold back that memory on compute nodes. This is for the deployer to do.

- _Clean up Cassandra_

Eventually the Cassandra database can become large and needs to be cleaned up.

- _Not being able to keep up with releases_

I believe Enterprise Midonet is in a 1.9 release now. It's not easy to keep up, and doing so would be something to take into consideration. Like OpenStack itself, you will want to upgrade as frequently as you can, which in the end...is never enough. Again, this is a deployer issue not a Midonet issue. New code is a good thing and we need to get it deployed.

## Related things I have not (yet) worked with

- VXLAN off-loading
- VTEP integration
- vhost-net driver (should increase performance)

## Conclusion

I chose to deploy Midonet based on several assumptions and project requirements. In the end, Midonet works as advertised, and, I feel, makes my life as an OpenStack operator easier. I definitely recommend Midonet for OpenStack deployments, or anyone who is interested in software defined networking.
