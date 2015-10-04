---
layout: post
title: check_mk and FreeNAS, pt. 3
tags: check_mk freenas
---

A continuation of [check_mk and FreeNAS,  pt. 2]({% post_url 2014-05-25-check_mk-and-freenas-pt-2 %})

Now that we have all our smart data nicely set up, let's see if we can't
get some stats on I/O speed. I'm pretty sure FreeNAS is supposed to have
a I/O section in its "Reports" section, but for whatever reason, it's
not in my install, and I'd like to have the data in Nagios in any case.

Just like with the SMART data, we're going to write a small script that
the check\_mk agent can use. Unlike the SMART script, getting IO stats
is incredibly easy.

{% highlight bash %}
iostat -xI | grep da
{% endhighlight %}

Yep, that's all it is. We're only really interesting in the drives being
used in ZFS, but you could open it up to all drives if you wanted to.

Next up is to let check\_mk be able to recognize the agent's output. The
check script that I use can be found
[here](/check_mk_freenas_iostat/iostat). It should be placed in the
"checks/" directory of your check\_mk install.

{% highlight python %}
#!/bin/python
def check_iostat(item, params, info):
    warn, crit = params
    this_time = time.time()
    for line in info:
        if line[0] == item:
            try:
                timedif, read_ps = get_counter("iostat.%s.readps" % item, this_time, savefloat(line[1]))
                timedif, write_ps = get_counter("iostat.%s.writeps" % item, this_time, savefloat(line[2]))
                timedif, read_kb_ps = get_counter("iostat.%s.readkbps" % item, this_time, savefloat(line[3]))
                timedif, write_kb_ps = get_counter("iostat.%s.writekbps" % item, this_time, savefloat(line[4]))
                read_ps = round(read_ps, 2)
                write_ps = round(write_ps, 2)
                read_kb_ps = round(read_kb_ps, 2)
                write_kb_ps = round(write_kb_ps, 2)
            except MKCounterWrapped:
                read_ps = 0.0
                write_ps = 0.0
                read_kb_ps = 0.0
                write_kb_ps = 0.0
            data = [('qlen', line[5]),
                    ('svc_t', line[6]),
                    ('busy', line[7]),
                    ('readps', read_ps),
                    ('writeps', write_ps),
                    ('readkbps', read_kb_ps, warn, crit, 0, ""),
                    ('writekbps', write_kb_ps, warn, crit, 0, "")]
            if read_kb_ps > crit or write_kb_ps > crit:
                return (2, "%s kb/s read, %s kb/s write" % (read_kb_ps, write_kb_ps), data)
            elif read_kb_ps > warn or write_kb_ps > warn:
                return (1, "%s kb/s read, %s kb/s write" % (read_kb_ps, write_kb_ps), data)
            else:
                return (0, "%s kb/s read, %s kb/s write" % (read_kb_ps, write_kb_ps), data)

def inventory_iostat(info):
    return [(line[0], None) for line in info]

check_info["iostat"] = {
    'check_function':            check_iostat,
    'inventory_function':        inventory_iostat,
    'service_description':       'iostat drive %s',
    'has_perfdata':              True
}
{% endhighlight %}

I also created a quick template for pnp4nagios, found
[here](/check_mk_freenas_iostat/check_mk-iostat.php), which should be
placed in the "templates/" directory.

{% highlight php %}
<?php
#
# Plugin: check_mk-iostat
#
$ds_name[1] = "$NAGIOS_AUTH_SERVICEDESC";

$parts = explode("_", $servicedesc);
$disk = $parts[2];

$opt[1] = "--vertical-label 'Throughput (mb/s)' --title \"Disk throughput $hostname - disk $disk\" ";

$INDEX = array_flip($NAME);

$def[1] = rrd::def("readkbps", $RRDFILE[$INDEX['readkbps']], $DS[$INDEX['readkbps']], "AVERAGE");
$def[1] .= rrd::cdef("readmbps", "readkbps,1024,/");
$def[1] .= rrd::cdef("readmbps_neg", "readmbps,-1,*");
$def[1] .= rrd::def("writekbps", $RRDFILE[$INDEX['writekbps']], $DS[$INDEX['writekbps']], "AVERAGE");
$def[1] .= rrd::cdef("writembps", "writekbps,1024,/");
$def[1] .= rrd::cdef("writembps_neg", "writembps,-1,*");

if ($WARN[1] != "") {
  $def[1] .= "HRULE:$WARN[1]#FFFF00 ";
}
if ($CRIT[1] != "") {
  $def[1] .= "HRULE:$CRIT[1]#FF0000 ";
}
$def[1] .= rrd::area("readmbps", "#40c080", "Read  ") ;
$def[1] .= rrd::gprint("readmbps", "LAST", "%6.1lf MB/s last ");
$def[1] .= rrd::gprint("readmbps", "AVERAGE", "%6.1lf MB/s avg ");
$def[1] .= rrd::gprint("readmbps", "MAX", "%6.1lf MB/s max\\n");
$def[1] .= rrd::area("writembps_neg", "#4080c0", "Write ") ;
$def[1] .= rrd::gprint("writembps", "LAST", "%6.1lf MB/s last ");
$def[1] .= rrd::gprint("writembps", "AVERAGE", "%6.1lf MB/s avg ");
$def[1] .= rrd::gprint("writembps", "MAX", "%6.1lf MB/s max\\n");

$opt[2] = "--vertical-label 'Throughput (ops/s)' --title \"Disk throughput $hostname - disk $disk\" ";
$def[2] = rrd::def("readps", $RRDFILE[$INDEX['readps']], $DS[$INDEX['readps']], "AVERAGE");
$def[2] .= rrd::def("writeps", $RRDFILE[$INDEX['writeps']], $DS[$INDEX['writeps']], "AVERAGE");
$def[2] .= rrd::cdef("writeps_neg", "writeps,-1,*");
$def[2] .= rrd::area("readps", "#40c080", "Read  ");
$def[2] .= rrd::gprint("readps", array("LAST", "AVERAGE", "MAX"), "%6.2lf ");
$def[2] .= rrd::area("writeps_neg", "#4080c0", "Write ");
$def[2] .= rrd::gprint("writeps", array("LAST", "AVERAGE", "MAX"), "%6.2lf ");

$ds_name[3] = "$NAGIOS_AUTH_SERVICEDESC";
$opt[3] = "--vertical-label 'Duration (ms)' --title \"Transaction Duration $hostname - disk $disk\" ";
$def[3] = rrd::def("svc_t", $RRDFILE[$INDEX['svc_t']], $DS[$INDEX['svc_t']], "AVERAGE");
$def[3] .= rrd::line1("svc_t", "#FF2211", "Duration") ;
$def[3] .= rrd::gprint("svc_t", array("LAST", "AVERAGE", "MAX"), "%6.2lf ");

$ds_name[4] = "$NAGIOS_AUTH_SERVICEDESC";
$opt[4] = "--vertical-label '% Busy' --title \"% of time transaction were blocked $hostname - disk $disk\" ";
$def[4] = rrd::def("busy", $RRDFILE[$INDEX['busy']], $DS[$INDEX['busy']], "AVERAGE");
$def[4] .= rrd::area("busy", "#3366FF", "Busy") ;
$def[4] .= rrd::gprint("busy", array("LAST", "AVERAGE", "MAX"), "%6.2lf ");

$ds_name[5] = "$NAGIOS_AUTH_SERVICEDESC";
$opt[5] = "--vertical-label 'Length' --title \"Transaction Queue length $hostname - disk $disk\" ";
$def[5] = rrd::def("qlen", $RRDFILE[$INDEX['qlen']], $DS[$INDEX['qlen']], "AVERAGE");
$def[5] .= rrd::line1("qlen", "#FF2211", "Queue") ;
$def[5] .= rrd::gprint("qlen", array("LAST", "AVERAGE", "MAX"), "%6.2lf ");
?>
{% endhighlight %}

After all this, we've finally got a solid set up for tracking disks in
FreeNAS. For each disk, there will be three associated services: SMART
data, temperature (taken from the SMART data), and iostat data. See
below for a small [gallery](/galleries/check_mk/) of what it looks like
in my setup.
