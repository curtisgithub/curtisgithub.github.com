---
layout: post
title: Dealing with Zombie Cinder Volumes
categories:
header_image: /img/cinder_block.jpg
---

# {{ page.title }}

For some reason I ended up with a few volumes that were attached to non-existent virtual machines--ie. the vm had been deleted, but cinder thinks it is still attached to the vm. But the vm doesn't exist so you can't unattach or delete the volume. Maybe it's a chicken-egg volume not a zombie volume.

Here's a query to find these "zombie" volumes. I'm building somewhat on this [blog post](http://blog.carlos-spitzer.com/openstack-delete-cinder-orphan-volumes/). (Don't tell anyone but this was my first ever join SQL command--I haven't had to work much with databases.) This join was kind of fun because I had to pull information from both the nova and cinder databases, so note that it is using both the cinder and nova databases and expecting that their names will be "cinder" and "nova" respectively.

<pre>
<code>MariaDB [cinder]> SELECT tn.display_name as 'VM Name', tn.uuid as 'VM UUID', tn.vm_state as 'VM State', tc.status as 'Vol Status', tc.attach_status as 'Vol Attach Status', tc.id as "Vol UUID" FROM cinder.volumes tc JOIN nova.instances tn ON tc.instance_uuid = tn.uuid WHERE tn.vm_state = 'deleted' AND tc.attach_status = 'attached';
+-----------------------------------------+--------------------------------------+----------+------------+-------------------+--------------------------------------+
| VM Name                                 | VM UUID                              | VM State | Vol Status | Vol Attach Status | Vol UUID                             |
+-----------------------------------------+--------------------------------------+----------+------------+-------------------+--------------------------------------+
| coolvm1                                 | 2cf50467-044e-4f6c-8aad-283fb1c18f49 | deleted  | in-use     | attached          | 59a92e86-df60-4046-9cbe-221f9501cc5d |
| coolvm2                                 | 80f837ec-a076-4116-8443-34c17ff8b363 | deleted  | in-use     | attached          | 92926e7e-8015-449e-893e-e84b4a7a9fdc |
| coolvm3                                 | 7b6a15b2-b9ad-439b-a05f-acf6af4e7a86 | deleted  | in-use     | attached          | ffc905ad-6655-4e8b-8e21-6322e93773c4 |
+-----------------------------------------+--------------------------------------+----------+------------+-------------------+--------------------------------------+
3 rows in set (0.00 sec)
</code>
</pre>

Once I found all the volumes I considered "zombie" I updated their status manually in the cinder database. This was not much fun from an operator perspective. I think there is a "force_detach" command coming in future cinders.

<pre>
<code>MariaDB [cinder]> update volumes set attach_status='detached',status='available',instance_uuid=NULL where id='6fbd830c-3976-4427-be4b-c4daa11f9f49';
Query OK, 1 row affected (0.00 sec)
Rows matched: 1  Changed: 1  Warnings: 0
</code>
</pre>

I haven't been able to replicate this issue, so not sure why it happened in the first place.


