---
layout: post
title: Wordpress with FORCE_SSL_ADMIN behind a reverse proxy
categories: 
---

# {{ page.title }}

This is a pretty specific problem that I have run into a couple of times, both times forgetting the solution and spending an hour or two googling, so I decided to blog it so I can easily find it next time. :)

I am running a wordpress site behind [hipache](https://github.com/dotcloud/hipache), which is also doing ssl termination. So the actual wordpress site is served via plaintext http, and if ssl is required then hipache will provide it. Thus the connection from the client to hipache is ssl, but the connection from hipache to the apache server running wordpress is http.

Further, I want to force logins and access to /wp-admin to be ssl enabled using the wordpress option FORCE_SSL_ADMIN.

The problem is that with FORCE_SSL_ADMIN configured, but no https for apache serving wordpress, connections to wp-login.php or /wp-admin enter into a redirect loop. 

The solution is adding an extra bit of configuration code to wp-config.php when using a reverse proxy to terminate ssl. Here is the [page](http://codex.wordpress.org/Administration_Over_SSL) that describes what addtional configuration changes to make, which I also show below.

<pre>
<code>define('FORCE_SSL_ADMIN', true);
if ($_SERVER['HTTP_X_FORWARDED_PROTO'] == 'https')
       $_SERVER['HTTPS']='on';
</code>
</pre>

With that added to the top of wp-config.php I can now have hipache serve up plain http for all of the wordpress site except wp-login.php and /wp-admin. Hopefully I remember to look here next time... :) 





