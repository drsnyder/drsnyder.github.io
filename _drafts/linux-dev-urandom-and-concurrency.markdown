---
layout: post
title: Linux /dev/urandom and concurrency
comments: true
---

This morning I was surprised to find out that a process that I expected to
to complete in about 8 hours was still running after 20. Everything appeared
to be operating normally. The load on the server was what we expected, IO was
minimal, and the external services it was using were responding with latencies
that were normal.

After tracing one of the processes (strace) I noticed that reads from
`/dev/urandom` were not what I expected. It was taking 80-200ms to read 4k from
this device. At first I thought that this was an issue with entropy but
`/dev/urandom` is non-blocking so that probably isn't the issue.

I didn't think that 80-200ms was typical so I tried a `dd` on the system
in question and another similar system in production. The system in question
took about 3 minutes to write 10Mb while the reference system took about 3s.
The only difference between the systems with respect to `/dev/urandom` was the
rate and number of processes reading from the device. The reads were probably on
the order of hundreds per second.

The number of processes reading from `/dev/urandom` made me wonder if maybe
there was a spinlock in the kernel in the read code. It turns out that there is.
You can see the spinlock [here](https://github.com/torvalds/linux/blob/v2.6.18/drivers/char/random.c#L470)
in the Linux kernel source code. The author mentions the concurrency issue
in a [thread](http://lkml.iu.edu//hypermail/linux/kernel/0412.1/0181.html)
on the mailing list from December 2004.

Fast forward 10 years and contention on this device is a real issue. Our
application uses `curl` from within PHP to fetch data from a
[cache](http://github.com/drsnyder/poky). The application has to process 10s of
millions of text objects and we don't want to wait days for that processing
to complete so we split the work of processing each object over N threads. The
problem appears to be coming from [`ares_init`](https://github.com/bagder/c-ares/blob/742e234855e9c038edb717ebe754169e9bf5e137/ares_init.c#L1899)
which is being called from `curl_easy_init` in our version of PHP+curl. Removing
the `AsynchDNS` feature from curl causes the problem to go away
(tracing confirms that the read from `/dev/urandom` is no longer there).

So why is this an issue? I wrote a simple script to measure the read times from
`/dev/urandom` as you increase the content. Here is a plot with the results.

![/dev/urandom reads as a function of threads]({{ site.url }}/assets/urandom-reads-t.png)

Running the same script with a user-land file is more or less linear out to 16
threads. A simple spinlock can have a big impact in the multi-core world of
2014!
