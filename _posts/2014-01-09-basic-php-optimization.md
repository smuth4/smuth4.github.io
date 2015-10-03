---
layout: post
title: Basic PHP Optimization
tags: php
---

A while ago I came across a nice peice of php software for controlling a
daemon (I'm specifically avoid the software's name for privacy reasons).
It worked well with a small data set, but quickly became laggy with a
dataset numbering in the thousands. Admittedly, it really wasn't built
for that kind of load, so I removed it and controlled the daemon
manually, which wasn't a big deal.

Then a while later, I came across a post by someone who managed to
mitigate the problem by shifting a particularly array-intensive
operation to an external python program. Obviously, this was not exactly
the most elegant solution, so I decided to take a look at the
problematic section of code.

It looked something like this:

{% highlight php %}
for ($i = 0; $i<count($array->val); $i+=$cnt) {
  $output[$array->val[$i]] = array_slice($array->val, $i+1, $cnt-1);
}
{% endhighlight %}

Looks pretty basic, right? It cycles through an array and splits
it out into a new dictionary (`$output`) based on a fixed length (`$cnt`).
However, if we replace the `array_slice` and `count` calls with the values
borrowed from [this serverfault post](http://stackoverflow.com/a/2484455), the problem quickly becomes
apparent.

{% highlight php %}
for ($i = 0; $i<O(n); $i+=$cnt) {
  $output[$req->val[$i]] = O(n);
}
{% endhighlight %}

Taking into account the for loop, this would appear to mean that the
operation is O(2n<sup>2</sup>), in contrast to the very similar
[`array_chunk`](http://www.php.net/manual/en/function.array-chunk.php)
O(n) function. So how do we optimize this? The most important thing to
do is make it so php can complete this in one loop over the array.
Everything else will be a nice improvement, but when scaling, the big O
is king.

Here's the new code:

{% highlight php %}
foreach($array->val as $index=>$value) {
  if($index % $cnt == 0) {
    $current_index = $value;
    $output[$current_index] = array();
  } else {
    $output[$current_index][] = $value;
  }
}
{% endhighlight %}

We've dropped the for/count() loop in favor of foreach, and eliminated
slicing in favor of appending to newly created elements. In a real world
test, this cut down the response time of the module from 12s to 4s on
average. A pretty big improvement for a pretty small change...
