---
layout:     post
title: Bash documentation
date: 2014-01-13
tags: bash documentation
---

One of the things that always bothers me is lack of proper
documentation. Now, I'm lazy just like everyone else, but if I'm going
to document something, I prefer to do it properly and keep it up to
date. I've inherited a nice suite of bash scripts, which aren't really
complicated, but they all have the same copy & pasted header that's
dated from 2003. Not exactly helpful.

So while I have a wiki that explains how some of the processes work on a
higher level, it would be nice to have clean documentation in my bash
scripts. Ideally, it would be embeddable, exportable and human read-able.

Here are a list of options I found while browsing around:

-   The old-fashioned embedded comments
-   [bashdoc](https://launchpad.net/bashdoc) (awk + ReST structure via
    python's docutils)
-   embedded Perl POD (via a [heredoc
    hack](http://bahut.alma.ch/2007/08/embedding-documentation-in-shell-script_16.html))
-   [ROBODoc](http://rfsber.home.xs4all.nl/Robo/robodoc.html)

Of these choices, POD seems a bit bloated to be inside a script, and
ROBODoc looks way overblown for my simple needs, so I've decided to go
with bashdoc. I'm already working with ReST, via this blog, and it fits
pretty much all the criteria. Plus, it has few dependencies (awk, bash
and python's docutils) and doesn't require a package for itself, so I
wouldn't feel bad about setting this up on production servers (although
I should really set it up as a git hook in the script repo or
something). However, the documentation for bashdoc is quite limited (irony
at it's finest). The best way to figure out what is going on is to read
lib/basic.awk, and the docutils source code, which isn't exactly
everyone's cup of tea. That said, it shouldn't be too difficult to build
a small template I can copy and paste everywhere, which will hopefully
be more useful than the current header.
