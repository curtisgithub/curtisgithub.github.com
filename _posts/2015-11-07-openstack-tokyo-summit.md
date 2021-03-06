---
layout: post
title: OpenStack Tokyo Summit 
categories:
header_image: /img/tokyo.jpg
header_permalink: https://www.flickr.com/photos/gullevek/8583984199/
---

# {{ page.title }}

I was lucky enough to be able to attend the OpenStack Tokyo Summit, which of course, is in Tokyo. Weirdly enough it is quite easy to get to Tokyo from Edmonton--all I had to do was the short flight from Edmonton to Vancouver, up and over the Rockies, and then from there take All Nippon Air (ANA) to Tokyo. I flew on on a nice, new 787 complete with USB connections. (Aside: ANA is the best airline I've flown on.) 

I think it was cheaper, just flight and hotel, to send me to Tokyo for a week than it was for the Palo Alto operators meetup for three days. Tokyo is not that expensive for a major city, and Palo Alto bloody well is (for a minor one).

## The Summit

The summit took place at the Grand Prince International Convention center. This was a collection of buildings, more like a campus, that the summit spread out over. I've never looked at the same map so many times. I missed the start of a couple of sessions because I got a bit lost. The Vancouver convention center was much easier to navigate, but it's also much newer, so not a fair comparison.

For the last few summits I've been to I spend most, if not all of my time in the design summit going to operator sessions. Which makes sense because I am a OpenStack operator. Surprisingly I am a real person, not some mythical beast. :) I have to wrangle this massive OpenStack conglomerate into a working, highly available system. It's not easy, but it's not impossible either.

Here are some of the sessions and talks I went to:

- Ops - Quotas and Billing (which I moderated as best I could)
- Ops - How to work with the community on a small budget
- Cross project workshop - Performance
- Talk - Kuryr: Docker networking in an OpenStack World
- Ops - Upgrades
- Ops working session - Large Deployments Team
- Talk - How to reboot the cloud (in the face of a Xen vulnerability)
- Magnum - Networking past, present, and future (related to Kuryr)
- Ansible collaboration day - Most sessions
- Talk - Real world devops experience securing a cloud (this was not a great presentation unfortunately)
- Security - How should security serve the community? (very few in attendance, mostly for security team members, which I am not)
- Talk - CloudKitty

All of the sessions were great, if short and either really busy or not busy at all. There were some sessions where it was so busy I couldn't even get into the room and had to go watch something else. Basically I learned my lesson on getting there early. A couple of the Ansible collaboration day sessions and the informal operators meetup on the Friday were both so busy I couldn't get in or get a seat because I showed up a bit late.

Primarily it seemed that at this summit I was interested in billing and networking. I am not a networking expert, but I certainly have to know a lot about it, especially with Neutron. I almost want to run a few clouds to try out different networking models, like Midokura, DragonFlow, and Calico. One thing I noticed is how there are fewer security sessions now. OpenStack security has moved to primarily supplying tools like Bandit and security notes, and not much more than that.

I can definitely say that Ansible's popularity has expanded considerably compared to the last summit, which was only six months ago. Here's hoping the Redhat purchase goes well. OpenStack infra uses Ansible as well and the new [Shade](http://docs.openstack.org/infra/shade/) library is an important piece of the puzzle. The OpenStack Ansible project (OSAD) also continues to move along, almost too fast for me to keep up. Well, not almost--I can't keep up with the improvements just due to lack of time. If you want to learn about how to use Ansible that is the best project to look at in terms of actually writing playbooks and modules and callbacks.

Billing is also interesting because it's heavily tied into metrics, ie. Ceilometer. CloudKitty is fascinating and I'm sure I will be implementing it soon. Perhaps that is something we can contribute back to.

## Regional Clouds

Probably the best session I had was at the LDT meeting where we talked a bit about public clouds, specifically what I call "regional" public clouds, which are typically clouds meant for use in a particular country or geographical area, eg. Canada or New Zealand or Europe. Many people don't know this, but, for example, in Canada there are many companies that want to use IaaS but don't want their data in the US. It's like some kind of data-protectionism and it happens in many countries, which leads to regional clouds.

These clouds are usually smaller than other clouds, but have to be more featureful--or "complete" was another word that was used. I think it's harder to run a smaller cloud with more features, as opposed to a larger cloud that has less features, especially around the networking component. 

Weirdly it's somewhat lonely working in a regional cloud. Typically we (as in regional clouds) don't have a large staff, nor a large budget to send people to meetups and summits. Further we don't have the resources to contribute back much because we're so busy (like all operators generally speaking), and, what's more, because of the smaller scale we don't typically run into common larger issues like running cells or applying custom patches to run a simpler network setup. Instead we run a lot of features such as LBaaS and have to do real billing. Customers want features like backups and DNS and containers and VPNaaS, etc, etc. Larger OpenStack public clouds typically don't provide those services, but we have to. Oh, and AWS compatibility, that's a big one too. Big clouds do less but have more vms running, regional clouds do more but have less vms.

I'm hoping that various regional clouds can start working together in some fashion, even if it only means sharing operational aspects, or perhaps even doing a exchange. I'm not sure if the OpenStack foundation is interested in helping regional clouds, but we'll do our best anyways.

## Japan/Tokyo

I stayed a few days longer in Tokyo. I didn't leave the city except for a short trip out to Hakone (which kind of felt like Japan's Jasper). I visited most of the museums, including the incredible 500 Arhats exhibit at the Mori. I also visited all the areas such as Akihabara and "Kitchen Town." Mostly I rode the subway a lot and then caught an awful cold. The public transport system is massive, but by the end I had it kinda figured out. I think I visited every vegetarian restaurant in Tokyo. For a city of 13 million plus, there is not much for vegetarian options. Edmonton probably has twice the vegetarian restaurants that Tokyo does. Strange.

My trip was a bit up and down--at first I was like "Wow! Look at all these people!" then after a while it became overwhelming. I spent a lot of mental energy just walking around trying not to bump into people and remembering to walk on the left side. But then after a couple more days I came back around to enjoying it. I had one poor experience, which I won't mention other than to say I had one poor experience.

My favorite morning activity was to grab a coffee at the Starbucks in the train station (yeah, Starbucks, but the location was perfect for people watching) and sit and watch all the commuters on their way to work. The number of men wearing a black suit and a white shirt with no tie was staggering. I tried counting for a while but after ten or twenty seconds I couldn't keep up. Literally thousands would pass in an hour. Sometimes the cadence of their footsteps would all fall in line and echo throughout the station. Very surreal.

Finally it was time to come home. In the end I packed four bottles of (what I hope is) premium sake--which the nice Canadian border guard let me in without having to pay duty on! She sure asked me a lot of questions but then didn't care about the sake.

Overall I had a great first trip to Japan, and now that I've been there once I think I have a good idea of what I'd like to do the next time I go. I found the flight to Tokyo to be fairly easy, but the jet lag was a killer. I'm still not sleeping 100% correctly. But only having one stop in Vancouver is convenient.

I'm looking forward to the next operators meetup which should be relatively soon, and also, I'm guessing, may not be in North America.
