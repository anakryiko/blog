+++
date = 2020-02-20
updated = 2021-04-02
title = "BCC to libbpf conversion guide"
description = '''
A practical guide to converting your BCC-based BPF application to libbpf and
BPF CO-RE (Compile Once – Run Everywhere).
'''

[extra]
toc = true

[taxonomies]
categories = ["BPF"]
tags = ["bpf", "bcc"]
+++

A practical guide to converting your BCC-based BPF application to libbpf and
[BPF CO-RE](/posts/bpf-portability-and-co-re/).

*This post was originally posted on Facebook's [BPF
blog](https://facebookmicrosites.github.io/bpf/blog/2020/02/20/bcc-to-libbpf-howto-guide.html).
This version has some minor fixes and adjustments. It also has as an updated
section on [using BPF subprograms](#helper-sub-programs) to reflect new libbpf
features added since the original publication.*

## Why libbpf and BPF CO-RE?

Historically, [BCC](https://github.com/iovisor/bcc/) was a framework of choice
when you had to develop a BPF application that required peering into the
internals of the kernel when implementing all sorts of tracing BPF programs.
BCC provided a built-in Clang compiler that could compile your BPF code in
runtime, tailoring it to a specific target kernel on the host. This was the
only way to develop a maintainable BPF application that had to deal with
internals of an ever-changing kernel. See [“BPF Portability and
CO-RE”](/posts/bpf-portability-and-co-re/) post, which does a better and more
detailed job explaining why is that so and why BCC was previously pretty much
the only viable choice. It also explains why
[libbpf](https://github.com/libbpf/libbpf) is a good choice now. Libbpf over
last year got a major boost in capabilities and sophistication, closed many
existing gaps with BCC, especially for tracing applications, and also gained
lots of new and powerful features not available in BCC (for instance, global
variables and BPF skeletons).

Granted, BCC is going to great lengths to simplify BPF developer’s life, but
sometimes that extra convenience gets in the way and makes it actually harder
to figure out what’s wrong and how to fix it. It feels like BCC simply has too
much magic at times. You have to remember naming conventions and
auto-generated structs for tracepoints. You have to rely on code rewriting to
read kernel data and fetch kprobe arguments. You’ll write
a semi-object-oriented C code when working with BPF maps, which doesn’t
exactly match what really happens in the kernel. And despite all the magic,
BCC will still make you write a bunch of boilerplate code in user-space part
of your application, setting up most trivial pieces by hand.

As mentioned above, BCC relies on runtime compilation and brings the entire
huge LLVM/Clang library in and embeds it inside itself. This has many
consequences, all of which are less than ideal:
*   heavy resource utilization (both memory and CPU) during compilation,
    potentially disrupting main workflow on a busy server;
*   dependency on kernel headers package, that has to be installed on every
    target host. And even then, if you need something from kernel that is not
    exposed through public headers – you’ll need to copy/paste type
    definitions into your BPF code by hand to get your work done;
*   even a trivial compilation-time errors are detected only during the
    runtime, after you’ve rebuilt and restarted your user-space application
    completely; this significantly reduces development iteration time (and
    increases frustration levels...)

Libbpf + BPF CO-RE (Compile Once — Run Everywhere) chose a different way.
Their philosophy is that BPF programs are not much different from any “normal”
user-space program: they ought to be compiled once into small binaries and
then deployed unmodified in a compact form to target hosts. Libbpf plays the
role of BPF program loader, performing mundane set up work (relocations,
loading and verifying BPF programs, creating BPF maps, attaching to BPF hooks,
etc), letting developers worry only about BPF program correctness and
performance. Such approach keeps overhead to the minimum, eliminates heavy
dependencies, makes the overall developer experience much more pleasant.

In terms of APIs and code conventions, libbpf sticks to the philosophy of the
least surprise, which means that most of the things have to be spelled out
explicitly: there won’t be any implicitly included headers and no code
rewriting. Just plain C code and a healthy dose of helper macro to eliminate
most mundane parts. Other than that, what you write is what gets executed and
the structure of your BPF application is 1-to-1 with what kernel ultimately
verifies and executes.

These guidelines were written to make BCC to libbpf + BPF CO-RE conversion
process easier, faster, and less painful. They explain various preliminary set
up steps, outline common patterns, explains differences, problems, and
gotchas, which you inevitably will encounter due to differences between BCC
and libbpf.

Switching from BCC to vanilla libbpf + BPF CO-RE might feel unusual and
confusing at first, but you’ll get a hang of it pretty quickly and will
appreciate libbpf’s explicitness and straightforwardness next time you run
into a compilation or verification problem.

*Also, keep in mind that a bunch of Clang features used for BPF CO-RE are
pretty new, so you’ll need a Clang 10 or newer to make all this work.*

## Setting up user-space parts

### Building everything

Building libbpf-based BPF application using BPF CO-RE consists of few steps:

*   generating `vmlinux.h` header file with all kernel types;
*   compiling your BPF program source code using recent Clang (version 10 or
    newer) into `.o` object file;
*   generating BPF skeleton header file from compiled BPF object file;
*   including generated BPF skeleton header to use from user-space code;
*   then, at last, compiling user-space code, which will get BPF object code
    embedded in it, so that you don’t have to distribute extra files with your
    application.

How exactly this is done will depend on your specific setup and build system,
which can’t be addressed here in enough details. For one way to do this please
check [BCC’s libbpf-tools](https://github.com/iovisor/bcc/tree/master/libbpf-tools),
which set up a generic Makefile rules that take care of all this for you in
a succinct manner.

When BPF code is compiled and BPF skeleton is generated, include libbpf and
skeleton headers in your user-space code to have necessary APIs ready to be
used:

```c
#include <bpf/bpf.h>
#include <bpf/libbpf.h>
#include "path/to/your/skeleton.skel.h"
```

### Locked memory limits

BPF is using locked memory for BPF maps and various other things. By default,
this limit is very low, so unless it’s increased, even a trivial BPF program
won’t load successfully into the kernel. BCC unconditionally sets this limit
to infinity, but libbpf doesn’t do this automatically (by design).

Depending on your production environment, there might be better and more
preferred ways of doing this. But for quick experimentation or if there is no
better way of doing this, you can do it yourself through
[setrlimit(2)](http://man7.org/linux/man-pages/man2/setrlimit.2.html) syscall,
which should be called at the very beginning of your program:

```c
    #include <sys/resource.h>

    rlimit rlim = {
        .rlim_cur = 512UL << 20, /* 512 MBs */
        .rlim_max = 512UL << 20, /* 512 MBs */
    };

    err = setrlimit(RLIMIT_MEMLOCK, &rlim);
    if (err)
        /* handle error */
```

### Libbpf log

When something doesn’t work as expected, the best way to start investigating
is to look at libbpf log output. Libbpf outputs a bunch of useful logs at
various levels of verbosity. By default, libbpf will emit error-level output
to console. We recommend installing a custom logging callback and set up
ability to turn on/off verbose debug-level output:

```c
int print_libbpf_log(enum libbpf_print_level lvl, const char *fmt, va_list args)
{
    if (!FLAGS_bpf_libbpf_debug && lvl >= LIBBPF_DEBUG)
        return 0;
    return vfprintf(stderr, fmt, args);
}

/* ... */

libbpf_set_print(print_libbpf_log); /* set custom log handler */
```

### BPF skeleton and BPF app lifecycle

Detailed explanation of using BPF skeleton (and libbpf API in general) is
beyond the scope of this document, existing [kernel selftests](https://git.kernel.org/pub/scm/linux/kernel/git/bpf/bpf-next.git/tree/tools/testing/selftests/bpf)
and BCC [libbpf-tools examples](https://github.com/iovisor/bcc/tree/master/libbpf-tools)
are probably the best way to get a feel for it. Check out
[runqslower](https://github.com/iovisor/bcc/blob/master/libbpf-tools/runqslower.c)
example as a simple but real tool utilizing skeleton.

Nevertheless, it’s useful to explain main libbpf concepts and phases that each
BPF application goes through. BPF application consists of a set of BPF
programs, either cooperating or completely independent, and BPF maps and
global variables, shared between all BPF programs (allowing them to cooperate
on a common set of data). BPF maps and global variables are also accessible
from user-space (we interchangeably refer to user-space part of application as
a "control app"), allowing the control app to get or set any extra data
necessary.  BPF application typically goes through the following phases:

*   **Open phase.** BPF object file is parsed: BPF maps, BPF programs, and
    global variables are discovered, but not yet created. After a BPF app is
    opened, it’s possible to make any additional adjustments (setting BPF
    program types, if necessary; pre-setting initial values for global
    variables, etc), before all the entities are created and loaded.
*   **Load phase.** BPF maps are created, various relocations are resolved,
    BPF programs are loaded into the kernel and verified. At this point, all
    the parts of a BPF application are validated and exist in kernel, but no
    BPF program is yet executed. After the load phase, it’s possible to set up
    initial BPF map state without racing with the BPF program code execution.
*   **Attachment phase.** This is the phase at which BPF programs get attached
    to various BPF hook points (e.g., tracepoints, kprobes, cgroup hooks,
    network packet processing pipeline, etc). This is the phase at which BPF
    starts performing useful work and read/update BPF maps and global
    variables.
*   **Tear down phase**. BPF programs are detached and unloaded from the
    kernel.  BPF maps are destroyed and all the resources used by the BPF app
    are freed.

Generated BPF skeleton has corresponding functions to trigger each phase:

*   `<name>__open()` – creates and opens BPF application;
*   `<name>__load()` – instantiates, loads, and verifies BPF application parts;
*   `<name>__attach()` – attaches all auto-attachable BPF programs (it’s
    optional, you can have more control by using libbpf APIs directly);
*   `<name>__destroy()` – detaches all BPF programs and frees up all used
    resources.


## BPF code conversion

In this part, we’ll go over typical conversion flow and will outline typical
mismatches between how BCC and libbpf/BPF CO-RE do things. Hopefully, this
will make it easy for you to convert your BPF code to be both BCC- and BPF
CO-RE compatible.

### Detecting BCC vs libbpf modes

For cases where you’ll need to support both BCC and libbpf "modes", it’s
useful to be able to detect which mode BPF program code is compiled for.
Simplest way to do this is to rely on the presence of `BCC_SEC` macro in BCC:

```c
#ifdef BCC_SEC
#define __BCC__
#endif
```

After this, throughout your BPF code, you can do:

```c
#ifdef __BCC__
/* BCC-specific code */
#else
/* libbpf-specific code */
#endif
```

This allows to have a common BPF source code with only necessary pieces of
logic be BCC- or libbpf-specific.

### Header includes

With libbpf/BPF CO-RE, you don’t need to include kernel headers (i.e., all
those `#include <linux/whatever.h>`), instead include a single `vmlinux.h` and
few libbpf helper headers:

```c
#ifdef __BCC__
/* linux headers needed for BCC only */
#else /* __BCC__ */
#include "vmlinux.h"               /* all kernel types */
#include <bpf/bpf_helpers.h>       /* most used helpers: SEC, __always_inline, etc */
#include <bpf/bpf_core_read.h>     /* for BPF CO-RE helpers */
#include <bpf/bpf_tracing.h>       /* for getting kprobe arguments */
#endif /* __BCC__ */
```

`vmlinux.h` might not contain some useful kernel `#define` constants, so for
those cases you’ll need to re-declare them here as well. Most common set of
constants is going to be provided inside `bpf_helpers.h`, though.

### Field accesses

BCC silently rewrites your BPF code and turns field accesses like
`tsk->parent->pid` into a series of `bpf_probe_read()` calls. Libbpf/BPF CO-RE
doesn’t have such a luxury, but `bpf_core_read.h` provides a set of helpers to
get as close to this in vanilla C as possible. The above `tsk->parent->pid`
will become `BPF_CORE_READ(tsk, parent, pid)`. With `tp_btf` and
`fentry`/`fexit` BPF program types, available since Linux 5.5, natural
C syntax is possible as well. But for older kernels and other BPF program
types (e.g., tracepoints and kprobes), your best bet is to convert to
`BPF_CORE_READ`.

Further, `BPF_CORE_READ` macro also works in BCC mode, so to avoid duplication
of every field access with `#ifdef __BCC__`/`#else`/`#endif`, you can convert
all the field reads into `BPF_CORE_READ` for both BCC and libbpf modes. With BCC,
make sure that `bpf_core_read.h` header is part of your final BPF program,
though.

### BPF maps

The way that BCC and libbpf define BPF maps declaratively is different, but
conversion is very straightforward. Here are some of the examples:

```c
/* Array */
#ifdef __BCC__
BPF_ARRAY(my_array_map, struct my_value, 128);
#else
struct {
    __uint(type, BPF_MAP_TYPE_ARRAY);
    __uint(max_entries, 128);
    __type(key, u32);
    __type(value, struct my_value);
} my_array_map SEC(".maps");
#endif

/* Hashmap */
#ifdef __BCC__
BPF_HASH(my_hash_map, u32, struct my_value);
#else
struct {
    __uint(type, BPF_MAP_TYPE_HASH);
    __uint(max_entries, 10240);
    __type(key, u32);
    __type(value, struct my_value);
} my_hash_map SEC(".maps")
#endif

/* Per-CPU array */
#ifdef __BCC__
BPF_PERCPU_ARRAY(heap, struct my_value, 1);
#else
struct {
    __uint(type, BPF_MAP_TYPE_PERCPU_ARRAY);
    __uint(max_entries, 1);
    __type(key, u32);
    __type(value, struct my_value);
} heap SEC(".maps");
#endif
```

**N.B.** Pay attention to the **default size of maps in BCC**, it’s usually
**10240**. With libbpf you have to specify the size explicitly.

`PERF_EVENT_ARRAY`, `STACK_TRACE` and few other specialized maps (`DEVMAP`,
`CPUMAP`, etc) don’t support (yet) BTF types for key/value, so specify
`key_size`/`value_size` directly instead:

```c
/* Perf event array (for use with perf_buffer API) */
#ifdef __BCC__
BPF_PERF_OUTPUT(events);
#else
struct {
    __uint(type, BPF_MAP_TYPE_PERF_EVENT_ARRAY);
    __uint(key_size, sizeof(u32));
    __uint(value_size, sizeof(u32));
} events SEC(".maps");
#endif
```

### Accessing BPF maps from BPF code

BCC employs pseudo-C++ syntax for working with maps, which gets rewritten to
actual BPF helper calls under the cover. Typically, the following pattern:

```c
some_map.operation(some, args)
```

Needs to be rewritten into the following one:

```c
bpf_map_operation_elem(&some_map, some, args);
```

Here are some examples:

```c
#ifdef __BCC__
    struct event *data = heap.lookup(&zero);
#else
    struct event *data = bpf_map_lookup_elem(&heap, &zero);
#endif

#ifdef __BCC__
    my_hash_map.update(&id, my_val);
#else
    bpf_map_update_elem(&my_hash_map, &id, &my_val, 0 /* flags */);
#endif

#ifdef __BCC__
    events.perf_submit(args, data, data_len);
#else
    bpf_perf_event_output(args, &events, BPF_F_CURRENT_CPU, data, data_len);
#endif
```

### BPF programs

All functions representing BPF programs need to be marked with custom section
name through use of `SEC()` macro, coming from `bpf_helpers.h`, like in the
example below:

```c
#if !defined(__BCC__)
SEC("tracepoint/sched/sched_process_exec")
#endif
int tracepoint__sched__sched_process_exec(
#ifdef __BCC__
    struct tracepoint__sched__sched_process_exec *args
#else
    struct trace_event_raw_sched_process_exec *args
#endif
) {
/* ... */
}
```

It’s just a convention, but you’ll get a much better experience overall if you
follow libbpf’s section naming. Detailed list of expected names can be found
[here](https://github.com/libbpf/libbpf/blob/787abf721ec8fac1a4a0a7b075acc79a927afed9/src/libbpf.c#L7935-L8075).
Few most common ones would be:

*   `tp/<category>/<name>` for tracepoints;
*   `kprobe/<func_name>` for kprobe and `kretprobe/<func_name>` for kretprobe;
*   `raw_tp/<name>` for raw tracepoint;
*   `cgroup_skb/ingress`, `cgroup_skb/egress`, and a whole family of
    `cgroup/<subtype>` programs.


### Tracepoints

From the example above, notice how there is a slight difference between type
name of tracepoint context type. BCC follows the
`tracepoint__<category>__<name>` naming pattern for tracepoint
“<category>/<name>”. BCC auto-generates corresponding type during runtime
compilation. Libbpf doesn’t have this luxury, but, luckily, kernel already
provides very similar types with all the tracepoint data. Typically, it will
be named `trace_event_raw_<name>`, but sometimes few tracepoints in kernel
will reuse common type, so if the mentioned pattern doesn’t work, you might
need to look it up in kernel sources (or check `vmlinux.h`) to find out exact
type name. E.g., instead of `trace_event_raw_sched_process_exit`, you’d
have to use `trace_event_raw_sched_process_template`.

**N.B.** For the most part, code that accesses tracepoint context data is
exactly the same, except for special variable-length string fields. For them,
the conversion is straightforward: `data_loc_<some_field>` becomes
`__data_loc_<some_field>` (notice double underscores).


### Kprobes

BCC also has a bunch of magic for declaring kprobes. In reality, such BPF
programs accept a single pointer to `struct pt_regs` as a context argument,
but BCC allows you to pretend like kernel function arguments are available
directly to a BPF program. With libbpf, you can get close with the help of
`BPF_KPROBE` macro, which currently is part of kernel selftests’
`bpf_trace_helpers.h` header, but should become part of libbpf soon:

```c
#ifdef __BCC__
int kprobe__acct_collect(struct pt_regs *ctx, long exit_code, int group_dead)
#else
SEC("kprobe/acct_collect")
int BPF_KPROBE(kprobe__acct_collect, long exit_code, int group_dead)
#endif
{
    /* BPF code accessing exit_code and group_dead here */
}
```

For return probes, there is corresponding `BPF_KRETPROBE` macro as well.

**Note!** Syscall functions got renamed in 4.17 kernels. Starting from 4.17
version, syscall kprobe that used to be called, say, `sys_kill`, is called now
`__x64_sys_kill` (on x64 systems, other architectures will have different
prefix, of course). You’ll have to account for that when trying to attach
a kprobe/kretprobe. If possible, though, try to stick to tracepoints.

**N.B.** If you are developing a new BPF application with the need for
tracepoint/kprobe/kretprobe, check out new raw_tp/fentry/fexit probes. They
provide better performance and usability and are available starting from 5.5
kernels.


### Dealing with compile-time #if’s in BCC

It’s pretty popular in BCC code to rely on preprocessor `#ifdef` and `#if`
conditions. Most typically this is done because of differences between kernel
version or to enable/disable optional pieces of logic (which depend on
application configuration). In addition, BCC allows to provide custom
`#define`’s from user-space side and substitute them in runtime during BPF
code compilation. This is often used to customize various parameters.

It’s impossible to do this with libbpf + BPF CO-RE in the same way (by using
compile-time logic), because the whole idea is that your BPF program has to be
compiled once and be able to handle all possible variations of kernel and
application configurations.

For dealing with kernel version differences, BPF CO-RE supplies two
complementary mechanisms: **Kconfig externs** and **struct “flavors”**. BPF
code can know which kernel version it’s dealing with by declaring the
following extern variable:

```c
#define KERNEL_VERSION(a, b, c) (((a) << 16) + ((b) << 8) + (c))

extern int LINUX_KERNEL_VERSION __kconfig;

if (LINUX_KERNEL_VERSION < KERNEL_VERSION(5, 2, 0)) {
  /* deal with older kernels */
} else {
  /* 5.2 or newer */
}
```

Similarly to getting the kernel version, you can extract any `CONFIG_xxx`
value from Kconfig:

```c
extern int CONFIG_HZ __kconfig;

/* now you can use CONFIG_HZ in calculations */
```

Often, if some field got renamed or moved to a sub-struct, it’s enough to just
detect that fact by checking whether the field exists in a target kernel. You
can do that using a helper `bpf_core_field_exists(<field>)`, which will return
1, if specified field is present in target kernel; or 0, otherwise. Paired
with struct flavors, this allows to deal with major changes in kernel struct
layouts (see [“BPF Portability and CO-RE”](/posts/bpf-portability-and-co-re/)
post for more details on using struct flavors). Here’s a short example on how
one can accommodate, as an example, `struct kernfs_iattrs` differences between
recent kernel versions:

```c
/* struct kernfs_iattrs will come from vmlinux.h */

struct kernfs_iattrs___old {
    struct iattr ia_iattr;
};

if (bpf_core_field_exists(root_kernfs->iattr->ia_mtime)) {
    data->cgroup_root_mtime = BPF_CORE_READ(root_kernfs, iattr, ia_mtime.tv_nsec);
} else {
    struct kernfs_iattrs___old *root_iattr = (void *)BPF_CORE_READ(root_kernfs, iattr);
    data->cgroup_root_mtime = BPF_CORE_READ(root_iattr, ia_iattr.ia_mtime.tv_nsec);
}
```


### Application configuration

BPF CO-RE’s way to customize behavior of your program is through using *global
variables*. Global variables allow user-space control app to pre-setup
necessary parameters and flags before a BPF program is loaded and verified.
Global variables can be either mutable or constant. Constant (read-only)
variables are most useful for specifying one-time configuration of a BPF
program, right before it is loaded into the kernel and verified. Mutable ones
can be used for bi-directional exchange of data between BPF program and its
user-space counterpart after BPF program is loaded and running.

**On BPF code side**, you can declare read-only global variables using
a `const volatile` global variable (for mutable ones, just drop `const
volatile` qualifiers):

```c
const volatile struct {
    bool feature_enabled;
    int pid_to_filter;
} my_cfg = {};
```

Few very important things to note here:

*   `const volatile` **has to be specified** to prevent too clever compiler
    optimizations (compiler might and will erroneously assume zero values and
    inline them in code);
*   if you are defining a _mutable_ (non-`const`) variable, make sure they are
    not marked as `static`: non-static globals interoperate with compiler the
    best. `volatile` is usually not necessary in such case;
*   your variables **have to be initialized**, otherwise libbpf will decline
    to load the BPF application. Initialization can be to zeroes or any other
    value you need. Such value will be a default value of variable, unless
    overridden from a control app.

Using global variables from BPF code is trivial:

```c
if (my_cfg.feature_enabled) {
    /* … */
}

if (my_cfg.pid_to_filter && pid == my_cfg.pid_to_filter) {
    /* … */
}
```

Global variables provide much nicer user experience and avoid BPF map lookup
overhead. Additionally, for constant variables, their values are well-known to
BPF verifier and treated as constants during program verification, which
allows BPF verifier to verify code more precisely and eliminate dead code
branches effectively.

The way **control app** provides values for such variables is simple and
natural with the usage of BPF skeleton:

```c
struct <name> *skel = <name>__open();
if (!skel)
    /* handle errors */

skel->rodata->my_cfg.feature_enabled = true;
skel->rodata->my_cfg.pid_to_filter = 123;

if (<name>__load(skel))
    /* handle errors */
```

Read-only variables, can be set and modified from user-space _only before
a BPF skeleton is loaded_. Once a BPF program is loaded, neither BPF nor
user-space code will be able to modify it. This guarantee allows BPF verifier
to treat such variables as constants during validation and perform better dead
code elimination. Non-const variables, on the other hand, can be modified
after BPF skeleton is loaded throughout the entire lifetime of BPF program,
both from BPF and user-space sides. They can be used for exchanging mutable
configuration, stats, etc.


## Common issues

There are a bunch of surprises that you might run into. Sometimes it’s just
a popular misconception, sometimes a difference between how something is
achieved in BCC vs how it should be done with libbpf. This is not an
exhaustive list, but it should help you with BCC to libbpf + BPF CO-RE
conversion.


### Global variables

BPF global variables look and behave exactly like a user-space variables: they
can be used in expressions, updated (the non-`const` ones), you can even take
their address and pass around into helper functions. But that is only true for
the **BPF code side**. From user-space, they can be read and updated **only
through BPF skeleton**:
*   `skel->rodata` for read-only variables;
*   `skel->bss` for mutable zero-initialized variables;
*   `skel->data` for non-zero-initialized mutable variables.

You can still read/update them from user-space and those updates will be
immediately reflected on the BPF side. But they are not global variables on
the user-space side, they are just members of BPF skeleton’s `rodata`, `bss`,
or `data` members, which are initialized during the skeleton load phase. This,
subsequently, means that declaring exactly the same global variable in BPF
code and user-space code will declare completely independent variables, which
won’t be connected in any way.


### Loop unrolling

Unless you are targeting 5.3+ kernel, all the loops in your BPF code have to
be marked with `#pragma unroll` to force Clang to unroll them and eliminate
any possible control flow loops:

```c
#pragma unroll
for (i = 0; i < 10; i++) { ... }
```

Without loop unrolling or if the loop doesn’t terminate within fixed amount of
iterations, you’ll get a verifier error about “back-edge from insn X to Y”,
meaning that BPF verifier detected an infinite loop (or can't prove that loop
will finish in a limited amount of iterations).


### Helper sub-programs

If you are using static functions and running on kernels older than
4.16, you'll have to mark such function as alway inlined with `static
__always_inline`, so that BPF verifier will see them as a single big function:

```c
static __always_inline unsigned long
probe_read_lim(void *dst, void *src, unsigned long len, unsigned long max)
{
    ...
}
```

But as of 4.16 (see
[commit](https://github.com/torvalds/linux/commit/cc8b0b92a1699bc32f7fec71daa2bfc90de43a4d)),
kernel supports BPF-to-BPF function calls within your BPF application.  libbpf
(v0.2+) has full generic support for this feature as well, making sure that
all the right code relocations and adjustments are performed. So feel free to
drop `__always_inline`. You might even consider enforcing no inlining with
`__noinline`, which quite often will improve code generation and will avoid
some of the common BPF verification failures due to unwanted register-to-stack
spilling:

```c
static __noinline unsigned long
probe_read_lim(void *dst, void *src, unsigned long len, unsigned long max)
{
    ...
}
```

Non-inlined *global* functions are also supported starting from 5.5 kernels,
but they have different semantics and verification constraints than static
functions. Make sure to check them out as well!

### bpf_printk debugging

There is no conventional debugger available for BPF programs, allowing setting
a breakpoint, inspecting variables and BPF maps, or single-stepping through
your code. But often times figuring out what's wrong with your BPF code is
nearly impossible without such a tool.

For such cases, logging extra debug information is your best bet. Use
`bpf_printk(fmt, args...)` to emit extra pieces of data to help understand
what's going on. It accepts `printf`-like format string and can handle only
**up to 3 arguments**. It's simple and easy to use, but it's _quite
expensive_, making it unsuitable to be used in production. So it's mostly
appropriate only for ad-hoc debugging. Usage:

```c
char comm[16];
u64 ts = bpf_ktime_get_ns();
u32 pid = bpf_get_current_pid_tgid();

bpf_get_current_comm(&comm, sizeof(comm));
bpf_printk("ts: %lu, comm: %s, pid: %d\n", ts, comm, pid);
```

Logged messages can be read from a special
`/sys/kernel/debug/tracing/trace_pipe` file:

```bash
$ sudo cat /sys/kernel/debug/tracing/trace_pipe
...
      [...] ts: 342697952554659, comm: runqslower, pid: 378
      [...] ts: 342697952587289, comm: kworker/3:0, pid: 320
...
```

## How can you contribute?

If all this whetted your appetite and you'd like to play with libbpf and BPF
CO-RE, helping with BCC tools conversion would probably be the best way to
start. See [BCC PR](https://github.com/iovisor/bcc/pull/2755) that added first
such converted tool (runqslower), and set up build scripts to easily add more
such tools. Just pick up any tool you like or use, and try converting it to
libbpf. If you have any questions, [BPF mailing
list](http://vger.kernel.org/vger-lists.html#bpf) is the best place to ask
questions and send bug reports about BPF, libbpf, and CO-RE in general. BCC
tools-related issues are best routed to [BCC
project](https://github.com/iovisor/bcc) itself. Have fun!
