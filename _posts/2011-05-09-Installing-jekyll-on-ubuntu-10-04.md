---
layout: post
title: Installing Jekyll on Ubuntu 10.04
categories:
- serverascode
---

# {{ page.title }}

Another small post...given that I've recently changed jobs and thus have new workstation(s) to install and configure as I like them, _and_ that I recently purchased a used Lenovo T61 laptop (which BTW is running very well on Ubuntu 10.04/Lucid, perhaps fodder for another post) I've repeated a several software installations lately on Lucid, including getting jekyll running locally so I can review blog posts before I send them up to [github](http://github.com) to run [serverascode.com](http://serverascode.com).

When you run @gem install jekyll@ on Lucid, you will recieve this error message:

<pre>
<code>$ grep -i release /etc/lsb-release 
DISTRIB_RELEASE=10.04
$ gem install jekyll
ERROR:  Error installing jekyll:
	liquid requires RubyGems version >= 1.3.7
</code>
</pre>

Lucid comes with @gem 1.3.5@ which is not the version that Jekyll's gem requires. When searching for the error message I found [this](http://help.rubygems.org/discussions/problems/350-installing-rubygems-137-and-jekyll-on-ubuntu-1004#comment_3175741) post which describes one way of getting jekyll running on lucid, which is to install the gem package from Ubuntu 10.10. Now, obviously installing a package from what essentially is a different version of Ubuntu isn't usually a recommended way to go, it's certainly a quick and easy one (duh! :) ). I downloaded the [rubygems1.8_1.3.7-2](http://packages.ubuntu.com/maverick/all/rubygems1.8/download) package and installed it. Then I was able to run @gem install jekyll@ and then run jekyll:

<pre>
<code>$ jekyll --server --auto
Configuration from /ccollicutt.github.com/_config.yml
Auto-regenerating enabled: /ccollicutt.github.com -> ccollicutt.github.com/_site
[2011-04-20 13:20:25] regeneration: 12 files changed
[2011-04-20 13:20:25] INFO  WEBrick 1.3.1
[2011-04-20 13:20:25] INFO  ruby 1.8.7 (2010-01-10) [x86_64-linux]
[2011-04-20 13:20:30] INFO  WEBrick::HTTPServer#start: pid=7082 port=4000
SNIP!
</code>
</pre>

It remains to be seen if I'll run into issues with having the `gem` from Ubuntu 10.10 running on Ubuntu 10.04. I'll update this post if I do. *:)*




