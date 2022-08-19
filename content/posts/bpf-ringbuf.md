+++
date = 2020-10-26
updated = 2021-04-02
title = "BPF ring buffer"
description = '''
BPF ring buffer (ringbuf) and how it is similar and different from BPF perf
buffer. BPF ringbuf's APIs, performance and usability advantages. With
examples of source code.
'''

[extra]
toc = true

[taxonomies]
categories = ["BPF"]
tags = ["bpf"]
+++

There is now a new BPF data structure available: BPF ring buffer. It solves
memory efficiency and event re-ordering problems of the BPF perf buffer (a de
facto standard today for sending data from kernel to user-space) while meeting
or beating its performance. It provides both perfbuf-compatible for easy
migration, but also has the new reserve/commit API with better usability. Also,
both synthetic and real-world benchmarks show that in almost all cases so think
about making it a default choice for sending data from the BPF program to
user-space.

# BPF ringbuf vs BPF perfbuf

Today, whenever a BPF program needs to send collected data to user-space for
post-processing and logging, it usually uses BPF perf buffer (perfbuf) for
this. Perfbuf is a collection of per-CPU circular buffers, which allows to
efficiently exchange data between kernel and user-space. It works great in
practice, but due to its per-CPU design it has two major short-comings that
prove to be inconvenient in practice: inefficient use of memory and event
re-ordering.

To address these issues, starting from Linux 5.8, BPF provides a new BPF data
structure (BPF map): **BPF ring buffer** (ringbuf). It is a multi-producer,
single-consumer (MPSC) queue and can be safely shared across multiple CPUs
simultaneously.

BPF ringbuf support a familiar functionality from BPF perfbuf:
  - variable-length data records;
  - ability to efficiently read data from user-space through memory-mapped
    region without extra memory copying and/or syscalls into the kernel;
  - both epoll notifications support, as well as an ability to busy-loop for
    the absolutely minimal latency.

At the same time, BPF ringbuf solves the following issues with BPF perfbuf:
  - memory overhead;
  - data ordering;
  - wasted work and extra data copying.

## Memory overhead

BPF perfbuf allocates a separate buffer for each CPU. This often means that BPF
developers have to make a trade off between allocating big enough per-CPU
buffers (accommodating possible spikes of emitted data) or being
memory-efficient (by not wasting unnecessary memory for mostly empty buffers in
a steady state, but dropping data during data spikes). This is especially
tricky for applications that have big swings between being mostly idle most of
the time, but going through periodic big influx of events produced in a short
period of time. It's quite hard to find just the right balance, so BPF
applications would typically either over-allocate perfbuf memory to be on the
safe side, or will suffer inevitable data drops from time to time.

Being shared across all CPUs, BPF ringbuf allows using one big common buffer to
deal with this. Bigger buffer can absorb bigger spikes, but also might allow
using less RAM overall, compared to BPF perfbuf. BPF ringbuf *memory usage*
also scales better with increased amount of CPUs, because going from 16 to 32
CPUs doesn’t necessarily require twice as big a buffer to accommodate more
load.  With BPF perfbuf, you would have little choice in this matter,
unfortunately, due to per-CPU buffers.

## Event ordering

If a BPF application has to track correlated events (e.g., process start and
exit, network connection lifetime events, etc), proper ordering of events
becomes critical. This is problematic with BPF perfbuf, though. If correlated
events happen in rapid succession (within a few milliseconds) on different
CPUs, they might get delivered out of order. This is, again, due to the per-CPU
nature of BPF perfbuf.

As a real-world example, an application written by me a few years ago had to
track process fork/exec/exit events and collect per-process resource usage
stats. My application had to emit events into BPF perfbuf for each of those,
but quite often they were arriving out of order.  This is because fork(),
exec(), and exit() can happen in a very rapid succession on different CPUs for
short-lived processes due to the kernel scheduler migrating them from one CPU
to another. To deal with that required a significant increase of complexity in
application's handling logic, much more than one would expect given the
otherwise pretty straightforward nature of the problem.

This wouldn't be an issue at all with BPF ringbuf, which solves this problem by
emitting events into a shared buffer and guarantees that if event A was
submitted before event B, then it will be also consumed before event B. This
often will noticeably simplify handling logic.

## Wasted work and extra data copying

With BPF perfbuf, BPF program has to prepare a data sample, before copying it
into perf buffer to send to user-space. This means that the same data has to be
copied twice: first into a local variable or per-CPU array (for big samples
that can’t fit on a small BPF stack), and then into perfbuf itself. What's
worse, all this work could be wasted if it turns out that perfbuf doesn't have
enough space left in it.

BPF ringbuf supports an alternative reservation/submit API to avoid this. It’s
possible to first reserve the space for data. If reservation succeeds, the BPF
program can use that memory directly for preparing a data sample. Once done,
submitting data to user-space is an extremely efficient operation that can't
possibly fail and doesn't perform any extra memory copies at all. If
reservation failed due to running out of space in a buffer, at least you know
this before you’ve spent all that work trying to record the data, just to be
later dropped on the floor. `ringbuf-reserve-commit` example below will show
what that looks like in practice.

## Performance and applicability

BPF ringbuf performance beats BPF perfbuf for all practical purposes
(especially given a somewhat suboptimal *default* settings in BCC/libbpf for
consuming perfbuf data). You can find extensive synthetic benchmarking results
of various scenarios in [this
patch](https://patchwork.ozlabs.org/project/netdev/patch/20200529075424.3139988-5-andriin@fb.com/),
if you like hard numbers.

BPF perfbuf, theoretically, can support higher throughput of data due to
per-CPU buffers, but this becomes relevant only when we are talking about
*millions of events per second*. But experiments with writing a real-world
high-throughput application confirmed that BPF ringbuf is still a more
performant replacement for BPF perfbuf, if used (similarly to BPF perfbuf) as a
per-CPU buffer. Especially, if employing manual data availability notification.
You can check out basic multi-ringbuf example ([BPF
side](https://github.com/torvalds/linux/blob/master/tools/testing/selftests/bpf/progs/test_ringbuf_multi.c),
[user-space
side](https://github.com/torvalds/linux/blob/master/tools/testing/selftests/bpf/prog_tests/ringbuf_multi.c))
in one of kernel selftests. We'll look at an example of manual control over
data availability notification a bit later.

The only case where you might need to be careful and experiment first is when a
BPF program has to run from NMI (non-maskable interrupt) context (e.g., for
handling perf events, such as `cpu-cycles`). BPF ringbuf internally uses a very
lightweight spin-lock, which means that data reservation might fail, if lock is
contended in NMI context. So *in NMI context*, if CPU contention is high, there
might be some data drops even when ringbuf itself still has some space
available.

In all other cases it is a pretty clear choice in favor of the new BPF ringbuf.
BPF ringbuf provides a better performance and memory efficiency, better
ordering guarantees, and better API (both kernel-side and in user-space).

# Show me the code!

To show BPF ringbuf APIs, compare them to BPF perfbuf ones, and see how they
would be typically used in practice, I wrote a small [bpf-ringbuf-examples
project](https://github.com/anakryiko/bpf-ringbuf-examples), that we’ll go over
in the rest of this post.

This repo implements three variants of the same BPF application, which traces
all process execs capturing spawning of new processes. For each `exec()`,
process ID (`pid`), process name (`comm`), and executable file path
(`filename`) are captured into a sample and sent to the user-space for
post-processing (in our demo, just printf()'ing all that into the stdout).
Here's how the output from all three examples should look like (don't forget to
run it with `sudo`):

```bash
$ sudo ./ringbuf-reserve-commit    # or ./ringbuf-output, or ./perfbuf-output
TIME     EVENT PID     COMM             FILENAME
19:17:39 EXEC  3232062 sh               /bin/sh
19:17:39 EXEC  3232062 timeout          /usr/bin/timeout
19:17:39 EXEC  3232063 ipmitool         /usr/bin/ipmitool
19:17:39 EXEC  3232065 env              /usr/bin/env
19:17:39 EXEC  3232066 env              /usr/bin/env
19:17:39 EXEC  3232065 timeout          /bin/timeout
19:17:39 EXEC  3232066 timeout          /bin/timeout
19:17:39 EXEC  3232067 sh               /bin/sh
19:17:39 EXEC  3232068 sh               /bin/sh
^C
```

Here's the [C struct
definition](https://github.com/anakryiko/bpf-ringbuf-examples/blob/main/src/common.h#L22-L30)
of the sample data sent from BPF program and consumed on the user-space part of
the application:

```c
#define TASK_COMM_LEN 16
#define MAX_FILENAME_LEN 512

/* definition of a sample sent to user-space from BPF program */
struct event {
	int pid;
	char comm[TASK_COMM_LEN];
	char filename[MAX_FILENAME_LEN];
};
```

## BPF perfbuf: bpf_perf_event_output()

Let's start with a BPF perfbuf use case, BPF part of which can be found
[here](https://github.com/anakryiko/bpf-ringbuf-examples/blob/main/src/perfbuf-output.bpf.c).

First, we include some basic BPF definitions from kernel's `<linux/bpf.h>`,
BPF helper definitions from libbpf's `<bpf/bpf_helpers.h>`, and our
application's types from `"common.h"`, which is shared between BPF and
user-space code. We also specify that our program is under the dual
GPL-2.0/BSD-3 license:

```c
// SPDX-License-Identifier: GPL-2.0 OR BSD-3-Clause
/* Copyright (c) 2020 Andrii Nakryiko */
#include <linux/bpf.h>
#include <bpf/bpf_helpers.h>
#include "common.h"

char LICENSE[] SEC("license") = "Dual BSD/GPL";
```

Next, we define BPF perfbuf itself as a `BPF_MAP_TYPE_PERF_EVENT_ARRAY` map.
There is no need to define `max_entries` property, because libbpf will take
care of it and will automatically size it to the number of CPUs available on
the system. The size of each per-CPU buffer is specified separately from
user-space, which we'll get to in a second.

```c
/* BPF perfbuf map */
struct {
	__uint(type, BPF_MAP_TYPE_PERF_EVENT_ARRAY);
	__uint(key_size, sizeof(int));
	__uint(value_size, sizeof(int));
} pb SEC(".maps");
```

Because the sample (`struct event` defined in
[common.h](https://github.com/anakryiko/bpf-ringbuf-examples/blob/main/src/common.h#L6-20))
is pretty big (>512 bytes, as I set the maximum captured size of a filename to
512 bytes), we can't prepare the data on the stack. So we use a single-element
per-CPU array as a temporary storage:

```c
struct {
	__uint(type, BPF_MAP_TYPE_PERCPU_ARRAY);
	__uint(max_entries, 1);
	__type(key, int);
	__type(value, struct event);
} heap SEC(".maps");
```

Now we define the BPF program itself and specify that it should be attached to
the `sched:sched_process_exec`, which is triggered on each successful `exec()`
syscall. `struct trace_event_raw_sched_process_exec` is also defined in
[common.h](https://github.com/anakryiko/bpf-ringbuf-examples/blob/main/src/common.h#L6-20)
and is just copy/pasted from Linux sources. It defines the layout of input
data for that particular tracepoint.

```c
SEC("tp/sched/sched_process_exec")
int handle_exec(struct trace_event_raw_sched_process_exec *ctx)
{
	unsigned fname_off = ctx->__data_loc_filename & 0xFFFF;
	struct event *e;
	int zero = 0;

	e = bpf_map_lookup_elem(&heap, &zero);
	if (!e) /* can't happen */
		return 0;

	e->pid = bpf_get_current_pid_tgid() >> 32;
	bpf_get_current_comm(&e->comm, sizeof(e->comm));
	bpf_probe_read_str(&e->filename, sizeof(e->filename), (void *)ctx + fname_off);

	bpf_perf_event_output(ctx, &pb, BPF_F_CURRENT_CPU, e, sizeof(*e));
	return 0;
}
```

The BPF program logic is pretty straightforward. It fetches a temporary
storage for our sample and fills it out with data from tracepoint context.
When done, it sends the sample down to the BPF perfbuf with
`bpf_perf_event_output()` call.  This API reserves the space for `struct
event` in perf buffer on the current CPU, copies `sizeof(*e)` bytes of data
from `e` to that reserved space, and when done it will signal user-space that
the new data is available. At that point epoll subsystem will wakeup the
user-space handler and will pass the pointer to that copy of data for
processing.

And that's literally the entirety of the BPF side of things. Pretty
straightforward, if you have ever seen any other modern BPF application.

Let's skim through [the user-space
side](https://github.com/anakryiko/bpf-ringbuf-examples/blob/main/src/perfbuf-output.c)
now. It's relying on BPF skeleton (you can read more about it
[here](https://nakryiko.com/posts/bcc-to-libbpf-howto-guide/#bpf-skeleton-and-bpf-app-lifecycle)),
so it's pretty short and straightforward. After doing a minimal initial setup
(setting libbpf logging handler, interrupt handler, bumping `RLIMIT_MEMLOCK`
limit for BPF system), it just opens and loads the BPF skeleton. If all that
succeeds, we then use libbpf's user-space `perf_buffer__new()` API to create
an instance of perf buffer consumer:

```c
	struct perf_buffer *pb = NULL;
	struct perf_buffer_opts pb_opts = {};
	struct perfbuf_output_bpf *skel;

	...

	/* Set up ring buffer polling */
	pb_opts.sample_cb = handle_event;
	pb = perf_buffer__new(bpf_map__fd(skel->maps.pb), 8 /* 32KB per CPU */, &pb_opts);
	if (libbpf_get_error(pb)) {
		err = -1;
		fprintf(stderr, "Failed to create perf buffer\n");
		goto cleanup;
	}
```

Here we specified that each per-CPU buffer is 32KB (8 pages x 4096 bytes per
page), and for each submitted sample libbpf will call our `handle_event()`
callback, which just outputs data with `printf()`:

```c
void handle_event(void *ctx, int cpu, void *data, unsigned int data_sz)
{
	const struct event *e = data;
	struct tm *tm;
	char ts[32];
	time_t t;

	time(&t);
	tm = localtime(&t);
	strftime(ts, sizeof(ts), "%H:%M:%S", tm);

	printf("%-8s %-5s %-7d %-16s %s\n", ts, "EXEC", e->pid, e->comm, e->filename);
}
```

The last step is just to continuously consume data as soon as it becomes
available until it's time to exit (e.g., if user presses `Ctrl-C`):

```c
	/* Process events */
	printf("%-8s %-5s %-7s %-16s %s\n",
	       "TIME", "EVENT", "PID", "COMM", "FILENAME");
	while (!exiting) {
		err = perf_buffer__poll(pb, 100 /* timeout, ms */);
		/* Ctrl-C will cause -EINTR */
		if (err == -EINTR) {
			err = 0;
			break;
		}
		if (err < 0) {
			printf("Error polling perf buffer: %d\n", err);
			break;
		}
	}
```

## BPF ringbuf: bpf_ringbuf_output()

BPF ringbuf's `bpf_ringbuf_output()` API is designed to follow the semantics
of BPF perfbuf's `bpf_perf_event_output()` to make the migration a complete
no-brainer. To show how similar and close the usability is, I'll literally
show the diff between the `perfbuf-output` and `ringbuf-output` examples. You
can check out full [BPF-side
code](https://github.com/anakryiko/bpf-ringbuf-examples/blob/main/src/ringbuf-output.bpf.c)
and [user-space
code](https://github.com/anakryiko/bpf-ringbuf-examples/blob/main/src/ringbuf-output.c)
on Github.

Here's the BPF side diff:

```patch
--- src/perfbuf-output.bpf.c	2020-10-25 18:52:22.247019800 -0700
+++ src/ringbuf-output.bpf.c	2020-10-25 18:44:14.510630322 -0700
@@ -6,12 +6,11 @@
 
 char LICENSE[] SEC("license") = "Dual BSD/GPL";
 
-/* BPF perfbuf map */
+/* BPF ringbuf map */
 struct {
-	__uint(type, BPF_MAP_TYPE_PERF_EVENT_ARRAY);
-	__uint(key_size, sizeof(int));
-	__uint(value_size, sizeof(int));
-} pb SEC(".maps");
+	__uint(type, BPF_MAP_TYPE_RINGBUF);
+	__uint(max_entries, 256 * 1024 /* 256 KB */);
+} rb SEC(".maps");
 
 struct {
 	__uint(type, BPF_MAP_TYPE_PERCPU_ARRAY);
@@ -35,7 +34,7 @@
 	bpf_get_current_comm(&e->comm, sizeof(e->comm));
 	bpf_probe_read_str(&e->filename, sizeof(e->filename), (void *)ctx + fname_off);
 
-	bpf_perf_event_output(ctx, &pb, BPF_F_CURRENT_CPU, e, sizeof(*e));
+	bpf_ringbuf_output(&rb, e, sizeof(*e), 0);
 	return 0;
 }
 
```

So just two simple changes:

1. BPF ringbuf map is defined slightly differently. Its size (*but now it's
   the size of the buffer shared across all CPUs*) **can be** now defined on
   the BPF side. Keep in mind, it's still possible to omit it in BPF-side
   definition and specify (or override if you do specify it on BPF side) on
   user-space side with `bpf_map__set_max_entries()` API. The other difference
   is that the size (`max_entries` property) is specified in number of bytes,
   with the only restriction being that it should be a multiple of kernel page
   size (almost always 4096 bytes) and a power of 2 (similarly as for
   perfbuf's number of pages, which also has to be a power of 2). The BPF
   perfbuf size is specified from the user-space side and is in a number of
   memory pages.

2. `bpf_perf_event_output()` is replaced with the very similar
   `bpf_ringbuf_output()`, with the only difference that ringbuf API doesn't
   need a reference to the BPF program's context.

That’s the only two differences on the BPF side.

On the user-space side the changes are also minimal. Ignoring `perf_buffer`
<-> `ring_buffer` renames, it comes down to just two changes. First, the
definition of event handler callback can now return an error (which will
terminate consumer loop) and it doesn't take the index of a CPU that produced
the event:

```patch
-void handle_event(void *ctx, int cpu, void *data, unsigned int data_sz)
+int handle_event(void *ctx, void *data, size_t data_sz)
{
	const struct event *e = data;
	struct tm *tm;
```

If knowing the CPU index is important, you'll have to record that in your
sample from the BPF side explicitly. Also, `ring_buffer` API doesn't provide
a callback for lost samples, which `perf_buffer` does. That too would need to
be handled explicitly from the BPF side, if necessary. This was done to
minimize lock contention in a shared (across CPUs) ring buffer and not pay the
price when it's not needed. Also, in practice, there is rarely much one can do
about that, beyond just reporting this, which can be done more efficiently and
conveniently with explicit BPF code.

The second difference is a slightly simpler `ring_buffer__new()` API, which
allows specifying callback without resorting to extra options struct:

```patch
 	/* Set up ring buffer polling */
-	pb_opts.sample_cb = handle_event;
-	pb = perf_buffer__new(bpf_map__fd(skel->maps.pb), 8 /* 32KB per CPU */, &pb_opts);
-	if (libbpf_get_error(pb)) {
+	rb = ring_buffer__new(bpf_map__fd(skel->maps.rb), handle_event, NULL, NULL);
+	if (!rb) {
 		err = -1;
-		fprintf(stderr, "Failed to create perf buffer\n");
+		fprintf(stderr, "Failed to create ring buffer\n");
 		goto cleanup;
 	}
```

With that, simply replacing `perf_buffer__poll()` with `ring_buffer__poll()`
allows you to start consuming ring buffer data in exactly the same way:
 
```patch
 	printf("%-8s %-5s %-7s %-16s %s\n",
 	       "TIME", "EVENT", "PID", "COMM", "FILENAME");
 	while (!exiting) {
-		err = perf_buffer__poll(pb, 100 /* timeout, ms */);
+		err = ring_buffer__poll(rb, 100 /* timeout, ms */);
 		/* Ctrl-C will cause -EINTR */
 		if (err == -EINTR) {
 			err = 0;
 			break;
 		}
 		if (err < 0) {
-			printf("Error polling perf buffer: %d\n", err);
+			printf("Error polling ring buffer: %d\n", err);
 			break;
 		}
 	}
```

## BPF ringbuf: reserve/commit API

The goal for `bpf_ringbuf_output()` API was to allow a smooth transition from
BPF perfbuf to BPF ringbuf without any substantial changes to BPF code. But it
also means that it shares some of the shortcomings of BPF perfbuf APIs: extra
memory copy and very late data reservation. The former means that you need
extra space to construct a sample before copying it over to the buffer. This
is both less efficient and often requires extra complexity with single-element
per-CPU arrays. The latter means that all the work to construct a sample could
be wasted, if there is no more space left in the buffer due to the lagging
user-space Or because of a quick burst of incoming events overflowing the
buffer. But if you knew that the data will be dropped regardless, you could
just skip collecting it in the first place and save some resources for the
consumer side to catch up faster. But it's not possible with the
`xxx_output()`-style APIs.

That's where the `bpf_ringbuf_reserve()`/`bpf_ringbuf_commit()` APIs come in
handy. Reserve allows you to do just that: reserve the space early on or
determine that it's not possible (returning `NULL` in such case). If we can't
get enough data to submit the sample, we can skip spending all the resources
to capture data. But if the reservation succeeded, then we have a guarantee
that, once we are done with data collection, publishing it to the user-space
will never fail. I.e., if `bpf_ringbuf_reserve()` returns a non-NULL pointer,
subsequent `bpf_ringbuf_commit()` will always succeed.

Further, the reserved space in the ring buffer itself is not visible to
user-space until it is committed, so it can be freely used to construct
a sample, however complicated and multi-step operation it is. And it obviates
the need for extra memory copying and temporary storage spaces. The only
restriction is that the size of the reservation has to be known to the BPF
verifier at verification time, so samples with dynamic size would have to be
handled with `bpf_ringbuf_output()` and pay the price of extra copy.

But in most cases, reserve/commit is what you should prefer. Here's the diff
for BPF program code (full
[BPF](https://github.com/anakryiko/bpf-ringbuf-examples/blob/main/src/ringbuf-reserve-submit.bpf.c)
and
[user-space](https://github.com/anakryiko/bpf-ringbuf-examples/blob/main/src/ringbuf-reserve-submit.c)
code is on Github as well):

```patch
--- src/ringbuf-output.bpf.c	2020-10-25 18:44:14.510630322 -0700
+++ src/ringbuf-reserve-submit.bpf.c	2020-10-25 18:36:53.409470270 -0700
@@ -12,29 +12,21 @@
 	__uint(max_entries, 256 * 1024 /* 256 KB */);
 } rb SEC(".maps");
 
-struct {
-	__uint(type, BPF_MAP_TYPE_PERCPU_ARRAY);
-	__uint(max_entries, 1);
-	__type(key, int);
-	__type(value, struct event);
-} heap SEC(".maps");
-
 SEC("tp/sched/sched_process_exec")
 int handle_exec(struct trace_event_raw_sched_process_exec *ctx)
 {
 	unsigned fname_off = ctx->__data_loc_filename & 0xFFFF;
 	struct event *e;
-	int zero = 0;
 	
-	e = bpf_map_lookup_elem(&heap, &zero);
-	if (!e) /* can't happen */
+	e = bpf_ringbuf_reserve(&rb, sizeof(*e), 0);
+	if (!e)
 		return 0;
 
 	e->pid = bpf_get_current_pid_tgid() >> 32;
 	bpf_get_current_comm(&e->comm, sizeof(e->comm));
 	bpf_probe_read_str(&e->filename, sizeof(e->filename), (void *)ctx + fname_off);
 
-	bpf_ringbuf_output(&rb, e, sizeof(*e), 0);
+	bpf_ringbuf_submit(e, 0);
 	return 0;
 }
 
```

Gone is the per-CPU array and instead we use the result of
`bpf_ringbuf_reserve()` to fill the sample with data.

The user-space part is exactly the same (modulo the name of the BPF skeleton
object), which makes sense because in the end you are consuming exactly the
same data from the BPF ring buffer.

## BPF ringbuf: data notification control

When dealing with high-throughput cases, often the biggest overhead comes from
in-kernel signalling of data availability when a sample is submitted (this
lets kernel’s poll/epoll system to wake up user-space handlers blocked on
waiting for the new data). This is true for both perfbuf and ringbuf.

Perfbuf handles that with the ability to set up sampled notification, in which
case only every Nth sample will send a notification. You can do that when
creating a BPF perfbuf map from the user-space. And you need to make sure that
it works for you that you won’t see the last N-1 samples, until the Nth sample
arrives. This might or might not be a big deal for your particular case.

BPF ringbuf went a different route with this. `bpf_ringbuf_output()` and
`bpf_ringbuf_commit()` accept an extra flags argument and you can specify
either `BPF_RB_NO_WAKEUP` or `BPF_RB_FORCE_WAKEUP` flag. Specifying
`BPF_RB_NO_WAKEUP` inhibits sending in-kernel data availability notification.
While `BPF_RB_FORCE_WAKEUP` will force sending a notification. This allows for
the precise manual control, if necessary. To see how that can be done, please
check [BPF ringbuf
benchmark](https://github.com/torvalds/linux/blob/master/tools/testing/selftests/bpf/progs/ringbuf_bench.c#L22-L31),
which will send notifications only when a configurable amount of data is
enqueued in the ring buffer.

By default, if no flag is specified, BPF ringbuf code will do an adaptive
notification depending on whether the user-space consumer is lagging behind or
not, which results in the user-space consumer never missing a single sample
notification, but not paying unnecessary overhead. No flag is a good and safe
default, but if you need to get an extra performance, manually controlling
data notifications depending on your custom criteria (e.g., amount of enqueued
data in the buffer) might give you a big boost in performance.

# Conclusion

This post explains the problems BPF ring buffer is addressing and motivation
behind API choices. Hopefully, code samples and extra links to the kernel
selftests and benchmarks will let you get a good grasp on BPF ringbuf APIs and
demonstrate both a simple and more advanced use of the APIs to meet the needs
of your application.

