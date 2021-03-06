---
layout: post
title: Easy VPN or Proxy for Firefox with SSH
categories:
header_image: /img/glitch7.jpg
header_permalink: https://flic.kr/p/gV5m7b
---

# {{ page.title }}

I thought I would write a quick post on using OpenSSH as a simple vpn for use with Firefox. There are quite a few blog posts on how to do this already, but I think it's so simple and easy that it's worth repeating. It's really easy to setup, and if you use Firefox profiles you can have a profile that is pre-configured to use with any ssh socks connection. I will often use Firefox over an SSH connection to a remote host when using wifi at airports, on planes, in coffee shops, etc.

Also--I use it to gain access to internal web services remotely, as opposed to using some awful vpn system, one where the vpn server hasn't been updated in X number of years. Who wants to maintain a complex vpn system when all one needs is a safe and secure ssh server? Not me. :)

When using Firefox over an ssh vpn as discussed in this post, all your web and dns traffic from Firefox will be tunneled through the ssh socks proxy, and come out through the remote server. What's more, the tunnel is encrypted with ssh.

Of course, in order to use this you need a remote ssh server. But those are easy to come by, and you could easily startup a server in a public cloud, such as Digital Ocean. The instance could be quite small...512MB should be fine. $5 a month, start it up only when you know when you are going to need it, very inexpensive.

## Setup SSH Session

It can be this easy:

<pre>
<code>your-laptop$ ssh -D 127.0.0.1:8080 you@some.remote.host
some.remote.host$ # now logged in to remote host, ssh proxy is setup
</code>
</pre>

You can see that ssh is listening on localhost 8080.

<pre>
<code>your-laptop$ lsof -n -P -i :8080 -sTCP:LISTEN
COMMAND   PID   USER   FD   TYPE  DEVICE SIZE/OFF NODE NAME
ssh     27005 curtis    4u  IPv4 1769809      0t0  TCP 127.0.0.1:8080 (LISTEN)
</code>
</pre>

You could also use the -N and -T switches to not execute a remote command and disable getting a tty.

As long as that ssh session is up the proxy will be working.

## Configuring Firefox

It's straightforward to configure Firefox. To access the network settings in Firefox, go to Preferences -> Advanced -> Network.

1. Manual proxy
2. Socks host: 127.0.0.1
3. Port 8080 (or any you want, just has to be the same as you set in the ssh command)
4. Select "Socks V5"
5. Use remote DNS (which I prefer, but you don't have to)

![](/img/ssh-vpn.jpg)

## Firefox profiles

Firefox can have profiles. I have a specific profile setup to use with an ssh-based vpn on port 8080. So when I want to use Firefox over the ssh vpn I just open up Firefox using that profile. Unless you use Firefox with the ssh vpn all the time I prefer to use a profile so I don't have to go into the settings every time and make changes. I believe there might be some plugins that make it easier to have proxies setup, turned off and on and such, but I haven't used one so can't make a recommendation.

## Conclusion

We all need to use wifi, especially when travelling. However, wifi networks can have security issues. I definitely recommend using a vpn when working remotely over unknown wifi, and one of the easiest ways to do that is with ssh. 
