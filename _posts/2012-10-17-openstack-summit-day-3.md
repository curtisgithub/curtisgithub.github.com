---
layout: post
title: OpenStack 2012 Summit Day &#35;3
categories:
---

# {{ page.title }}

## Keynotes

HP says something enterprise business-like. Not too sure. 100s of billions of "spend" available. Capex -> Opex. Many, many slides with business pictographs.

[Troy Toman](https://twitter.com/troytoman) from Rackspace talks about how proud they are of OpenStack. Brags about how Rackspace's contribution percentage has actually reduced from 54% in Essex and 30% in Folsom and how that is a good thing, which it is. :)

Notes from Rackspace's production usage of OpenStack:
- They have Quantum and [Melange](http://wiki.openstack.org/Melange) in production
- Glance is backed by Swift
- They have deployed the Cells code; at least three cells in each regon
- Three regions worldwide
- All of the "control" pieces, eg. API servers, are running on a private OpenStack cloud; an internal infrastructure called _inova_
- Continuous delivery model for building and deploying to cloud--they pull from trunk at least once a day, and do this in under and hour, and have been deploying this once a week or so.
- Fail fast fix fast
- Coming soon: block storage and network products
- Releasing PHP and Java cloud SDKs

Cisco Webex talks about what they do. Lots of open source technologies...puppet, salt, cobbler, etc to run a private cloud so they have "one throat to choke".

## Service Resiliency Doesn't Always mean "HA" or "Cluster"

The guys from CloudScaling talk about redundancy patterns. This was actually pretty fascinating, and looks like they have taken care of a lot of scale-out redundancy, specifically not "HA-mmer" pairs, all the way up to the DB level, at which part they fall back to MMR MySQL.

# "HA" pairs are not the only type of redundancy
- Airplanes require seven catastrophic failures to fall out of the sky, whereas with HA pairs you just need one layer to fail
- Create small failure domains that don't propagate = scale up
# Alternative redundancy patterns
# Redundancy patterns in Open Source Cloud Software

CloudScaling have come up with a methodology for "HA" they call Service Distribution.

Service Distribution
- Resilient
- Stateless
- Scale-out
- Implementation
- OSPF (quagga), Anycast (zebra), Load-balancing proxy (pound)
- Perfect for site resiliency
- Traditional HA pairs don't support this
- Use ZeroMQ (peer-to-peer) vs RabittMQ and CloudScaling contributed the code back to OpenStack for ZeroMQ
- Making data centers look like ISP backbones
<p></p>

## CirrOS, cloud-init: the future of cloud guests

I think the dev sessions are really about starting conversations that will be finished elsewhere, and if you aren't fairly deep into the topic there's not a lot of ways to get your head into the session. I've felt the pain of the lack of Cloud-init in Redhat. I'm surprised that there is just one person creating Cirros.

Topics Covered
- Cirros
- Cloud-init
- Config drive 2
- Execute code from metadata in various ways

## Adding OpenVZ support to Nova

Rackspace has code that will eventually make it's way into trunk for using OpenVZ in OpenStack. The session leader, Devananda van der Veen,  has created a way to build an OpenVZ kernel on Ubuntu. Rackspace uses OpenVZ to run their cloud database system because they get better performance than using something like KVM for virtualization.

- [devstack-gate](https://github.com/openstack-ci/devstack-gate)
- [Etherpad for this session](http://etherpad.openstack.org/grizzly-nova-openvz)

Most of the discussion was about how to get the OpenVZ driver into the OpenStack continuous integration system.

## Cloud Hosted Desktops and OpenStack

Ken Ringdahl, Vice President Engineering, Desktone discusses Desktop as a Service. I'm interested in this topic not because I'm interested in the topic, but because the project I'm currently working on is essentially desktop in the cloud via [Apache VCL](https://cwiki.apache.org/VCL/apache-vcl.html) and I'm constantly thinking about how to get off VCL and away from it's 40k lines of perl, and just use OpenStack with a thin layer on top of it. The reality is that OpenStack has 95% of the code required to implement a service similar to VCL.

So Ken goes through his slides. Mentions VDI and that VDI is hard! Capex, buy more hardware, can't fit it in their data center, SAN administrators, etc. VDI is tough. But DaaS is not VDI.

The point of the session is that DaaS is a three billion dollar market and is a new business area that OpenStack can access. Further he talks about the fact that running a desktop resource manager/broker on top of OpenStack can create operational and other cost savings. It also allows and organization like Desktone to focus on user experience versus running infrastructure.

Desktops are Different:

- Storage is key
- Random IO
- Write intensive IO 
- Scale: 500 servers is a lot, 500 desktops is not
- Optimize OpenStack for DaaS
- HA for desktops is different from servers
- The future of VDI is non-persistence...nothing stored in VMs
- # of VMs under management requires a different approach
- Eg. VM and guest state changes
- Monitoring is critically important
- DaaS encompasses all parts of OpenStack
