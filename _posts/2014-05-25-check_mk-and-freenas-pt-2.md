---
layout: post
title: check_mk and FreeNAS, pt. 2
tags: FreeNAS check_mk
---

A continuation of [check_mk and FreeNAS]({% post_url 2014-02-23-check_mk-and-freenas %})

So I've got my check\_mk on set up on my NAS, and it's monitoring stuff
beautifully. However, it's not monitoring something very near and dear
to my heart for this server: S.M.A.R.T. data. FreeNAS comes with
smartctl, and there's already S.M.A.R.T. data plugins for the linux
agents, so I figured this wouldn't be a big deal. And I was right! All I
had to do was add the following script to my plugins/ folder for
check\_mk to find, and the server picked it up automatically.

{% highlight bash %}
#!/bin/bash

# Only handle always updated values, add device path and vendor/model
if which smartctl > /dev/null 2>&1 ; then
    echo '<<<smart>>>'
    disks=$(sysctl kern.disks | cut -d " " -f 2-)
    for disk in ${disks[A]}; do
        D=/dev/$disk
        VEND="ATA"
        MODEL="$(sudo smartctl -i $D | grep 'Serial Number' | awk '{print $3}')"
        CMD="sudo smartctl -v 9,raw48 -A $D"

        $CMD | grep Always | egrep -v '^190(.*)Temperature(.*)' | sed "s|^|$D $VEND $MODEL |"
    done 2>/dev/null
fi
{% endhighlight %}

> **note**
>
> The linux script has a lot fancier checking for edge cases. I figured
> FreeNAS would be homogeneous enough that it wouldn't be worth
> converting all those edge cases, so this is a very simple script,
> shared on a "works for me" basis.
