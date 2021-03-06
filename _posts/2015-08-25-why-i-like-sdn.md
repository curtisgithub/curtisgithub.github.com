---
layout: post
title: Why I like SDN
categories:
header_image: /img/knot.jpg
header_permalink: https://www.flickr.com/photos/gabepopa/14605341837/in/photolist-ofCchz-t39Ge-hnmAWV-8bhrRf-3RUaK-bmann8-jkeT6Z-DNgt9-6iPNaD-qqoaRz-dkAUPQ-kXv9FR-bman9Z-EZD6u-8NcrZZ-9yV5Vj-tp4Gqs-3qBxZY-8RGfyh-ozeFw3-5iVV28-nTPSMb-6ak4Di-4eZQd6-5wzw9G-bmajPn-4AmW5Q-bmam2V-cYp6kG-9SWamu-9j8aDz-71uENs-6VFVAJ-7hixgR-3qBzLN-9tvdoA-rAdDmh-4dFSg9-8U3j8o-cTQ4sQ-6aQSPb-6Cngxs-8ohbAi-EJxzR-8ysKnF-AiH5P-o3vpnh-bmajeD-Aa4VP-86kLRf
---

# {{ page.title }}

Recently, I helped give a presentation on SDN to the [Edmonton Go](https://edmontongo.org/) meetup. Basically my part of the presentation was to talk about what SDN is and how it relates to containers and Go. That is actually easy because SDN, containers, and Go are the parts of a "so hot right now" trifecta. There's a lot of money in those three things, perhaps more when together. I did joke that I hoped we'd come out of the presentation with a fully funded startup. Alas that did not happpen, as we are in Edmonton, not Palo Alto.

The only startup I'm aware of that did all three, Socketplane, was bought by Docker before they even put out a real product. I'm surprised there aren't more Go-based SDN startups. Maybe there are and I'm just not aware of them. Maybe they all get bought before they get a twitter account. [1]

## I use SDN every day

As a systems administrator, I tend to concern myself with applying technologies in production, which is to say that theory is not enough. I use Midokura's Midonet (which is open source[2]) to provide multi-tenant virtual private networks for openstack users. It's in production and has been for months. Midonet acts as a plugin to Neutron and centralizes state in a Zookeeper cluster. Everything's been working great. My point is that I am using a software defined network every day to meet customers needs, and I think it would be difficult to do it without some kind of SDN system.

There are quite a few "plugins" (or perhaps "network models" would be a better term) for OpenStack that could be considered SDN. If you don't like overlay or encapsulation models and prefer a large layer 3 design, perhaps [Project Calico](http://www.projectcalico.org/) would be of interest. Also the neutron team and some large OpenStack deployers are working on a large layer 3 design.

## Some thoughts on SDN 

First off, SDN is "an abstract concept which can mean many things." The term of overloaded. 

For a while OpenFlow defined SDN, but that's no longer the case as it seems the market has outrun OpenFlow. I would think that at the very least an SDN system would have to provide an API. After that it gets harder to define.

### What's great about SDN?

- Making the network programmable
- Reducing capex and opex
- Enabling multi-tenancy and massive networks
- Avoid some limitations (eg. > 4096 VLANs, which, honestly, is still a lot of VLANs)
- Providing a bit of a shakeup to networking (yes, the Internet works amazingly well in terms of layer 3 routing, but is it the ultimate networking model?[3])

### What's great about containers and SDN?

Docker has been around for a couple years now, but the container "space" is still on fire. Every day there are new announcements. Even if you don't like containers, you have to admit there is a lot of work being done, a lot of innovation happening and that innovation is "trickling down" to other areas, such as networking. Maybe all the "new" ideas existed before. At any rate I think Linux namespaces and cgroups have been a catalyst for change, perhaps under the guise of Docker (and now the [Open Container Initiative](https://www.opencontainers.org/)). These systems need network information, and screen-scraping or grepping dhcp.leases isn't going to do it.

The canonical example of containers and SDN is how easy it is to setup one Docker host, and how much tricker it is to setup two docker hosts with networking between containers.

Another example I like is if you have many container hosts with a large proxy layer on top, and you are recreating containers like crazy, how does that proxy layer know the network address of all the containers and what they do? Through an API provided by SDN.

I believe the model used by most container schedulers is one IP per container. Given that containers don't contain, multi-tenant container systems aren't used in production (yet) which means a flat layer 3 model works well. But at some point I think multi-tenant containers will be commonplace and then it will make more sense to apply other networking models, just like we do with vms.

### Network ops pushback

I get a lot of pushback on SDN. Other than being interested in SDN, enough knowledge to get servers up and running, and messing around with BGP and OSPF for high availability, I'm not a networking expert. No one has ever let me login to a switch at work (they defend those things with their lives). And yet here I am, managing a rather large (virtual, software defined) network inside an OpenStack cloud. I'm looking at change in a few areas, 1) SDN on the WAN 2) Using whitebox switches and Cumulus Linux and 3) getting rid of virtual IPs by using BPG and OSPF. The first two are tough battles, only #3 seems doable in my current environment.

### Centralized control plane?

> We’ve had centralized architectures for decades, from SNA to various WAN technologies (SDH/SONET, Frame Relay and ATM). They all share a common problem: when the network partitions, the nodes cut off from the central intelligence stop functioning (in SNA case) or remain in a frozen state (WAN technologies). [4]

I'm still slightly confused on the centralized control plane. The above quote makes it fairly clear that a centralized control plane is counter-productive when thinking about scale-out systems.

## Conclusion

So, while there are many popular new technologies (some would say "over-hyped," though I would not) I think over time SDN will prove invaluable, and the ability to program a network will slowly seep into almost every vendors system, and will dominate open source networking solutions. Eventually SDN will just be the way networks are done. Certainly that could take years, even decades (as IPv6 hasn't even been accepted yet, and we really need it), but I would feel safe making a long-term bet on SDN and related technologies.

<hr >

1: It's not like it really matters whether it's Java, C, Go, or some other language. I just happen to like Go.

2: Midokura has open sourced their application. But it is definitely more like open core. There aren't a lot of non-Midokura contributors. But again, networking gear isn't free either.

3: I'm really asking. Is it?

4: [Does Centralized Control Plane Make Sense?](http://blog.ipspace.net/2014/05/does-centralized-control-plane-make.html)
