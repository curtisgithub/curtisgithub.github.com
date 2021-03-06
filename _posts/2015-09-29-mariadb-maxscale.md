---
layout: post
title: MariaDB MaxScale Read and Write Splitting 
categories:
header_image: /img/ms.jpg
header_permalink: https://www.flickr.com/photos/hellocatfood/10545396704/in/photolist-h4RV3y-6ZY5Yw-7R6usE-5BK5qi-ddn5G7-5hrCbv-da8jMn-ddn815-4nWCrn-98rFiH-gbA7WS-kwvy4z-kwvVNz-kwxBw1-amkoeY-fkKdFR-p1mpRv-pVKYKn-fY9CgE-92kp1F-7PEwp1-ddn5s6-8eKo29-c2nrdA-nKuAaC-h4ST6z-9U1s1d-5aN4gx-hyq3Zy-dRuMzt-8LCNW2-6EHcWT-2Wa9no-fS7qGg-egsmCb-dYwDe2-4mo6sG-7Xk9LM-dL8cys-7mfH7s-iuSuaX-hypmYn-uqFgjs-3rdsAL-dg6Pv5-7ULkET-reqYZ7-r62QE-4tWxze-fBUhie
---

_(the image is supposed to be like that ;P )_

# {{ page.title }}

Maxscale is a new open source product from MariaDB. It's a MySQL/MariaDB proxy and loadbalancer, and, what's most interesting to me, is that it can send write queries to one particular node of a cluster and reads to other nodes, thereby avoiding some nastiness in terms of writing to multiple nodes. There are some caveats to writing to multiple masters and not every application can.

> MariaDB MaxScale is an open-source, database-centric proxy that works with MariaDB Enterprise, MariaDB Enterprise Cluster, MariaDB 5.5, MariaDB 10 and Oracle MySQL®. It’s pluggable architecture is designed to increase flexibility and aid customization.

## Read Write Split

It's fairly common to want to try to split reads and writes up using some kind of proxy system. I'm not up to date on the history of the requirement, but it has been around for a long time. There are, I believe, a few systems that try to do the RW split, but I'm not sure anything has really succeeded. 

As an OpenStack operator I'm keenly aware that I need a highly available MySQL/MariaDB Galera cluster, but also that I can only, as far as I know at this time, write to one of the cluster nodes at a time. If I write to all the nodes using a master/master strategy I'll run into issues. This is better laid out by [this](https://www.percona.com/blog/2014/09/11/openstack-users-shed-light-on-percona-xtradb-cluster-deadlock-issues) Percona post. Some parts of OpenStack use a "SELECT ... FOR  UPDATE":

> The SELECT … FOR UPDATE construct reads the given records in InnoDB, and locks the rows that are read from the index the query used, not only the rows that it returns. Given how write set replication works, the row locks of SELECT … FOR UPDATE are not replicated. -- From the above Percona post

But, using some kind of load balancing system, eg. haproxy, so that I only write to one cluster node also means that I only read from one cluster node as well. That's where MaxScale's ability to split read and writes comes into play.

I should note there is a [good post](http://www.severalnines.com/blog/deploy-and-configure-maxscale-sql-load-balancing) on SeveralNine's site that has some good information in it on MaxScale.

## Install MaxScale

I created an account on MariaDB's site. MaxScale is, I believe, [GPL2](https://github.com/mariadb-corporation/MaxScale/blob/master/LICENSE). But in order to grab the deb file from MariaDB you need to login and get your own unique repository. Once I did that I configured the repository and now MaxScale is available to install via apt. You could always download the source and compile on your own if you would like.

<pre>
<code>ubuntu@maxscale-1:~$ apt-cache policy maxscale
maxscale:
  Installed: 1.2.0
  Candidate: 1.2.0
  Version table:
 *** 1.2.0 0
       1000 http://downloads.mariadb.com/enterprise/<unique id>/mariadb-maxscale/latest/ubuntu/ trusty/main amd64 Packages
        100 /var/lib/dpkg/status
</code>
</pre>

## Configuration

Surprisingly MaxScale is pretty straight forward to configure for read/write splitting. 

This is what my config file looks like (just in my testing phase, so this is not in production at all):

<pre>
<code>[maxscale]
threads=4

[Splitter Service]
type=service
router=readwritesplit
servers=dbserv1,dbserv2,dbserv3
user=maxscale
passwd=7313125C85ABDFB93A4CE397FC2B198D
max_slave_connections=100%
router_options=slave_selection_criteria=LEAST_CURRENT_OPERATIONS

[Splitter Listener]
type=listener
service=Splitter Service
protocol=MySQLClient
port=3306
socket=/tmp/ClusterMaster

[dbserv1]
type=server
address=192.168.44.32
port=3306
protocol=MySQLBackend

[dbserv2]
type=server
address=192.168.44.33
port=3306
protocol=MySQLBackend

[dbserv3]
type=server
address=192.168.44.34
port=3306
protocol=MySQLBackend

[Galera Monitor]
type=monitor
module=galeramon
diable_master_failback=1
servers=dbserv1, dbserv2, dbserv3
user=maxscale
passwd=7313125C85ABDFB93A4CE397FC2B198D

[CLI]
type=service
router=cli

[CLI Listener]
type=listener
service=CLI
protocol=maxscaled
address=localhost
port=6603
</code>
</pre>

As you can see each of the three nodes in my cluster is listed in the configuration file: dbserv1, dbserv2, and dbserv3.

I like the config file. While I hand-rolled this one, the format certainly lends itself to automation.

Now maxscale can be started.

## maxpasswd and maxkeys

I should mention that the "password" in the configuration file above was creating using a combination of maxpasswd and maxkeys. I believe you can just enter the plaintext password into the config file if you want to avoid that extra step.

## Start maxscale

If you used the deb to install, then it comes with the init scripts. Just use "service maxscale start" and it should start up.

<pre>
<code>ubuntu@maxscale-1:/etc$ sudo service maxscale status
 * Checking MaxScale
 * maxscale is running
</code>
</pre>

## maxadmin

MaxScale has a command line interface to MaxScale.

<pre>
<code>ubuntu@maxscale-1:/etc$ maxadmin -pmariadb list servers
Servers.
-------------------+-----------------+-------+-------------+--------------------
Server             | Address         | Port  | Connections | Status              
-------------------+-----------------+-------+-------------+--------------------
dbserv1            | 192.168.44.32   |  3306 |           5 | Slave, Synced, Running
dbserv2            | 192.168.44.33   |  3306 |          83 | Master, Synced, Running
dbserv3            | 192.168.44.34   |  3306 |          89 | Slave, Synced, Running
-------------------+-----------------+-------+-------------+--------------------
</code>
</pre>

As we can see above, MaxScale has made dbserv2 "the master" which is a MaxScale only construction (ie. has nothing really to do with the cluster itself, rather it'll send writes to that node and reads to the other, assuming it's setup in a RW split configuration).

We can list the services configured.

<pre>
<code>ubuntu@maxscale-1:/etc$ maxadmin -pmariadb list services
Services.
--------------------------+----------------------+--------+---------------
Service Name              | Router Module        | #Users | Total Sessions
--------------------------+----------------------+--------+---------------
Splitter Service          | readwritesplit       |      2 | 54357
CLI                       | cli                  |      2 |    68
--------------------------+----------------------+--------+---------------

</code>
</pre>

We can also check stats on the splitter service.

<pre>
<code>ubuntu@maxscale-1:/etc$ maxadmin -pmariadb show service "Splitter Service"
Service 0x35bc5e0
    Service:                Splitter Service
    Router:                 readwritesplit (0x7f0ad8ace4a0)
    State:                  Started
    Number of router sessions:              54339
    Current no. of router sessions:         0
    Number of queries forwarded:            4052477
    Number of queries forwarded to master:  2704498
    Number of queries forwarded to slave:   1347979
    Number of queries forwarded to all:     55274
    Started:                Sun Sep 27 21:54:32 2015
    Root user access:           Disabled
    Backend databases
        192.168.44.34:3306  Protocol: MySQLBackend
        192.168.44.33:3306  Protocol: MySQLBackend
        192.168.44.32:3306  Protocol: MySQLBackend
    Users data:                 0x35168d0
    Total connections:          54357
    Currently connected:            2
    SSL:    Disabled
</code>
</pre>

While running a test we can check sessions too.

<pre>
<code>ubuntu@maxscale-1:/etc$ maxadmin -pmariadb list sessions
Sessions.
-----------------+-----------------+----------------+--------------------------
Session          | Client          | Service        | State
-----------------+-----------------+----------------+--------------------------
0x38819f0        | 127.0.0.1       | CLI            | Session ready for routing
0x7f0ab4044620   | 192.168.44.35   | Splitter Service | Session ready for routing
0x7f0ab4063280   | 192.168.44.35   | Splitter Service | Session ready for routing
0x7f0ab4001800   | 192.168.44.35   | Splitter Service | Session ready for routing
0x7f0ab403a630   | 192.168.44.35   | Splitter Service | Session ready for routing
0x7f0ab4029110   | 192.168.44.35   | Splitter Service | Session ready for routing
0x7f0ab00c60d0   | 192.168.44.35   | Splitter Service | Session ready for routing
0x7f0ab00c73a0   | 192.168.44.35   | Splitter Service | Session ready for routing
0x7f0ab4017710   | 192.168.44.35   | Splitter Service | Session ready for routing
0x7f0ab4090310   | 192.168.44.35   | Splitter Service | Session ready for routing
0x7f0aa80113e0   | 192.168.44.35   | Splitter Service | Session ready for routing
0x7f0aa8034430   | 192.168.44.35   | Splitter Service | Session ready for routing
0x7f0aa8032e30   | 192.168.44.35   | Splitter Service | Session ready for routing
0x7f0aa8032180   | 192.168.44.35   | Splitter Service | Session ready for routing
0x7f0ab405e270   | 192.168.44.35   | Splitter Service | Session ready for routing
0x7f0aa8022f20   | 192.168.44.35   | Splitter Service | Session ready for routing
0x35dfa90        | 192.168.44.35   | Splitter Service | Session ready for routing
0x7f0aa8022790   | 192.168.44.35   | Splitter Service | Session ready for routing
0x35c9070        | 192.168.44.35   | Splitter Service | Session ready for routing
0x7f0ab4000de0   | 192.168.44.35   | Splitter Service | Session ready for routing
0x35c95b0        | 192.168.44.35   | Splitter Service | Session ready for routing
0x35b8b90        | 192.168.44.35   | Splitter Service | Session ready for routing
0x35c7b30        |                 | CLI            | Listener Session
0x35b8960        |                 | Splitter Service | Listener Session
0x35b98f0        |                 | Splitter Service | Listener Session
-----------------+-----------------+----------------+--------------------------
</code>
</pre>

## Stop a cluster node

If I stop MariaDB on one of the cluster nodes, MaxScale knows:

<pre>
<code>ubuntu@maxscale-1:/etc$ maxadmin -pmariadb list servers
Servers.
-------------------+-----------------+-------+-------------+--------------------
Server             | Address         | Port  | Connections | Status              
-------------------+-----------------+-------+-------------+--------------------
dbserv1            | 192.168.44.32   |  3306 |           5 | Slave, Synced, Running
dbserv2            | 192.168.44.33   |  3306 |          83 | Master, Synced, Running
dbserv3            | 192.168.44.34   |  3306 |         108 | Down
-------------------+-----------------+-------+-------------+--------------------
</code>
</pre>

MaxScale will only bring the node back into rotation once it's synced up.

## Test it out

First, I tried using sysbench. However I found that sysbench seems to work in such a way that it will only, even though maxscale, connect to one node of the cluster at time. I'll have to look into this. I was even using the "--oltp-skip-trx=on" option.

In the end I just used a mysqlslap test that is running a simple select from the fake employees database I installed. The 192.168.77.6 IP address is that of the MaxScale node.

<pre>
<code>ubuntu@mysql-client-1:~$ cat mysqlslap.sh 
#!/bin/bash

mysqlslap \
--user=sysbench \
--password=syb3nch \
--host=192.168.77.6 \
--concurrency=20 \
--number-of-queries=1000 \
--create-schema=employees \
--query="/home/ubuntu/select.sql" \
--delimiter=";" \
--verbose \
--iterations=2 \
--debug-info
ubuntu@mysql-client-1:~$ cat select.sql
SELECT * FROM employees;
</code>
</pre>

Here's innotop output on one of the nodes:

<pre>
<code>root@dbsrv1:/home/ubuntu# innotop --count 1 --nonint
cmd mysql_thread_id state   user    hostname    db  time    info
Query   132 Sending data    sysbench    192.168.77.6    employees   00:03   SELECT * FROM employees
Query   135 Sending data    sysbench    192.168.77.6    employees   00:03   SELECT * FROM employees
Query   139 Sending data    sysbench    192.168.77.6    employees   00:03   SELECT * FROM employees
Query   141 Sending data    sysbench    192.168.77.6    employees   00:03   SELECT * FROM employees
Query   130 Writing to net  sysbench    192.168.77.6    employees   00:02   SELECT * FROM employees
Query   136 Sending data    sysbench    192.168.77.6    employees   00:02   SELECT * FROM employees
Query   129 Sending data    sysbench    192.168.77.6    employees   00:01   SELECT * FROM employees
Query   131 Sending data    sysbench    192.168.77.6    employees   00:01   SELECT * FROM employees
Query   133 Sending data    sysbench    192.168.77.6    employees   00:01   SELECT * FROM employees
Query   137 Sending data    sysbench    192.168.77.6    employees   00:01   SELECT * FROM employees
Query   138 Sending data    sysbench    192.168.77.6    employees   00:01   SELECT * FROM employees
Query   134 Sending data    sysbench    192.168.77.6    employees   00:00   SELECT * FROM employees
Query   140 Sending data    sysbench    192.168.77.6    employees   00:00   SELECT * FROM employees
</code>
</pre>

And on the "master" node, which shouldn't be getting any reads:

<pre>
<code>root@dbsrv2:/home/ubuntu# innotop --nonint --count 1
cmd mysql_thread_id state   user    hostname    db  time    info

</code>
</pre>

These are the results of my overly simplistic test using the MaxScale proxy:

<pre>
<code>ubuntu@mysql-client-1:~$ ./mysqlslap.sh 
Benchmark
    Average number of seconds to run all queries: 112.675 seconds
    Minimum number of seconds to run all queries: 111.592 seconds
    Maximum number of seconds to run all queries: 113.759 seconds
    Number of clients running queries: 20
    Average number of queries per client: 50


User time 137.57, System time 69.91
Maximum resident set size 717064, Integral resident set size 0
Non-physical pagefaults 2160414, Physical pagefaults 0, Swaps 0
Blocks in 0 out 0, Messages in 0 out 0, Signals 0
Voluntary context switches 610086, Involuntary context switches 277132
</code>
</pre>

And then without it...going directly to a single node:

<pre>
<code>ubuntu@mysql-client-1:~$ ./mysqlslap.sh 
Benchmark
    Average number of seconds to run all queries: 194.296 seconds
    Minimum number of seconds to run all queries: 193.582 seconds
    Maximum number of seconds to run all queries: 195.011 seconds
    Number of clients running queries: 20
    Average number of queries per client: 50


User time 144.00, System time 71.97
Maximum resident set size 746008, Integral resident set size 0
Non-physical pagefaults 1948959, Physical pagefaults 0, Swaps 0
Blocks in 0 out 0, Messages in 0 out 0, Signals 0
Voluntary context switches 1781510, Involuntary context switches 163916
</code>
</pre>

## Basic introduction

So that was my initial exploration of MaxScale. Seems promising. The biggest problem is that I don't know a whole heck of a lot about databases. Despite being a sysadmin for quite a while, I've not had to do much with databases, be it performance or otherwise, so the limitations here seem to be on my end.

The other issue is that MaxScale is relatively new. I'd be willing to try it out in a staging environment for sure. Using it might require a support contract for me to feel entirely comfortable, but then again if I had issues with it I could just pull it from use and go straight to on node. I don't say that lightly either because I don't have much in the way of support contracts right now. ;)

Hopefully in the future I can do more testing with MaxScale as it seems extremely useful, especially in terms of being able to stop servers and split read/write.

I also have some more work to do with sysbench. It seems like it's the best tool, that I know of, to test database performance with, but it is not working well with MaxScale.
