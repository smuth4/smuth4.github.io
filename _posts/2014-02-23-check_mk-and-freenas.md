---
layout: post
title: check_mk and FreeNAS
tags: FreeNAS check_mk
---

> **note**
>
> Software involved:
>
> -   FreeNAS 9.2.0
> -   OMD 1.10 (check\_mk 1.2.2p3)

FreeNAS is great, and the web interface makes it easy and simple to
understand my NAS's overall structure. However, my favored method of
monitoring in my homelab is OMD with Check\_mk, while FreeNAS prefers a
self-contained collectd solution. We're in luck however, in that FreeNAS
is heavily based on FreeBSD, which check\_mk happens to have a plugin
for, so it shouldn't be too hard to set things up the way I like them.
There are two possible ways to do this:

-   Enable inetd and point it to the check\_mk\_agent
-   Call check\_mk\_agent over ssh through as shown in the [datasource
    programs](http://mathias-kettner.com/checkmk_datasource_programs.html)
    documentation

I decided to go the second way, as I prefer to avoid making manual
changes to FreeNAS if I can avoid it.

If you do decided to go the inetd route, [this
thread](http://forums.freenas.org/index.php?threads/activation-of-inetd-server.3926/)
may come in useful.

Agent Setup
===========

The first thing we need to do is set up a user with a home directory
where we can store the check\_mk\_agent program. If you already have a
non-root user set up for yourself (which is good practice), that will
work perfectly fine (they may need root access to collect certain data
points). If you want to be more secure, you can set up a check\_mk-only
user, and limit it to just the agent command, which I will explain
below.

Once the user is set up with a writeable home directory, it's as simple
as copying check\_mk\_agent.freebsd into the home directory. Run it once
or twice to make sure it's collecting data correctly.

Check\_mk setup
===============

From here it's basically following the instructions on the [datasource
programs](http://mathias-kettner.com/checkmk_datasource_programs.html)
documentation link. Here's a quick overview of the steps involved:

1.  Add datasource programs configuration entry to main.mk. It will
    looks something like this:

{% highlight python %}
datasource_programs = [
    ( "ssh -l omd_user <IP> check_mk_agent", ['ssh'], ALL_HOSTS ),
]
{% endhighlight %}

2.  Set up password-less key authentication ssh access for omd\_user to
    FreeNAS
3.  (Optional) Limit omd\_user to onyl check\_mk\_agent command by
    placing command="check\_mk\_agent"
4.  Add ssh tag to FreeNAS host, through WATO or configuration files
5.  Enjoy the pretty graphs and trends!

Tweaks
======

So we've got everything set up, but not everything is perfect.

Network
-------

The first thing that I noticed was missing was a network interface
counter. check\_mk\_agent was outputting a 'netctr' section, which
seemed to have all the necessary information, but it wasn't being
recognized in check-mk inventory, as it's been superseeded by lnx\_if.
It's possible to re-enable netctr, but not on a per-host basis.
