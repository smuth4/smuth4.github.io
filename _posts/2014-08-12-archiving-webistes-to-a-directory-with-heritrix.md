---
layout:     post
title:      Archiving websites to a directory with Heritrix
date:       2014-08-12
categories: heritrix archive
---

So I one day I found myself in the market for a good web archiver.
Specifically, there were some interesting open directories I wanted to
mirror. My ideal solution would be a web front end around
[wget](http://www.gnu.org/software/wget/manual/wget.html), but a little
bit of research and testing showed that such an architecture would be
too simplistic for the level of detail I wanted. There were a couple
spider frameworks I tried out, like [scrapy](http://scrapy.org/), but I
wasn't enthusiastic about the prospect of trying to roll my own
solution, when I knew sites like the [Internet
Archive](http://archive.org) had the exact kind of thing I had in mind,
and they use the
[Heritrix](https://webarchive.jira.com/wiki/display/Heritrix/Heritrix)
engine archive their material. The [Heritrix Wikipedia
page](http://en.wikipedia.org/wiki/Heritrix) mentions that it can output
in the same directory format as wget (perfect!), but there's no citation
for that, and the Heretrix documentation is unorganized, to say the
least.

Setting it up
=============

Software
--------

I used Heritrix 3.2.0, the most recent stable version, for this project.

Steps
-----

This is not going to be a full tutorial on how to use Heritrix. Be sure
to read the
[documentation](https://webarchive.jira.com/wiki/display/Heritrix/Heritrix#Heritrix-Documentation),
and to read the default job configuration file before starting a large
job.

Install Heritrix as per the instructions and get it started. Navigate to
the web interface and create a new job with the standard configuration.

Next we want to edit the disposition chain, which starts at line 335 in
my default configuration. The first bean defined should be the
`warcWriter`, which, obviously, writes out scraped content to WARC
files. WARC files are perfect for preserving websites exactly how they
were accessed, but are a little too clumsy to be convenient.

After the WARC bean, add the follow code:

{% highlight xml %}
<bean id="mirrorWriter" class="org.archive.modules.writer.MirrorWriterProcessor">
</bean>
{% endhighlight %}

Then, in the `dispositionProcessors` bean, remove the line
`<ref bean="warcWriter"/>` from the the `processors` list, and add
`<ref bean="mirrorWriter"/>`.

That's pretty much all that's needed to get started. There are more
parameters that can be tweaked, which can be found in the [3.2.0
Javadoc](http://builds.archive.org/javadoc/heritrix-3.2.0/org/archive/modules/writer/MirrorWriterProcessor.html),
but the defaults are not anything surprising.
