+++
title = "Building BPF applications with libbpf-boostrap"
date = 2020-11-29

[extra]
toc = true

[taxonomies]
categories = ["BPF"]
tags = ["bpf"]
+++

Get started with your own BPF application quickly and painlessly with
[libbpf-bootstrap](https://github.com/libbpf/libbpf-bootstrap) scaffolding,
which takes care of all the mundane setup steps and lets you dive right into
BPF fun and minimize the necessary boilerplate. We'll take a look at what
libbpf-bootstrap provides and how everything is tied together.

<!-- more -->

# Why libbpf-boostrap?

BPF is an amazing kernel technology, which allows anyone to take a pick
under the cover of how kernel functions without intense kernel development
experience and without spending tons of time to set things up for the kernel
development. BPF also eliminates the risk of crashing your OS while doing that.
Once you get up to speed with BPF, it's lots of fun and power in your hands.

But getting started with BPF can still be intimidating in a large part because
setting up a build workflow for even a simple "Hello, World"-like BPF
application requires a bunch of steps that could be frustrating and
intimidating for the new BPF developer. It's not really all that complicated,
but knowing the necessary steps is an (unnecessarily) hard part which probably
demotivates a lot of people from even trying, despite all the interest in and
promise of BPF.

[libbpf-bootstrap](https://github.com/libbpf/libbpf-bootstrap) is
a scaffolding playground setting up as much things as possible for beginner
users to let them dive straight into writing BPF programs and tinkering with
them without unnecessary frustrations of initial setup. It takes into account
best practices developed in BPF community over last few years and provides
a modern and convenient workflow with, arguably, best BPF user experience to
date. libbpf-bootstrap is relying on [libbpf](https://github.com/libbpf/libbpf)
and uses a simple Makefile. For users needed more advanced set ups, it should
be a good starting point. At the very least, if Makefile can't be used
directly, it's simple enough to just transfer the logic to whichever build
system needs to be used.

libbpf-bootstrap currently has two demo BPF applications available: `minimal`
and `bootstrap`. `minimal` is exactly that – the most minimal BPF application
that compiles, loads, and runs a simple BPF equivalent of `printf("Hello,
World!")`. Being the most minimal one, it also doesn't impose many
requirements on Linux kernel recentness and should run fine on quite old
kernel versions.

`minimal` is great for quick experimentation and trying things out locally,
but it's not set up to reflect the setup of a production-intended BPF-based
application deployable across a variety of kernels. `bootstrap` is such an
example. `bootstrap` demo shows off a real-world approach to building out
minimal, but fully functional and [portable](/posts/bpf-portability-and-co-re/)
BPF application. To that end, it does rely on [BPF CO-RE](/posts/bpf-portability-and-co-re/)
and kernel [BTF](/posts/btf-dedup/) support, so make sure that your Linux
kernel is built with `CONFIG_DEBUG_INFO_BTF=y` Kconfig. See [libbpf
README](https://github.com/libbpf/libbpf#bpf-co-re-compile-once--run-everywhere)
for the list of Linux distributions that have everything already setup for you.
If you'd like to minimize the hassle of building custom kernel, just stick
with the recent enough versions of any of the major Linux distros.

Additionally, `bootstrap` demonstrates BPF global variables usage (Linux 5.5+)
and [BPF ring buffer](/posts/bpf-ringbuf) use (Linux 5.8+). Neither of those
features are mandatory to build useful BPF application, but they bring huge
usability improvements and are the way that modern BPF application are built,
so I've added example of using them into a basic `boostrap` example.

# Prerequisites

BPF is a very dynamic technology that is constantly being developed and
evolved. This means that new features and capabilities are added all the time,
so depending on which of them you need, you might need newer kernel versions.
But BPF community takes backwards compatibility extremely seriously, which
means that old Linux kernels will still run BPF applications just fine,
provided you don't need the very latest feature sets. So the simpler and more
conservative your BPF application logic and feature set is, the higher the
chances are that you'll be able to run your BPF application on old kernels.

Having said that, BPF user experience gets better all the time and BPF in more
recent kernel versions provide profound improvements in BPF usability, so if
you are just getting started and don't have a strict requirements to support
outdated Linux kernel versions, make your life less painful and use the latest
kernel version you can get your hands on.

BPF program code is normally written in the C language with some code
organization conventions added to let [libbpf](https://github.com/libbpf/libbpf)
make sense of BPF code structure and load properly hand everything into the
kernel. [Clang](https://clang.llvm.org/) is the compiler used for BPF code
compilation and it's generally recommended to use the latest Clang you can.
Still, Clang 10 or newer should work fine for most BPF features, but some more
advanced [BPF CO-RE](/posts/bpf-portability-and-co-re/) features might require
Clang 11 or even 12 (e.g., for some of the more recent and  advanced CO-RE
relocation built-ins).

libbpf-bootstrap bundles with it libbpf (as a Git submodule) and bpftool (for
x86-64 architecture only) to avoid dependency on any specific (and potentially
outdated) versions available in your Linux distribution. Your system should
also have `zlib` (`libz-dev` or `zlib-devel` package) and `libelf`
(`libelf-dev` or `elfutils-libelf-devel` package) installed. Those are
dependencies of `libbpf` necessary to compile and run it properly.

This is not a primer on BPF technology itself, so some familiarity with basic
concepts like BPF program, BPF map, BPF hooks (attach points) are assumed. If
you need a refresher on BPF fundamentals, [these](https://docs.cilium.io/en/latest/bpf/)
[resources](https://qmonnet.github.io/whirl-offload/2016/09/01/dive-into-bpf/)
should be a good starting point.

In the rest of this post I'll walk you through the structure of
[libbpf-bootstrap](https://github.com/libbpf/libbpf-bootstrap), its Makefile
and both `minimal` and `bootstrap` examples. We'll look at libbpf conventions
and structuring BPF C code for use with libbpf as a BPF program loader, as well
as how to interact with your BPF programs from the user-space using libbpf
APIs.

# Libbpf-bootstrap overview

Here's the contents of the [`libbpf-bootstrap`](https://github.com/libbpf/libbpf-bootstrap)
repository:

```shell
$ tree
.
├── libbpf
│   ├── ...
│   ... 
├── LICENSE
├── README.md
├── src
│   ├── bootstrap.bpf.c
│   ├── bootstrap.c
│   ├── bootstrap.h
│   ├── Makefile
│   ├── minimal.bpf.c
│   ├── minimal.c
│   ├── vmlinux_508.h
│   └── vmlinux.h -> vmlinux_508.h
└── tools
    ├── bpftool
    └── gen_vmlinux_h.sh

16 directories, 85 files
```

`libbpf-bootstrap` bundles libbpf as a submodule in `libbpf/` sub-directory to
avoid depending on system-wide libbpf availability and version.

`tools/` contains `bpftool` binary, which is used to build [BPF
skeletons](/posts/bcc-to-libbpf-howto-guide/#bpf-skeleton-and-bpf-app-lifecycle)
of your BPF code. Similarly to libbpf, it's bundled to avoid depending on
system-wide bpftool availability and its version being sufficiently
up-to-date.

Additionally, bpftool can be used to generate your own `vmlinux.h` header
with all the Linux kernel type definitions. Chances are you won't need to do
that because libbpf-bootstrap already provides pre-generated
[vmlinux.h](https://raw.githubusercontent.com/libbpf/libbpf-bootstrap/master/src/vmlinux_508.h)
in `src/` sub-directory. It is based on default kernel config for Linux 5.8
with a bunch of extra BPF-related functionality enabled. This means it should
have lots of commonly needed kernel types and constants already. Due to [BPF
CO-RE](/posts/bpf-portability-and-co-re/), `vmlinux.h` doesn't have to match
your kernel configuration and version exactly. But if nevertheless you do need to
generate your custom `vmlinux.h`, feel free to check
[`tools/gen_vmlinux_h.sh`](https://github.com/libbpf/libbpf-bootstrap/blob/master/tools/gen_vmlinux_h.sh)
script to see how it can be done.

Beyond self-explanatory `LICENSE` and `README.md` the rest of `libbpf-bootstrap`
is contained in a `src/` sub-directory.

[Makefile](https://github.com/libbpf/libbpf-bootstrap/blob/master/src/Makefile)
defines the necessary build rules to compile all the supplied (and your
custom ones) BPF apps. It follows a simple file naming convention:
  - `<app>.bpf.c` files are the BPF C code that contain the logic which is to
    be executed in the kernel context;
  - `<app>.c` is the user-space C code, which loads BPF code and interacts with
    it throughout the lifetime of the application;
  - *optional* `<app>.h` is a header file with the common type definitions and
    is shared by both BPF and user-space code of the application.

So, `minimal.c` and `minimal.bpf.c` form the `minimal` BPF demo app. And
`bootstrap.c`, `bootstrap.bpf.c`, and `bootstrap.h` are the `bootstrap` BPF
app. Simple.

# Minimal app

`minimal` is a good example to start with. Consider it a minimalistic
playground for trying BPF things out. It doesn't use BPF CO-RE, so you can use
older kernels and just include your system kernel headers for kernel type
definitions. It's not the best approach for building production-ready
applications and tools, but is good enough for local experimentation.

## The BPF side

Here's the BPF-side code 
([minimal.bpf.c](https://github.com/libbpf/libbpf-bootstrap/blob/master/src/minimal.bpf.c))
*in its entirety*:

```c
// SPDX-License-Identifier: GPL-2.0 OR BSD-3-Clause
/* Copyright (c) 2020 Facebook */
#include <linux/bpf.h>
#include <bpf/bpf_helpers.h>

char LICENSE[] SEC("license") = "Dual BSD/GPL";

int my_pid = 0;

SEC("tp/syscalls/sys_enter_write")
int handle_tp(void *ctx)
{
	int pid = bpf_get_current_pid_tgid() >> 32;

	if (pid != my_pid)
		return 0;

	bpf_printk("BPF triggered from PID %d.\n", pid);

	return 0;
}
```

`#include <linux/bpf.h>` includes some basic BPF-related types and constants
necessary for using the kernel-side BPF APIs (e.g., BPF helper function
flags). This header is needed for the `bpf_helpers.h` header, included next.
`bpf_helpers.h` is provided by `libbpf` and contains most-often used macros,
constants, and BPF helper definitions, which are used by virtually every
existing BPF application. `bpf_get_current_pid_tgid()` above is an example of
such BPF helper.

`LICENSE` variable defines the license of your BPF code. Specifying the
license is mandatory and is enforced by the kernel. Some BPF functionality is
unavailable to non-GPL-compatible code. Note the special `SEC("license")`
annotation. `SEC()` (provided by `bpf_helpers.h`) puts variables and functions
into the specified sections. `SEC("license")`, along some other section names,
is the convention dictated by `libbpf`, so make sure you stick to it.

Next, we see the use of an exciting BPF feature: global variables. `int my_pid
= 0;` does exactly what you'd expect: it defines a global variable which BPF
code can read and update just like any user-space C code would do with
a global variable. It's extremely convenient and also performant to use BPF
global variables for maintaining the state of your BPF program.  Additionally,
such global variables can be read and written from the user-space side. This
feature is available starting from Linux 5.5 version. It is frequently used
for things like configuring BPF application with extra settings, low-overhead
stats, etc. It can also be used to pass data back-and-forth between in-kernel
BPF code and user-space control code.

`SEC("tp/syscalls/sys_enter_write") int handle_tp(void *ctx) { ... }` defines
the BPF program which will be loaded into the kernel.  It's is represented as
a normal C function in a specially-named section (using `SEC()` macro).
Section name defines what type of BPF program libbpf should create and
how/where it could be attached in the kernel. In this case, we define
a tracepoint BPF program, which will be called each time a `write()` syscall
is invoked from *any* user-space application.

> There could be many BPF programs defined within the same BPF C code file.
> They could have different types (i.e., `SEC()` annotations). E.g., you can
> have few different BPF programs, each for a different tracepoint or some
> other kernel event (e.g., network packet being processed, etc). You can also
> define multiple BPF programs with the same `SEC()` attribute: `libbpf` will
> handle that just fine. All BPF programs defined within the same BPF C code
> file share all the global state (like `my_pid` variable, but also any BPF
> map, if used). This is frequently utilized to coordinate few collaborating
> BPF programs.

Let's now look at what `handle_tp` BPF program is doing:

```c
	int pid = bpf_get_current_pid_tgid() >> 32;

	if (pid != my_pid)
		return 0;
```

This part gets the PID (or "TGID" in internal kernel terminology) encoded in
upper 32 bits of `bpf_get_current_pid_tgid()`'s return value. It then checks
if the process triggering `write()` syscall is our `minimal` process. This is
quite important on a busy system, because most probably lots of unrelated
processes are going to issue `write()`s, making it really hard to experiment
with your own BPF code on your own terms. `my_pid` global variable is going to
be initialized with the actual PID of the `minimal` process from the
user-space code below.

```c
	bpf_printk("BPF triggered from PID %d.\n", pid);
```

This is a BPF equivalent of `printf("Hello, world!\n")`! It emits formatted
string to the special file at `/sys/kernel/debug/tracing/trace_pipe`, which you
can cat to see its contents from the console (make sure you use `sudo` or run
under root):

```shell
$ sudo cat /sys/kernel/debug/tracing/trace_pipe
	<...>-3840345 [010] d... 3220701.101143: bpf_trace_printk: BPF triggered from PID 3840345.
	<...>-3840345 [010] d... 3220702.101265: bpf_trace_printk: BPF triggered from PID 3840345.
```

> `bpf_printk()` helper and `trace_pipe` file is not intended to be used in
> production, but it's indispensable for debugging BPF code and getting
> insights into what your BPF program is doing. As there is no BPF debugger
> yet, `bpf_printk()` is usually the fastest and most convenient way to debug
> a problem in a BPF code.

That's it for the BPF-side of `minimal` app. Feel free to add any extra code to the
body of `handle_tp()` BPF program and extend it according to your needs.

## The user-space side

Let's now look at how things are tied together from user-space
([minimal.c](https://github.com/libbpf/libbpf-bootstrap/blob/master/src/minimal.c)),
skipping some pretty obvious parts (please check the full sources anyways).

```c
#include "minimal.skel.h"
```

This includes a BPF skeleton of the BPF code in `minimal.bpf.c`. It is
auto-generated by bpftool in one of Makefile steps and reflects the high-level
structure of `minimal.bpf.c`. It also simplifies the BPF code deployment
logistics by embedding contents of the compiled BPF object code inside the
header file, which gets included from the user-space code. No extra files to
deploy along your application binary, just include the header and forget about
it.

> BPF skeleton is purely a `libbpf` construct, kernel knows nothing about it.
> But it is a huge quality of life improvement for BPF development process, so
> consider familiarizing yourself with it. See the [blog
> post](/posts/bcc-to-libbpf-howto-guide/#bpf-skeleton-and-bpf-app-lifecycle)
> for some more details about BPF skeleton.

libbpf-boostrap BPF skeletons are generated into `src/.output/<app>.skel.h`
after successful `make` invocation. To get a better intuition about it, here's
a high-level overview of the skeleton for `minimal.bpf.c`:

```c
/* SPDX-License-Identifier: (LGPL-2.1 OR BSD-2-Clause) */

/* THIS FILE IS AUTOGENERATED! */
#ifndef __MINIMAL_BPF_SKEL_H__
#define __MINIMAL_BPF_SKEL_H__

#include <stdlib.h>
#include <bpf/libbpf.h>

struct minimal_bpf {
	struct bpf_object_skeleton *skeleton;
	struct bpf_object *obj;
	struct {
		struct bpf_map *bss;
	} maps;
	struct {
		struct bpf_program *handle_tp;
	} progs;
	struct {
		struct bpf_link *handle_tp;
	} links;
	struct minimal_bpf__bss {
		int my_pid;
	} *bss;
};

static inline void minimal_bpf__destroy(struct minimal_bpf *obj) { ... }
static inline struct minimal_bpf *minimal_bpf__open_opts(const struct bpf_object_open_opts *opts) { ... }
static inline struct minimal_bpf *minimal_bpf__open(void) { ... }
static inline int minimal_bpf__load(struct minimal_bpf *obj) { ... }
static inline struct minimal_bpf *minimal_bpf__open_and_load(void) { ... }
static inline int minimal_bpf__attach(struct minimal_bpf *obj) { ... }
static inline void minimal_bpf__detach(struct minimal_bpf *obj) { ... }

#endif /* __MINIMAL_BPF_SKEL_H__ */
```

It has the `struct bpf_object *obj;` which can be passed to libbpf
API functions. It also has `maps`, `progs`, and `links` "sections", that
provide direct access to BPF maps and programs defined in your BPF code
(e.g., `handle_tp` BPF program). These references can be passed to
libbpf APIs directly to do something extra with BPF map/program/link.
Skeleton can also optionally have `bss`, `data`, and `rodata` sections that
allow **direct** (no extra syscalls needed) access to BPF global variables
from user-space. In this case, our `my_pid` BPF variable corresponds to
the `bss->my_pid` field.

Now onto what `main()` does in our `minimal` app:

```c
int main(int argc, char **argv)
{
	struct minimal_bpf *skel;
	int err;

	/* Set up libbpf errors and debug info callback */
	libbpf_set_print(libbpf_print_fn);
```

`libbpf_set_print()` provides a custom callback for all libbpf logs. This is
extremely useful, especially during active development, because it allows to
capture helpful libbpf debug logs. By default, libbpf will log only
error-level messages, if something goes wrong. Debug logs, though, are helpful
to get an extra context on what's going on and debug problems faster.

> If you ever need to report some problem with libbpf and/or your libbpf-based
> application (e.g., by sending email to the
> [bpf@vger.kernel.org](mailto://bpf@vger.kernel.org) mailing list), please
> always include full debug logs from libbpf.

In `minimal`'s case, `libbpf_print_fn()` just emits everything to stdout.

```c
	/* Bump RLIMIT_MEMLOCK to allow BPF sub-system to do anything */
	bump_memlock_rlimit();
```

This is a somewhat confusing, but necessary, step that pretty much any
realistic BPF application has to do. It bumps kernel's internal per-user
memory limit to allow BPF sub-system to allocate necessary resources for your
BPF programs, maps, etc. This limitation most probably is going away soon, but
for now you have to bump `RLIMIT_MEMLOCK`
[limit](https://man7.org/linux/man-pages/man2/getrlimit.2.html) one way or
another. Doing it through `setrlimit(RLIMIT_MEMLOCK, ...)`, as `minimal` code
is doing, is the simplest and the most convenient way.

```c
	/* Load and verify BPF application */
	skel = minimal_bpf__open_and_load();
	if (!skel) {
		fprintf(stderr, "Failed to open and load BPF skeleton\n");
		return 1;
	}
```

Now, using an auto-generated BPF skeleton, prepare and load BPF programs into
kernel and let the BPF verifier check it. If this step succeeds, your BPF
code is correct and ready to be attached to whatever BPF hooks you need.

```c
	/* ensure BPF program only handles write() syscalls from our process */
	skel->bss->my_pid = getpid();
```

But first, we need to communicate our PID to BPF code, so that it can filter
out irrelevant `write()` invocations from unrelated processes. This sets
`my_pid` BPF global variable value *directly, though the memory-mapped
region*. As mentioned above, this is how the user-space can access (read and
write) BPF global variables.

```c
	/* Attach tracepoint handler */
	err = minimal_bpf__attach(skel);
	if (err) {
		fprintf(stderr, "Failed to attach BPF skeleton\n");
		goto cleanup;
	}

	printf("Successfully started!\n");
```

We can finally attach `handle_tp` BPF program, which by now is readily
awaiting in the kernel, to the corresponding kernel tracepoint. This
"activates" the BPF program and the kernel will start executing our custom BPF
code in the kernel context in response to each `write()` syscall invocation!

> libbpf is able to automatically determine where to attach BPF program to by
> looking at its special `SEC()` annotation. This doesn't work for *all*
> possible BPF program types, but it does for lots of them: tracepoints,
> kprobes, and quite a few others. Additionally, libbpf provides extra APIs to
> do the attachment programmatically.

```c
	for (;;) {
		/* trigger our BPF program */
		fprintf(stderr, ".");
		sleep(1);
	}
```

Endless loop here makes sure that `handle_tp` BPF program stays attached in
the kernel until user kills the process (e.g., by pressing `Ctrl-C`). Also, it
will generate the `write()` syscall invocation periodically (once a second)
through `fprintf(stderr, ...)` call. This way it's possible to "monitor"
internals of the kernel from `handle_tp` and how the state changes over time.

```c
cleanup:
	minimal_bpf__destroy(skel);
	return -err;
}
```

If any of the previous steps go wrong, `minimal_bpf__destroy()` will clean up
all the resources (both in the kernel and in the user-space). It's a good
practice to make sure you always do this, but even if your application crashes
without cleaning up, the kernel will still clean up resources. Well, in most
cases, at least. There are some BPF program types which will stay active in
the kernel even if the owner user-space process dies, so make sure to check
that, if necessary.

And that's pretty much it for the `minimal` application. BPF skeleton use
makes all this pretty straightforward.

# Makefile

Now that we looked at the `minimal` app, we have enough context to look at
what [Makefile](https://github.com/libbpf/libbpf-bootstrap/blob/master/src/Makefile)
does to compile everything into a final executable. I'll skip some necessary
boilerplate parts and instead concentrate only on the essentials.

```Makefile
INCLUDES := -I$(OUTPUT)
CFLAGS := -g -Wall
ARCH := $(shell uname -m | sed 's/x86_64/x86/')
```

Here we define some extra parameters used during the compilation. By default,
all intermediate files will be written under `src/.output/` sub-directory, so
this directory is added into the include path for C compiler to find BPF
skeletons and libbpf headers. All the user-space files are compiled  with
debug info (`-g`) and without any optimizations to make it easier to debug
them. `ARCH` captures the host OS architecture, which is passed along into BPF
code compilation step later for use with low-level tracing helper macros (in
libbpf's `bpf_tracing.h`).

```Makefile
APPS = minimal bootstrap
```

This is a list of available applications. If you copy/paste `minimal` or
`bootstrap` and create your own copy, just add the name of your application
here to make it build. Each app defines corresponding make target, so you can
build just relevant files with:

```shell
$ make minimal
```

The whole build process happens in a few steps. First, libbpf is built as
a static library and its API headers are installed into `.output/`:

```Makefile
# Build libbpf
$(LIBBPF_OBJ): $(wildcard $(LIBBPF_SRC)/*.[ch] $(LIBBPF_SRC)/Makefile) | $(OUTPUT)/libbpf
	$(call msg,LIB,$@)
	$(Q)$(MAKE) -C $(LIBBPF_SRC) BUILD_STATIC_ONLY=1		      \
		    OBJDIR=$(dir $@)/libbpf DESTDIR=$(dir $@)		      \
		    INCLUDEDIR= LIBDIR= UAPIDIR=			      \
		    install
```

If you'd like to build against system-wide `libbpf` shared library, you can
remove this step and adjust compilation rules accordingly.

The next step builds BPF C code (`*.bpf.c`) into a compiled object file:

```Makefile
# Build BPF code
$(OUTPUT)/%.bpf.o: %.bpf.c $(LIBBPF_OBJ) $(wildcard %.h) vmlinux.h | $(OUTPUT)
	$(call msg,BPF,$@)
	$(Q)$(CLANG) -g -O2 -target bpf -D__TARGET_ARCH_$(ARCH) $(INCLUDES) -c $(filter %.c,$^) -o $@
	$(Q)$(LLVM_STRIP) -g $@ # strip useless DWARF info
```

We use Clang to do this. `-g` is mandatory to make Clang emit BTF information.
`-O2` is also necessary for BPF compilation. `-D__TARGET_ARCH_$(ARCH)` defines
necessary macro for `bpf_tracing.h` header dealing with low-level `struct
pt_regs` macro. You can disregard that if you are not dealing with kprobes and
`struct pt_regs`. Finally, we strip DWARF info out from the generated `.o`
file, as it's never used and is mostly just a compilation artifact of Clang.

> BTF is the only info necessary for BPF functionality and that one is
> preserved during stripping.  It's important to reduce the size of a `.bpf.o`
> file because it will get embedded into the final application binary through
> BPF skeleton, so there is no need to increase its size with unneeded DWARF
> data.

Now that we have `.bpf.o` file generated, `bpftool` is used to generate
a corresponding BPF skeleton header (`.skel.h`) with `bpftool gen skeleton`
command:

```Makefile
# Generate BPF skeletons
$(OUTPUT)/%.skel.h: $(OUTPUT)/%.bpf.o | $(OUTPUT)
	$(call msg,GEN-SKEL,$@)
	$(Q)$(BPFTOOL) gen skeleton $< > $@
```

With that, we make sure that whenever BPF skeleton is updated, user-space
parts of the application are rebuilt as well, because they need to embed BPF
skeleton during the compilation. The compilation of user-space `.c` → `.o` is
pretty straightforward otherwise:

```Makefile
# Build user-space code
$(patsubst %,$(OUTPUT)/%.o,$(APPS)): %.o: %.skel.h

$(OUTPUT)/%.o: %.c $(wildcard %.h) | $(OUTPUT)
	$(call msg,CC,$@)
	$(Q)$(CC) $(CFLAGS) $(INCLUDES) -c $(filter %.c,$^) -o $@
```

Finally, using only user-space `.o` file (together with `libbpf.a` static
library) the final binary is generated. `-lelf` and `-lz` are
dependencies of libbpf and need to be provided explicitly to the compiler:

```Makefile
# Build application binary
$(APPS): %: $(OUTPUT)/%.o $(LIBBPF_OBJ) | $(OUTPUT)
	$(call msg,BINARY,$@)
	$(Q)$(CC) $(CFLAGS) $^ -lelf -lz -o $@
```

That's it, after running through these few steps, you'll end up with a small
user-space binary that embeds compiled BPF code through BPF skeleton and has
statically linked libbpf in it, so doesn't depend on system-wide `libbpf`
availability. The result is a small (200KB), fast, stand-alone binary, just like
[Brendan Gregg asked](http://www.brendangregg.com/blog/2020-11-04/bpf-co-re-btf-libbpf.html).

# Bootstrap app

Now that we covered `minimal` app and how compilation is done in `Makefile`,
we'll go through some extra BPF features demonstrated by `bootstrap` app.
`bootstrap` is how I'd write a production-ready BPF application in the modern
BPF Linux environment. It relies on BPF CO-RE (read why
[here](/posts/bpf-portability-and-co-re/)) and requires Linux kernel built
with `CONFIG_DEBUG_INFO_BTF=y` (see
[here](https://github.com/libbpf/libbpf#bpf-co-re-compile-once--run-everywhere)).

`bootstrap` traces `exec()` syscalls (with `SEC("tp/sched/sched_process_exec")
handle_exit` BPF program), roughly corresponding to a spawning of a new
process (ignoring the `fork()` part, for simplicity). Additionally, it traces
`exit()`s (with `SEC("tp/sched/sched_process_exit") handle_exit` BPF program)
to know when each process exits. These two BPF programs, working together,
allow to capture interesting information about any new process, like binary's
filename, as well as to measure the lifetime of the process and collect
interesting stats when the process dies, like exit code or amount of consumed
resources, etc. I find it a great starting point to dive into the kernel
internals and observe how things really work under the hood.

`bootstrap` is also using [argp API](https://www.gnu.org/software/libc/manual/html_node/Argp.html)
(part of libc) for command-line argument parsing. Please check out
["Step-by-Step into Argp" tutorial](http://download.savannah.nongnu.org/releases-noredirect/argpbook/step-by-step-into-argp.pdf)
for the great intro into the `argp` usage. This is how optional minimum process
lifetime duration is parsed (see `min_duration_ns` read-only variable below;
use `sudo ./bootstrap -d 100` to show only processes that existed for at least
100ms), as well as verbose mode flag (try `sudo ./bootstrap -v`), enabling
`libbpf` debug logs.

## Includes: vmlinux.h, libbpf and app headers

Here's the include section on the BPF side of things
([bootstrap.bpf.c](https://github.com/libbpf/libbpf-bootstrap/blob/master/src/bootstrap.bpf.c)):

```
#include "vmlinux.h"
#include <bpf/bpf_helpers.h>
#include <bpf/bpf_tracing.h>
#include <bpf/bpf_core_read.h>
#include "bootstrap.h"
```

This differs from `minimal.bpf.c` in that we now use `vmlinux.h` header file,
which includes all the types from the Linux kernel in one file. It comes
[pre-generated](https://raw.githubusercontent.com/libbpf/libbpf-bootstrap/master/src/vmlinux_508.h)
with libbpf-bootstrap, but one can also generate the custom one with `bpftool`
(see [gen_vmlinux_h.sh](https://github.com/libbpf/libbpf-bootstrap/blob/master/tools/gen_vmlinux_h.sh)).

> All the types in `vmlinux.h` come with extra
> `__attribute__((preserve_access_index))` applied, which makes Clang generate
> [BPF CO-RE relocations](/posts/bpf-portability-and-co-re/#reading-kernel-structure-s-fields),
> allowing libbpf to adapt your BPF code to the specific memory layout of the
> host kernel, even if it differs from the one that `vmlinux.h` was originally
> generated from. This is a crucial aspect of building portable pre-compiled
> BPF application that doesn't require entire Clang/LLVM toolchain to be
> deployed alongside it to the target system. The alternative is
> [BCC](https://github.com/iovisor/bcc) way of compiling BPF code in runtime,
> which comes with [a bunch of downsides](/posts/bcc-to-libbpf-howto-guide/#why-libbpf-and-bpf-co-re).

Keep in mind that `vmlinux.h` can't be combined with other system-wide kernel
headers, as you'll inevitably run into type redefinitions and conflicts. So
please stick with using just `vmlinux.h`, libbpf-provided headers, and your
application's custom headers to avoid unnecessary headaches.

In addition to `bpf_helpers.h` we also use few extra libbpf-provided headers,
`bpf_tracing.h` and `bpf_core_read.h`, which provide some extra macros for
writing BPF CO-RE-based tracing BPF apps.

Finally, `bootstrap.h` contains common type definitions, shared between BPF and
user-space code of the `bootstrap` app (for BPF ringbuf, see below).

## BPF maps

> `bootstrap` demonstrates the use of BPF maps, which is a BPF concept for
> abstract data container. Many different things are modeled as BPF maps: from
> simple arrays and hash maps to per-socket and per-task local storage, BPF
> perf and ring buffers, and even some more exotic uses.  The important thing
> is that most BPF maps allow looking up, updating, and deleting its elements
> by some key. Some BPF maps allow extra (or alternative) operations, like
> [BPF ring buffer](/posts/bpf-ringbuf/), which allows to enqueue data, but
> never delete it from the BPF side. BPF maps are the means to share the state
> between (potentially many) BPF programs **and** user-space. The other one
> (more performant and convenient for storing simple plain data) is BPF global
> variables (which under the hood are still using BPF maps).

In `bootstrap`'s case, we define BPF map named `exec_start` of type
`BPF_MAP_TYPE_HASH` (a hash map) with the maximum size of 8192 entries, the
key is of `pid_t` type and the value is a 64-bit unsigned integer, storing the
nanosecond-granularity timestamp of process's exec event. This is a so-called
BTF-defined map.  `SEC(".maps")` annotation is mandatory to let libbpf know
that it needs to create the corresponding BPF map in the kernel and wire
everything properly in the BPF code:

```c
struct {
	__uint(type, BPF_MAP_TYPE_HASH);
	__uint(max_entries, 8192);
	__type(key, pid_t);
	__type(value, u64);
} exec_start SEC(".maps");
```

Adding/updating entries in such hashmap is simple:

```c
	pid_t pid;
	u64 ts;

	/* remember time exec() was executed for this PID */
	pid = bpf_get_current_pid_tgid() >> 32;
	ts = bpf_ktime_get_ns();
	bpf_map_update_elem(&exec_start, &pid, &ts, BPF_ANY);
```

`bpf_map_update_elem()` BPF helper takes pointers to the map itself, key and
value pointers, as well as extra flags, which in this case (`BPF_ANY`) tell to
either add a new key, or update the existing one.

Notice how the second BPF program (`handle_exit`) is looking up the element
from the same BPF map and subsequently deletes it. This shows how the
`exec_start` map is shared between the two BPF programs:

```c
	pid_t pid;
	u64 *start_ts;
	...
	start_ts = bpf_map_lookup_elem(&exec_start, &pid);
	if (start_ts)
		duration_ns = bpf_ktime_get_ns() - *start_ts;
	...
	bpf_map_delete_elem(&exec_start, &pid);
```

### Read-only BPF configuration variables

`bootstrap`, as opposed to `minimal`, is using a **read-only** global variable:

```c
const volatile unsigned long long min_duration_ns = 0;
```

`const volatile` part is important, it marks the variable as read-only for BPF
code and user-space code. In exchange, it makes the specific value of
`min_duration_ns` variable known to the BPF verifier during the BPF program
verification time. This (due to a more detailed knowledge) allows BPF
verifier to prune the dead code, if the read-only value provably omits some
code paths. This property is often desirable for some more advanced use cases,
like dealing with various compatibility checks and extra configuration.

> `volatile` is necessary to make sure Clang doesn't optimize away the
> variable altogether, ignoring user-space provided value. Without it, Clang
> is free to just assume 0 and **remove the variable completely**, which is
> not at all what we want.

From the user-space part (in [bootstrap.c](https://github.com/libbpf/libbpf-bootstrap/blob/master/src/bootstrap.c)),
there is a slight difference in initializing such read-only global variables.
They need to be set *before BPF skeleton is loaded* into the kernel. So,
instead of using a single-step `bootstrap_bpf__open_and_load()`, we need to
separately first `bootstrap_bpf__open()` the skeleton, set read-only
variable values, and only then `bootstrap_bpf__load()` skeleton into the
kernel:

```c
	/* Load and verify BPF application */
	skel = bootstrap_bpf__open();
	if (!skel) {
		fprintf(stderr, "Failed to open and load BPF skeleton\n");
		return 1;
	}

	/* Parameterize BPF code with minimum duration parameter */
	skel->rodata->min_duration_ns = env.min_duration_ms * 1000000ULL;

	/* Load & verify BPF programs */
	err = bootstrap_bpf__load(skel);
	if (err) {
		fprintf(stderr, "Failed to load and verify BPF skeleton\n");
		goto cleanup;
	}
```

Note that such read-only variables are part of `rodata` section in the
skeleton (not `data` or `bss`): `skel->rodata->min_duration_ns`. After the BPF
skeleton is loaded, user-space code can only read the value of the read-only
variable. BPF code can also only ever *read* such variables. BPF verifier will
*reject* the BPF program if it detects an attempt to write to such a variable.

## BPF ring buffer

`bootstrap` is using BPF ring buffer map heavily for preparing and sending
data back to user-space. It's using the `bpf_ringbuf_reserve()`/`bpf_ringbuf_submit()`
[combo](/posts/bpf-ringbuf/#bpf-ringbuf-reserve-commit-api) for best usability
and performance. Please check the BPF ring buffer [post](/posts/bpf-ringbuf/)
for more thorough coverage. That post goes through a very similar
functionality in detail, looking at examples in a separate
[bpf-ringbuf-examples](https://github.com/libbpf/bpf-ringbuf-examples/)
repo. It should also give you a pretty good idea how to use BPF perf buffer,
if you choose to do so.

## BPF CO-RE

BPF CO-RE (Compile Once – Run Everywhere) is a pretty big topic, covered
separately in a dedicated [blog post](/posts/bpf-portability-and-co-re/),
please make sure to check it out as well. Here's one example from
`bootstrap.bpf.c` of using BPF CO-RE features to read data from kernel's
`struct task_struct`:

```c
	e->ppid = BPF_CORE_READ(task, real_parent, tgid);
```

In non-BPF world, it would be written as just `e->ppid = task->real_parent->tgid;`,
but BPF verifier requires an extra effort because of the risk of reading an
arbitrary kernel memory. `BPF_CORE_READ()` takes care of this in a succinct
manner and records necessary BPF CO-RE relocations along the way, allowing
libbpf to adjust all the field offsets to the specific memory layout of the
host kernel. Refer to [this post](/posts/bpf-portability-and-co-re/#reading-kernel-structure-s-fields)
for more examples.

# Conclusion

This should do it for the broad coverage of `libbpf-bootstrap` and various
BPF/libbpf aspects. Hopefully, `libbpf-bootstrap` will allow you to get over
the initial hurdle of getting everything set up to get started with BPF
development and instead will allow to spend more time on BPF itself and
tinkering with kernel observability, tracing, what have you. That, after all,
is the most exciting part of using BPF (for me, at least).

For the more seasoned BPF developers, it should have demonstrated a way to set
up everything with modern BPF usability boosters like BPF skeleton, BPF
ringbuf, BPF CO-RE (just in case you haven't followed BPF development closely).

So please check the [Github repo](https://github.com/libbpf/libbpf-bootstrap)
and give it a go. PRs with bug fixes and improvements, as well as any
suggests, are always welcome. Have fun with BPF!
