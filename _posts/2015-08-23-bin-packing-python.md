---
layout: post
title: Bin Packing with Python
categories:
header_image: /img/containers.jpg
header_permalink: https://www.flickr.com/photos/glynlowe/10921733615/in/photolist-hD7JTZ-75mqRx-6gVcao-9F5bop-hD9cR2-3EK4k-9uDMLx-4NY3Yi-75qid1-3h5aAH-5o5H1E-85HX8j-3RUNDA-kL1Hg5-DkXpr-qyxMwu-tMKSiY-rqLLkq-hD7Jqz-5qQuwt-23rDNy-a3NYdi-8ffvhp-75mqLa-jxLp1q-qyxTko-hD7Kji-2UrVHv-gLRa3J-wwvCPe-4MtokT-5o5H31-8ffvuz-7HrALP-fMsCgA-5MQQVH-bRf2oP-7vBa2x-obgQZx-bMcSap-6W7SPZ-3FjPcz-4LGK2-gmkcM-s36SPn-oqxZaJ-hD812L-a85qp6-kKZgdV-s36UQM
---

# {{ page.title }}

I've been working with OpenStack for a while now, since back at the Essex release, and every once and a while I hear the phrase "bin packing" with regards to scheduling vms on physical hosts. I didn't study computer science in university, and am not particularly interested in that kind of thing, but at work we have also been discussing how to deploy racks of servers, or what size of physical hosts to buy (ie. how much memory, disk, cpu to put in them). 

As far as server racks go, we are constrained on power, space, and possibly network ports. Given those constraints, how can we best deploy servers and network gear? Then, with regards to vms, if we have several types of openstack flavors, how does that work into what we can support and what the distribution on the physical hosts will look like?

For the most part I believe people just do whatever in terms of hypervisor host sizing. If deploying openstack they just buy the same hosts as they have always bought for virtualization. But I wanted to 1) take planning a step further and at least be able to calculate requirements in some fashion and 2) finally figure out what bin packing is.

## What is bin-packing?

> In the bin packing problem, objects of different volumes must be packed into a finite number of bins or containers each of volume V in a way that minimizes the number of bins used. In computational complexity theory, it is a combinatorial NP-hard problem. [1]

Basically, if you have a bunch of different sized objects and bins to put them in, how do they best fit? I would imagine most compsci grads saw this problem and related problem in early courses.

The other interesting thing is that it is NP-hard. This is another term that I've heard quite often but haven't researched what it is. Until now. :)

>  What does it mean to be NP-hard? It means that if you can solve an NP-hard problem in polynomial time, then you can solve all the NP problems in polynomial time. [2]

From my layman point of view, it means that bin packing is a difficult problem to solve in that it can take a long time to get an answer, and what's more, that if we could solve it quickly then we could solve a lot of other things quickly too. Because it can take a long time to find the answer, often bin packing algorithms take shortcuts to provide an answer quickly.

## Bin packing with python

I did a lot of googling to find examples of bin packing with python. The first good one I came across was [pyShipping](https://github.com/hudora/pyShipping) which has a couple of examples of bin packing and 3d bin packing. But after looking through that a bit I realized that I was looking for bin packing with multiple constraints. 

Finally I ended up on one of the oldest looking websites I've seen in a long time, [OpenOpt](http://openopt.org). Yes, that "coming soon" gif is blinking.

![](/img/openopt.jpg)

Anyways, OpenOpt has a couple nice examples and there is a short page on the site for [bin packing](http://openopt.org/BPP). I based my work on is the [advanced, multiple constaints example](http://trac.openopt.org/openopt/browser/PythonPackages/OpenOpt/openopt/examples/bpp_2.py).

## Laymans bin packing

I've just been doing a bit of testing with the OpenOpt example. Below my example has three flavors (vm sizes): small, medium, and large, each with different cpu, memory, and disk requirements.

Then I have server "bins", ie. hypervisor hosts, with 2TB of disk, ~240GB of memory, and 48 cpus with a 4x overcommit (so 192 virtual cpus). When I run the bpp algo which uses the glpk solver, it distributes the vms over the hypervisors in what we hope is the best use of the resources, with the lowest number of hypervisors being used.

The code:

<pre>
<code>#!/usr/bin/python
from openopt import *

N = 60 

items = []
for i in range(N):
    small_vm = {
        'name': 'small%d' % i,
        'cpu': 2,
        'mem': 2048,
        'disk': 20,
        'n': 1
        }
    med_vm = {
        'name': 'medium%d' % i,
        'cpu': 4,
        'mem': 4096,
        'disk': 40,
        'n': 1
        }
    large_vm = {
        'name': 'large%d' % i,
        'cpu': 8,
        'mem': 8192,
        'disk': 80,
        'n': 1
        }
    items.append(small_vm)
    items.append(med_vm)
    items.append(large_vm)

bins = {
'cpu': 48*4, # 4.0 overcommit with cpu
'mem': 240000, 
'disk': 2000,
}
p = BPP(items, bins, goal = 'min') 
r = p.solve('glpk', iprint = 0) # requires cvxopt and glpk installed, see http://openopt.org/BPP for other solvers
print(r.xf) 
print(r.values) # per each bin
print "total vms is " + str(len(items))
print "servers used is " + str(len(r.xf))

for i,s in enumerate(r.xf):
    print "server " + str(i) + " has " + str(len(s)) + " vms"
</code>
</pre>

The results:

<pre>
<code>$ time python vms.py 
Initialization: Time = 6.7 CPUTime = 6.7

------------------------- OpenOpt 0.5625 -------------------------
problem: unnamed   type: MILP    goal: min
solver: glpk
  iter  objFunVal  log10(maxResidual)  
    0  0.000e+00               0.00 
GLPK Integer Optimizer, v4.54
33480 rows, 32580 columns, 162900 non-zeros
32580 integer variables, none of which are binary
Preprocessing...
720 rows, 32580 columns, 130140 non-zeros
32580 integer variables, all of which are binary
Scaling...
 A: min|aij| =  1.000e+00  max|aij| =  2.400e+05  ratio =  2.400e+05
GM: min|aij| =  9.204e-01  max|aij| =  1.087e+00  ratio =  1.181e+00
EQ: min|aij| =  8.812e-01  max|aij| =  1.000e+00  ratio =  1.135e+00
2N: min|aij| =  5.000e-01  max|aij| =  1.000e+00  ratio =  2.000e+00
Constructing initial basis...
Size of triangular part is 720
Solving LP relaxation...
GLPK Simplex Optimizer, v4.54
720 rows, 32580 columns, 130140 non-zeros
      0: obj =   0.000000000e+00  infeas =  1.706e+02 (0)
    500: obj =   3.010416667e+00  infeas =  4.995e+01 (0)
-   628: obj =   4.375000000e+00  infeas =  1.599e-14 (0)
OPTIMAL LP SOLUTION FOUND
Integer optimization begins...
+   628: mip =     not found yet >=              -inf        (1; 0)
+  6048: >>>>>   5.000000000e+00 >=   5.000000000e+00   0.0% (46; 0)
+  6048: mip =   5.000000000e+00 >=     tree is empty   0.0% (0; 91)
INTEGER OPTIMAL SOLUTION FOUND
    1  0.000e+00            -100.00 
istop: 1000 (optimal)
Solver:   Time Elapsed = 6.86   CPU Time Elapsed = 6.85
objFuncValue: 5 (feasible, MaxResidual = 0)
[{'medium12': 1, 'medium13': 1, 'medium10': 1, 'medium11': 1, 'small9': 1, 'medium18': 1, 'medium57': 1, 'medium55': 1, 'large8': 1, 'small11': 1, 'small10': 1, 'small12': 1, 'large1': 1, 'large3': 1, 'large2': 1, 'large6': 1, 'large33': 1, 'medium9': 1, 'small55': 1, 'medium0': 1, 'large14': 1, 'medium2': 1, 'large16': 1, 'large11': 1, 'large10': 1, 'large12': 1, 'medium26': 1, 'medium22': 1, 'large57': 1, 'large54': 1, 'small58': 1, 'medium1': 1, 'large56': 1, 'large22': 1, 'large26': 1}, {'medium39': 1, 'medium35': 1, 'medium36': 1, 'medium54': 1, 'medium52': 1, 'medium50': 1, 'medium51': 1, 'medium59': 1, 'small13': 1, 'small51': 1, 'small50': 1, 'small52': 1, 'large37': 1, 'large36': 1, 'large35': 1, 'large34': 1, 'large15': 1, 'large17': 1, 'small35': 1, 'large18': 1, 'medium49': 1, 'medium48': 1, 'large51': 1, 'large50': 1, 'large52': 1, 'medium47': 1, 'medium46': 1, 'large49': 1, 'large46': 1, 'large47': 1, 'large23': 1, 'large48': 1, 'small48': 1, 'small49': 1, 'small47': 1, 'medium7': 1, 'small40': 1, 'small41': 1}, {'medium38': 1, 'medium34': 1, 'small8': 1, 'medium37': 1, 'medium32': 1, 'medium33': 1, 'medium53': 1, 'large9': 1, 'small53': 1, 'large32': 1, 'small39': 1, 'small38': 1, 'small54': 1, 'small33': 1, 'large39': 1, 'large38': 1, 'small37': 1, 'small36': 1, 'medium6': 1, 'small34': 1, 'medium42': 1, 'medium41': 1, 'medium40': 1, 'large53': 1, 'large41': 1, 'medium45': 1, 'medium44': 1, 'medium20': 1, 'medium3': 1, 'small28': 1, 'large21': 1, 'large44': 1, 'large45': 1, 'large42': 1, 'large43': 1, 'large40': 1, 'large27': 1, 'small46': 1, 'small44': 1, 'small45': 1, 'small42': 1, 'small43': 1, 'small27': 1}, {'small1': 1, 'small0': 1, 'small3': 1, 'small2': 1, 'small5': 1, 'small4': 1, 'small7': 1, 'small6': 1, 'large0': 1, 'large5': 1, 'large4': 1, 'large7': 1, 'medium8': 1, 'medium56': 1, 'small59': 1, 'large58': 1, 'medium4': 1, 'medium5': 1}, {'medium16': 1, 'medium17': 1, 'medium14': 1, 'medium15': 1, 'medium19': 1, 'medium30': 1, 'medium31': 1, 'medium58': 1, 'small15': 1, 'small14': 1, 'small17': 1, 'small16': 1, 'small19': 1, 'small18': 1, 'large31': 1, 'large30': 1, 'large19': 1, 'small57': 1, 'small56': 1, 'small32': 1, 'small31': 1, 'small30': 1, 'medium43': 1, 'large13': 1, 'large59': 1, 'medium29': 1, 'medium28': 1, 'medium27': 1, 'large55': 1, 'medium25': 1, 'medium24': 1, 'medium23': 1, 'medium21': 1, 'large28': 1, 'large29': 1, 'large20': 1, 'small29': 1, 'large24': 1, 'large25': 1, 'small20': 1, 'small21': 1, 'small22': 1, 'small23': 1, 'small24': 1, 'small25': 1, 'small26': 1}]
{'mem': (196608.0, 196608.0, 196608.0, 75776.0, 194560.0), 'disk': (1920.0, 1920.0, 1920.0, 740.0, 1900.0), 'cpu': (192.0, 192.0, 192.0, 74.0, 190.0)}
total vms is 180
servers used is 5
server 0 has 35 vms
server 1 has 38 vms
server 2 has 43 vms
server 3 has 18 vms
server 4 has 46 vms

real    0m24.788s
user    0m23.894s
sys 0m0.944s
</code>
</pre>

So from the above, we know we would need five hypervisor hosts to run this set of 180 vms, 60 small, 60 medium, and 60 large.

From the output it also seems like we max out on cpus and are actually getting pretty close on disk too.

<pre>
<code>{
 'mem': (196608.0, 196608.0, 196608.0, 75776.0, 194560.0), 
 'disk': (1920.0, 1920.0, 1920.0, 740.0, 1900.0), 
 'cpu': (192.0, 192.0, 192.0, 74.0, 190.0)
}
</code>
</pre>

Now we can play around with numbers and see if we can reduce the number of hypervisors. In the next attempt I changed the cpu overcommit to 5 and the disk on the host to 3TB.

<pre>
<code>{
 'mem': (184320.0, 198656.0, 239616.0, 237568.0), 
 'disk': (1800.0, 1940.0, 2340.0, 2320.0), 
 'cpu': (180.0, 194.0, 234.0, 232.0)
}
total vms is 180
servers used is 4
server 0 has 37 vms
server 1 has 39 vms
server 2 has 55 vms
server 3 has 49 vms
</code>
</pre>

Now we are down to only four hypervisors fitting the 180 vms.

Certainly this is the most simplistic example. But I feel like it's still pretty powerful, and it will make much more sense when I can input real-world data for flavor usage in a cloud. If I can input real usage data in terms of flavors, instead of just 60 of each, then I can really start to understand what the minimum hardware investment is, or at least what the servers could look like for memory, disk, and cpu.

## Live migration

I've never been a big fan of live migration. Usually it means a distributed file system, or a shared file system, and those are both hard to run and scale. Also, there is the whole pets vs cattle thing. Using that metaphor, there's no reason to live migrate cattle.

However, I did read through a paper, "Adaptive Resource Provisioning for the Cloud
Using Online Bin Packing":http://www.cs.princeton.edu/~haipengl/papers/binpacking.pdf, that discussed using bin-packing to move vms around in a cloud so that some hypervisors could be turned off and thus save power. To me that sounds like a good use of live migration.

We have talked about making booting from volumes a default as potential methodology in an OpenStack cloud, which would enable live migration without a large shared file system or a distributed file system. So that could be a direction to go in. I've always thought of cinder with plain old lvm hosts to be pretty powerful and having a small failure domain. In theory we could let OpenStack schedule however it wants, and then run a bin packing algorithm every once and a while and live migrate instances to make better use of hypervisors. Easier said than done of course, but would be interesting.

## Conclusion

In the end, despite being a compsci layman, I think I've got what I wanted: a calculator that can help me to look at our rack resources and our hypervisor servers and do some sizing. As I continue down this path I'm sure I'll learn more about bin packing. It's a start. :)

The next area I need to understand is how OpenStack schedules vms...

<hr />

1: [Bin Packing Problem](https://en.wikipedia.org/wiki/Bin_packing_problem)

2: [Explanation of P versus NP](https://www.quora.com/What-is-an-explanation-of-P-versus-NP-problems-and-other-related-terms-in-laymans-terms)

