---
layout: post
title: OpenStack 2012 Summit Day &#35;2
categories:
---

# {{ page.title }}

## Keynotes

The keynotes mostly focused on the increased sized of the OpenStack community and the creation of the foundation that now runs OpenStack. Except Shuttleworth's in which he used Juju to live upgrade from Essex to Folsom which turned some heads. It's the first time I've heard him speak, and he seemed exactly like the kind of person that can sit in-between business and open source. Chris Kemp was good, and took a couple of shots at VMWare which I always think is fun. ;)

## High Availability

Florian Haas of [Hastexo](http://www.hastexo.com/) presented an update on high-availability with OpenStack. The company I currently work for has, in the past, contracted Florian/Hastexo to consult on OpenStack deployments and I like the things that he says. He's the go to person for HA + OpenStack.

He talked at length about [Pacemaker](http://clusterlabs.org/wiki/Pacemaker) which is a cluster resource manager. Interestingly he notes that it runs air traffic control systems. Also he mentions that it is extremely friendly to 3rd party functionality via plugins/resource agents,

He also went through the OpenStack High Availability Guide which he was largely responsible for creating, and mentioned that it is in [source control](https://github.com/openstack/openstack-manuals/tree/master/doc/src/docbkx/openstack-ha) and thus people can contribute patches.

The slides for this talk are [online](http://www.hastexo.com//resources/presentations/high-availability-update).

## Frameworks and APIs for Advanced Service Insertion

This was a developer session on the future methodologies of inserting things like firewalls, load balancers and such into Quantum. It was interesting to hear some of the thoughts on the topologies of Quantum and how OpenStack is trying not just to replicate existing networking technologies, but create something new. It was nice to hear the word router with an Italian accent. OpenStack is a global project.

## Lunch

Not much to say, on this. There was tortilla soup though. :)

## The Future of Infrastructure Automation

This session was not what I thought it was going to be. I didn't realize it was part of the strategy track until I was sitting down listening, and so before knowing that I figured it would be about what comes after puppet/chef/etc...which it was not.

I don't have a lot to say about this session, other than the speaker liked [webhooks](http://www.webhooks.org/) and push notifications, that it would be nice to for a guest in OpenStack to be able to write metadata instead of just reading, and that some organizations are working on this functionality. I definitely would like to be able to write metadata from guests so that was good to hear.

## Surviving your first check-in: An engineers guide to contributing to OpenStack

This was a great session on lessons learned by an engineer (specifically not a developer), [Collin McNamera](http://www.colinmcnamara.com/), who wanted to contribute code to OpenStack. To get his code into OpenStack he had a ratio of 100:1 in terms of time spent figuring out how to contribute and waiting for answers vs actually coding. That was for his first contribution so the ratio is better now, but it was a tough slog at first.

Lessons learned:
- Join, or start, a local meetup group
- Make sure that according to your employer you can contribute code to OpenSource projects
- Execute your OpenStack CLA (contributor license agreement)
- Setup your dev environment, probably using virtual machines
- [Devstack](http://devstack.org/)
- With devstack you can point it to a different OpenStack git repo if you want
- Use vm snapshotting to help you keep using your dev environment
- Configure git with your name, email, etc and this will save time in the future. Do it right first!
- Install git-review
- Clone a project, or use the directories that came with Devstack
- Add your ssh key to OpenStack Code Review
- Create a topic branch
- Change code
- Test code: run_tests.sh
- Commit changes - make sure to put a bug id or blueprint # in the first line
- Then give back to the community by teaching others! :)

While it's disappointing to hear how hard it is to contribute code, Collin was a great speaker is obviously passionate about doing good open source work and contributing back to the community.

## Swift Project Update

As I mentioned in my [previous post](http://serverascode.com/2012/10/15/openstack-summit-day-1.html) I am a big fan of object storage, so Swift is an important project to me. :)

- New features in the last six months
- Folsom version of Swift is 1.7.4
- Unique-as-possible - Allows more flexible growth, and more harddrives as hand-off nodes
- Deep statsd integration - Enables notifications that things like graphite can graph
- SSD optimization - It turns out that in large clusters the account and container servers can become limited by IOPs.
- Versioned writes - This feature is "almost" there
- Moved swift client into it's own project

<p></p>

- Code golf for the last six months
- 37 people have contributed
- 20 have provided their first patch
- Three new core developers
- 170 total commits
- 17 is the most files touched by a single commit (statsd integration)
- 3466 is the most lines removed in a single patch (moving swift-client out to a new project)

<p></p>

- What's next?
- Global clusters - ie. Geographic replication, will hopefully be in Grizzly
- CORS Support - Better integrate with the browser security model
- Optimize concurrent reads 

![](https://raw.github.com/ccollicutt/ccollicutt.github.com/master/img/nebula_pool.jpg)
_(Nebula logo projected on the pool at the club)_

## Parties

This night I went to both the Rackspace and Nebula parties, though I came back to the hotel pretty early. At the Rackspace party I learned that Miramar, the location of the flight school in Top Gun, is only a few minutes away and actually watched some military jets maneuvering out over the ocean. Surprised no one sang _You've Lost That Loving Feeling_. ;)

![](https://raw.github.com/ccollicutt/ccollicutt.github.com/master/img/topgun.jpg)
