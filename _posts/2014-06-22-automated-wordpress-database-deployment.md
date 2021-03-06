---
layout: post
title: Automated deployment of the Wordpress database
categories:
---
 
# {{ page.title }}

I've been working on a proof-of-concept (PoC) [Wordpress-as-a-service](http://serverascode.com/2014/06/16/build-your-own-paas-docker.html) (WPaas?) that uses docker (among other technologies). However, one thing I can't seem to find is anyone showing how to easily install the Wordpress database after the application files have been installed and the Wordpress configuration file...er...configured. 

Most automated installs seem to stop after creating the database, which means you access the site via http and then continue the install manually. But obviously I don't want to stop at that point, I want the site name, admin password, admin user, etc, all entered and the database installed so the user can simply access the the site they asked for, you know, as though the install was fully automated.

## First try: http post

My first try was just doing a post with the right variables. Here's a simplified (no error checking) snippet of the python code I was using to do this.

<pre>
<code>payload = {'weblog_title': sitename, 'user_name': 'adminName', 'admin_password': dbPass, 'admin_password2': dbPass, 'admin_email': adminEmail }
r = requests.post("http://" + fullSiteName + "/wp-admin/install.php?step=2", data=payload)
</code>
</pre> 

You could also use a post request with something like curl:

<pre>
<code>$ curl -d "weblog_title=<weblog_title>&user_name=<admin_user>&admin_password=<password>&admin_password2=<password>2&admin_email=<email>" \
http://<your_wp_URL>/wp-admin/install.php?step=2
</code>
</pre>

This worked and seemed like an Ok option, but the only thing I didn't like was that I would have to wait until the container was up to send the http request. Not only was it annoying to have to wait (it's containers after all) but it's not very secure to have the site up but not completely installed, ie. anyone who happened onto it could set the admin user name and password. Even if I didn't add the site to the web router to be available externally, it still doesn't feel right. So while this was Ok for some basic testing, it's not good enough.

## Second try: wp_install() function

After a bit more sleuthing I found that wordpress has a handy install function. There's a good example of using it on [this blog](http://www.openlogic.com/wazi/bid/324425/How-to-install-WordPress-from-the-command-line). However, I found that I need the WP_SITEURL set as well.

Below is a snippet of a bash script that calls out to php to use wp_install().

<pre>
<code>#!/bin/bash
# ...
# Variable settings not shown
/usr/bin/php -r "
define('WP_SITEURL', '"${WP_SITEURL}"');
include '/app/wp-admin/install.php';
wp_install('"${WP_SITE}"', 'admin', '"${WP_EMAIL}"', 1, '', '"${DB_PASSWORD}"');
"
</code>
</pre>

This works better. Note the above is the bare minimum, and there should be some error checking as well, in case the wp_install call fails. It should really be a separate php script that can take arguments and return errors to the bash code calling it.

Running this from the container still concerns me a little.

## Database snapshots?

As I mentioned previously, this is a PoC. What would likely happen in a production environment is that certain versions of Wordpress would be supported, and the SQL for that would be configured automatically, ie. generate an SQL file for each site, and simply install that into the database instead of using wp_install. Or perhaps do something more advanced with database or file system snapshots. Once the db was snapshotted/cloned you could just change variables like siteurl, the admin user, and their password. Would likely be faster too. I'm sure there are a lot of interesting ways to perform database snapshots or clones.

Obviously newer versions of Wordpress could come with different database schemas, which would mean I'd have to "certify" various versions and properly create snapshots for those versions, should their be any schema differences. Totally doable though. Something to look into in the future, but a bit much for my little PoC.

## WP-CLI

[WP-CLI](http://wp-cli.org/), or "wordpress command line", is a pretty interesting project. I'm not sure how valuable it is in my particular use case, but it seems like a good piece of software, especially if you admin a multisite wordpress installation. Worth taking a look at.


