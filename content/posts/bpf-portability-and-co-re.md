+++
date = 2020-02-19
updated = 2021-04-02
title = "BPF CO-RE (Compile Once – Run Everywhere)"
description = '''
Why most BPF applications have to be portable across multiple Linux kernels
and how BPF CO-RE (Compile Once – Run Everywhere) technology makes that
possible and easy.
'''

[extra]
toc = true

[taxonomies]
categories = ["BPF"]
tags = ["bpf", "linux"]
+++

What does it mean for a BPF application to be portable? And why it's actually
hard to achieve that without BPF Compile Once — Run Everywhere (CO-RE)? In
this post we'll see what are the challenges of writing BPF programs that can
work across multiple kernel versions and how BPF CO-RE is helping to address
this problem.

*This post was originally posted on Facebook's [BPF
blog](https://facebookmicrosites.github.io/bpf/blog/2020/02/19/bpf-portability-and-co-re.html).
If you are curious about some of the new things that happened since BPF CO-RE
got introduced initially, please see ["BPF CO-RE as of
2021"](#bpf-co-re-as-of-2021) section below. There is now also the ["BPF CO-RE
reference guide"](/posts/bpf-core-reference-guide/) with a lot of practical
tips on using BPF CO-RE features in real-world BPF applications.*

## BPF: state of the art

Since the inception of (e)BPF, it’s been a constant priority for the BPF
community to simplify BPF application development as much as possible, make it
as straightforward and familiar of an experience as it would be for
a user-space application. And with the steady progress around BPF
programmability, writing BPF programs has never been easier.

Despite these usability improvements, though, one aspect of BPF application
development has been neglected (mostly for technical reasons): portability.
What does "BPF portability" mean, though? **BPF portability** is the ability
to write a BPF program that will successfully compile, pass kernel
verification, and will work **correctly** across *different kernel versions*
without the need to recompile it for each particular kernel.

This note describes the BPF portability problem and our solution to it: BPF
CO-RE (Compile Once – Run Everywhere). First, we’ll look at the BPF
portability problem itself, describing why it is a problem and why it’s
important to solve it. Then, we will outline high-level components of the
solution, BPF CO-RE, and will give a glimpse into the pieces of the puzzle
that needed to be put together to make it happen. Lastly, we’ll conclude with
a tutorial of sorts, describing the user-visible API of the BPF CO-RE approach
and demonstrating its application with examples.

## The problem of BPF portability

A BPF program is a piece of user-provided code which is injected straight into
a kernel. Once loaded and verified, BPF programs execute in kernel context.
These programs operate inside kernel memory space with access to all the
internal kernel state available to it. This is extremely powerful and is one
of the reasons why BPF technology is successfully used in so many varied
applications. However, this powerful capability also creates the BPF
portability pains we have today: BPF programs do not control memory layout of
a surrounding kernel environment. They have to work with what they get from
independently developed, compiled, and deployed kernels.

Additionally, kernel types and data structures are in constant flux. Different
kernel versions will have struct fields shuffled around inside a struct, or
even moved into a new inner struct. Fields can be renamed or removed, their
types changed, either into some trivially-compatible ones or completely
different ones. Structs and other types can get renamed, or they can be
conditionally compiled out (depending on kernel configuration), or just plain
removed between kernel versions.

In other words, things change all the time between kernel releases and yet BPF
application developers are expected to cope with this problem somehow. How is
it even possible to do anything useful with BPF today considering this
ever-changing kernel environment? There are a few reasons for this.

First, not all BPF programs need to look into internal kernel data structures.
One example is `opensnoop` tool, which relies on kprobes/tracepoints to track
which processes open which files, and just needs to capture a few syscall
arguments to work. As syscall parameters offer a stable ABI, these don’t
change between kernel versions and as such portability is not a concern to
begin with. Unfortunately, applications like this are quite rare. These types
of applications are also typically quite limited in what they can do.

So, additionally, BPF machinery inside kernel provides a limited set of
"stable interfaces" that BPF programs can rely on to be stable between
kernels. In reality, underlying structures and mechanisms do change, but these
BPF-provided stable interfaces abstract such details from user programs.

As one example, for networking applications it is usually enough to look at
a limited set of `sk_buff`'s attributes (and packet data, of course) to be
extremely useful and versatile. To that end, BPF verifier provides a stable
`__sk_buff` "view" (notice underscores in front), which shields BPF programs
from changing `struct sk_buff` layout. All the `__sk_buff` field accesses are
transparently rewritten into an actual `sk_buff` accesses (sometimes quite
elaborate ones – doing a bunch of internal pointer chasing before finally
fetching the requested field). Similar mechanisms are available to a bunch of
different BPF program types. They are done as program type-specific BPF
contexts understood by BPF verifier. So, if you are developing a BPF program
with such context, consider yourself lucky, you can blissfully live in a nice
illusion of stability.

But as soon as you need to get a glimpse at any raw internal kernel data
(e.g., very commonly a `struct task_struct` which represents a process/thread
and contains a treasure trove of process information), you are on your own. It
is commonly the case for tracing, monitoring, and profiling applications,
which are a huge class of extremely useful BPF programs.

In such cases, how do you make sure you are not reading garbage data when some
kernel added an extra field before the field you thought is, say, at offset 16
from the start of `struct task_struct`? Suddenly, for that kernel, you'll need
to read data from, e.g., offset 24. And the problems don't end there: what if
a field got renamed, as was the case with `thread_struct`'s `fs` field (useful
for accessing thread-local storage), which got renamed to `fsbase` between 4.6
and 4.7 kernels. Or what if you have to run on two different configurations of
a kernel, one of which disabled some specific feature and completely compiled
out parts of the struct (a common case for additional accounting fields, which
are optional, but extremely useful if present)? All this means that you can no
longer compile your BPF program locally using kernel headers of your dev
server and distribute it in compiled form to other systems, while expecting it
to work and produce correct results. This is because kernel headers for
different kernel versions will specify a different memory layout of data your
program relies on.

So far, people have been dealing with this problem by relying on
[BCC](https://github.com/iovisor/bcc/) (BPF Compiler Collection). With BCC,
you embed your BPF program C source code into your user-space program (control
application) *as a plain string*. When control application is eventually
deployed and executed on target host, BCC invokes its embedded Clang/LLVM,
pulls in local kernel headers (which you have to make sure are installed on
the system from correct `kernel-devel` package), and performs compilation on
the fly. This will make sure that memory layout that BPF program expects is
exactly the same as in the target host's running kernel. If you have to deal
with some optional and potentially compiled-out stuff in kernel, you'll just
do `#ifdef`/`#else` guarding in your source code to accommodate such hazards
as renamed fields, different semantics of values, or any optional stuff not
available on current configuration. Embedded Clang will happily remove
irrelevant parts of your code and will tailor BPF program code to specific
kernel.

This sounds great, doesn't it? Not quite so, unfortunately. While this
workflow works, it's not without major drawbacks.

* Clang/LLVM combo is a big library, resulting in big fat binaries that need
  to be distributed with your application.

* Clang/LLVM combo is resource-heavy, so when you are compiling BPF code at
  start up, you'll use a significant amount of resources, potentially tipping
  over a carefully balanced production workfload. And vice versa, on a busy
  host, compiling a small BPF program might take minutes in some cases.

* You are making a big bet that the target system will have kernel headers
  present, which most of the time is not a problem, but sometimes can cause
  a lot of headaches. This is also an especially annoying requirement for
  kernel developers, because they often have to build and deploy custom
  one-off kernels as part of their development process. And without
  a custom-built kernel header package, no BCC-based application will work on
  such kernels, stripping developers of a useful set of tools for debugging
  and monitoring.

* BPF program testing and development iteration is quite painful as well, as
  you are going to get even most trivial compilation errors only in runtime,
  once you recompile and restart your user-space control application. This
  certainly increases friction and is not helping to iterate fast.

Overall, while BCC is a great tool, especially for quick prototyping,
experimentation, and small tools, it certainly has lots of disadvantages when
used for widely deployed production BPF applications.

But BPF CO-RE is stepping up the game of BPF portability and is arguably the
future of BPF program development, especially for complex real-world BPF
applications.

## High-level BPF CO-RE mechanics

BPF CO-RE brings together necessary pieces of functionality and data at all
levels of the software stack: kernel, user-space BPF loader library (libbpf),
and compiler (Clang) – to make it possible and easy to write BPF programs in
a portable manner, handling discrepancies between different kernels within the
same pre-compiled BPF program. BPF CO-RE requires a careful integration and
cooperation of the following components:

* BTF type information, which allows to capture crucial pieces of information
  about kernel and BPF program types and code, enabling all the other parts of
  BPF CO-RE puzzle;

* compiler (Clang) provides means for BPF program C code to express the intent
  and record relocation information;

* BPF loader ([libbpf](https://github.com/libbpf/libbpf)) ties BTFs from
  kernel and BPF program together to adjust compiled BPF code to specific
  kernel on target hosts;

* kernel, while staying completely BPF CO-RE-agnostic, provides advanced BPF
  features to enable some of the more advanced scenarios.

Working in ensemble, these components enable unprecedented ability to develop
portable BPF programs with ease, adaptability, and expressivity, previously
achievable only through compiling BPF program’s C code in runtime through BCC,
but without paying a high price for the BCC way.

### BTF

One of the crucial enablers of the entire BPF CO-RE approach is BTF. BTF ([BPF
Type Format](https://www.kernel.org/doc/html/latest/bpf/btf.html)) was created
as an alternative to a more generic and verbose DWARF debug information. BTF
is a space-efficient, compact, yet still expressive enough format to describe
all the type information of C programs. Due to its simplicity and [BTF
deduplication algorithm](/posts/btf-dedup/), BTF allows to achieve up to
**100x** reduction in size, when compared to DWARF. It is now practical to
have Linux kernel with always present embedded BTF type information at
runtime: just build the kernel with `CONFIG_DEBUG_INFO_BTF=y` option. Kernel’s
BTF is available to kernel itself and is now used to enhance BPF verifier’s
own capabilities beyond what the community imagined was possible just a year
ago (e.g., direct kernel memory reads without `bpf_probe_read()` are now
a thing).

More importantly for BPF CO-RE, kernel also exposes this self-describing
authoritative BTF information (defining, among other things, exact struct
layouts) through sysfs at `/sys/kernel/btf/vmlinux`. Try it for yourself,
with:

```shell
$ bpftool btf dump file /sys/kernel/btf/vmlinux format c
```

You'll get a compilable C header file (usually referred to as "**vmlinux.h**")
with all kernel types. And “all” does mean **all**, including those that are
never exposed through headers provided by the `kernel-devel` package!

### Compiler support

To enable BPF CO-RE and let BPF loader (i.e., libbpf) to adjust BPF program to
a particular kernel running on target host, Clang was extended with few
built-ins. They emit **BTF relocations** which capture a high-level
description of what pieces of information BPF program code intended to read.
If you were going to access `task_struct->pid` field, Clang would record that
it was exactly a field named "pid" of type “pid_t” residing within a `struct
task_struct`. This is done so that even if target kernel has a `task_struct`
layout in which “pid” field got moved to a different offset within
a `task_struct` structure (e.g., due to extra field added before “pid” field),
or even if it was moved into some nested anonymous struct or union (and this
is completely transparent in C code, so no one ever pays attention to details
like that), you’ll still be able to find it just by its name and type
information. This is called a **field offset relocation**.

It is possible to capture (and subsequently relocate) not just a field offset,
but other field aspects, like **field existence** or **size**. Even for
bitfields (which are notoriously "uncooperative" kinds of data in the
C language, resisting efforts to make them relocatable) it is still possible
to capture enough information to make them relocatable, all transparently to
BPF program developer.

### BPF loader (libbpf)

All the previous pieces of data (kernel BTF and Clang relocations) are coming
together and are processed by [libbpf](https://github.com/libbpf/libbpf),
which serves as a BPF program loader. It takes compiled BPF ELF object file,
post-processes it as necessary, sets up various kernel objects (maps,
programs, etc), and triggers BPF program loading and verification.

Libbpf knows how to tailor BPF program code to a particular running kernel on
the host. It looks at BPF program’s recorded BTF type and relocation
information and matches them to a BTF information provided by the running
kernel.  Libbpf resolves and matches all the types and fields, updates
necessary offsets and other relocatable data as needed to make sure that BPF
program’s logic is correctly functioning for a specific kernel on the host. If
everything checks out, you (BPF application developer) will get a BPF program,
"custom tailored" to a kernel on target host as if your program was
specifically compiled for it. But all that is achieved without paying the
overhead of distributing Clang with your application and performing
compilation in runtime on target host.

### Kernel

Amazingly, the kernel doesn’t need many changes to support BPF CO-RE. Due to
a good separation of concerns, after libbpf processed BPF program code, to
kernel it looks like any other valid BPF program code. It is indistinguishable
from a BPF program compiled right there on the host with up-to-date kernel
headers. This means that BPF CO-RE doesn’t require bleeding-edge kernel
functionality for a lot of its functionality and thus can be adapted more
widely and much sooner.

There will be some advanced scenarios that might require newer kernels, but
those should be rare. We’ll discuss such scenarios when explaining user-facing
mechanisms of BPF CO-RE in the next part, which goes into the details of
user-facing API of BPF CO-RE.

## BPF CO-RE: user-facing experience

Let's go through typical scenarios that real-world BPF applications have to
deal with and see how they are addressed with BPF CO-RE. As you’ll see below,
some portability issues (e.g., compatible struct layout differences) are
handled quite transparently and naturally, while others are handled more
explicitly, e.g., through `if`/`else` conditionals (as opposed to compile-time
`#ifdef`/`#else` constructs in BCC programs) and extra mechanisms provided by
BPF CO-RE.

### Getting rid of kernel header dependency

In addition to using kernel’s BTF information for field relocations, it’s also
possible to use it to generate a big header ("**vmlinux.h**") with all
internal kernel types and avoid dependency on system-wide kernel headers
altogether. You can get it with bpftool:

```shell
$ bpftool btf dump file /sys/kernel/btf/vmlinux format c > vmlinux.h
```

With `vmlinux.h` at your hands, there is no need for all those `#include
<linux/sched.h>`, `#include <linux/fs.h>`, etc, typically used in BPF
programs. You can now just `#include "vmlinux.h"` and forget about
`kernel-devel` package. This header contains all kernel types: those exposed
as part of UAPI, internal types available through `kernel-devel`, and some
more *internal kernel types not available anywhere else*. 

Unfortunately, BTF (as well as DWARF) doesn’t record `#define` macros, so some
common macros might be missing with `vmlinux.h`. Most commonly missing ones
might be provided as part of libbpf’s
[bpf_helpers.h](https://github.com/libbpf/libbpf/blob/master/src/bpf_helpers.h)
(kernel-side "library", provided by libbpf).

### Reading kernel structure’s fields

Most common and typical situation is just reading a field from one of many
kernel structs. Let’s say you want to read task_struct’s pid. This is easy and
simple with BCC:

BCC way:

```c
pid_t pid = task->pid;
```

BCC will conveniently rewrite `task->pid` into a call to `bpf_probe_read()`,
which is great (though sometimes might not work, depending on the complexity
of an expression used). With libbpf, because it doesn’t have BCC’s
code-rewriting magic at its disposal, there are a few ways you can achieve the
same result.

If you are using recently added
[`BTF_PROG_TYPE_TRACING`](https://patchwork.ozlabs.org/project/netdev/list/?series=139747&state=*)
BPF programs, then you have a smartness of BPF verifier on your side, which
now understands and tracks BTF types natively and allows you to follow
pointers and read kernel memory directly (and safely), avoiding
`bpf_probe_read()` calls, so you don’t need compiler rewriting magic to get
the same nice and familiar syntax.

Libbpf + BPF_PROG_TYPE_TRACING way:

```c
pid_t pid = task->pid;
```

Pairing this functionality with BPF CO-RE to support portable (i.e.,
relocatable) field reads, you’ll have to enclose this code into
`__builtin_preserve_access_index` compiler built-in:

BPF_PROG_TYPE_TRACING + BPF CO-RE way:

```c
pid_t pid = __builtin_preserve_access_index(({ task->pid; }));
```

That’s it. It will work as you expect, but will be portable between different
kernel versions. But given the bleeding-edge recency of
`BPF_PROG_TYPE_TRACING`, you might not have the luxury of using it yet, so you
have to use `bpf_probe_read()` explicitly instead.

Non-CO-RE libbpf way:

```c
pid_t pid; bpf_probe_read(&pid, sizeof(pid), &task->pid);
```

Now, with CO-RE+libbpf there are two ways to do this. One, directly replacing
`bpf_probe_read()` with `bpf_core_read()`:

```c
pid_t pid; bpf_core_read(&pid, sizeof(pid), &task->pid);
```

`bpf_core_read()` is a simple macro which passes all the arguments directly to
`bpf_probe_read()`, but it also makes Clang record field offset relocation for
third argument (`&task->pid`) by passing it through
`__builtin_preserve_access_index()`. So the last example is actually just
this, under the hood:

```c
bpf_probe_read(&pid, sizeof(pid), __builtin_preserve_access_index(&task->pid));
```

Alas, these `bpf_probe_read()`/`bpf_core_read()` calls can get old pretty
quickly, especially if you deal with a bunch of structs linked together
through pointers. For instance, to get inode number for current process’
executable binary, you’d have to do something like this with BCC:

```c
u64 inode = task->mm->exe_file->f_inode->i_ino;
```

With vanilla `bpf_probe_read()`/`bpf_core_read()`, this will turn into 4 calls
with an extra temporary variable to store value for all those intermediate
pointers, just to finally get to `i_ino` field. Fortunately, with BPF CO-RE,
there is a helper macro that will allow you to get almost BCC-like usability,
yet doesn’t require code-rewriting magic at all: 

BPF CO-RE way:

```c
u64 inode = BPF_CORE_READ(task, mm, exe_file, f_inode, i_ino);
```

Alternatively, if you already have a variable that you want to read into, you
can do the following and avoid extra intermediate variable (hidden inside
`BPF_CORE_READ`):

```c
u64 inode;
BPF_CORE_READ_INTO(&inode, task, mm, exe_file, f_inode, i_ino);
```

There is a corresponding `bpf_core_read_str()` which is a drop-in replacement
for `bpf_probe_read_str()`. There is also a `BPF_CORE_READ_STR_INTO()` macro,
which work similarly to `BPF_CORE_READ_INTO()`, but will perform
`bpf_probe_read_str()` call for the last field.

It’s also possible to check if field exists in target kernel, using aptly
named `bpf_core_field_exists()` macro, and do something differently based on
whether it’s there or not:

```c
pid_t pid = bpf_core_field_exists(task->pid) ? BPF_CORE_READ(task, pid) : -1;
```

Additionally, it’s possible to capture any field’s size, using
`bpf_core_field_size()` macro, for cases where you can’t guarantee that the
size of the data you are working with is not changing between kernel versions:

```c
u32 comm_sz = bpf_core_field_size(task->comm); /* will set comm_sz to 16 */
```

On top of that, for rare (but extremely painful to support across kernels)
cases when you have to read a bitfield out of a kernel struct, there are
special `BPF_CORE_READ_BITFIELD()` (using direct memory reads) and
`BPF_CORE_READ_BITFIELD_PROBED()` (relying on `bpf_probe_read()` calls)
macros. They abstract away otherwise gory and painful details of extracting
bit fields while preserving portability across kernel versions:

```c
struct tcp_sock *s = ...;

/* with direct reads */
bool is_cwnd_limited = BPF_CORE_READ_BITFIELD(s, is_cwnd_limited);

/* with bpf_probe_read()-based reads */
u64 is_cwnd_limited;
BPF_CORE_READ_BITFIELD_PROBED(s, is_cwnd_limited, &is_cwnd_limited);
```

Field relocations and related macros are a work horse of BPF CO-RE. They cover
lots of real-world use cases and will get you quite far with your BPF
programs.

### Dealing with kernel version and configuration differences

In some cases, BPF program has to deal with kernel differences that go beyond
slight structural changes of common kernel structures. Sometimes the field
gets renamed, so for all the purposes it becomes a completely different field
(but with the same meaning). And vice versa, while field stays the same, its
meaning might change. E.g., starting some time after 4.6 kernel,
`task_struct`’s `utime` and `stime` fields were switched from accounting in
jiffies to nanoseconds, so you’d have to do conversion differently before and
after. Other times, data you’d like to extract is present in some kernel
configurations, but compiled out on others. And there might be many other
scenarios, in which it’s just impossible to have a single common type
definition that will fit all kernels

For cases like that, BPF CO-RE provides two complementary solutions:
libbpf-provided **extern Kconfig variables** and **struct flavors**.

Libbpf-provided externs are a simple idea. BPF program can define an extern
variable with a well-known name (e.g., "*LINUX_KERNEL_VERSION*" to extract
a running kernel version) or a name that matches one of Kconfig’s keys (e.g.,
“*CONFIG_HZ*” to get the value of HZ that kernel was built with) and libbpf
will do its magic to set everything up in such a way that your BPF program can
use such extern variables as any other global variable. These variables will
have correct values, matching the active kernel your BPF program is executed
in.  Additionally, BPF verifier will track those variables as known constants
and will be able to use them for advanced control flow analysis and dead code
elimination. Check out an example on how thread’s CPU user time extraction can
be done with BPF CO-RE:

```c
extern u32 LINUX_KERNEL_VERSION __kconfig;
extern u32 CONFIG_HZ __kconfig;

u64 utime_ns;

if (LINUX_KERNEL_VERSION >= KERNEL_VERSION(4, 11, 0))
    utime_ns = BPF_CORE_READ(task, utime);
else
    /* convert jiffies to nanoseconds */
    utime_ns = BPF_CORE_READ(task, utime) * (1000000000UL / CONFIG_HZ);
```

The other mechanism, *struct flavors*, helps with cases where different
kernels have incompatible types, so it’s just impossible to compile a single
BPF program for both kernels with a single common struct definition. As
a somewhat contrived example, let’s see how struct flavors can be used to
extract `fs`/`fsbase` (which got renamed, as mentioned above) to do some
thread-local data processing:

```c
/* up-to-date thread_struct definition matching newer kernels */
struct thread_struct {
    ...
    u64 fsbase;
    ...
};

/* legacy thread_struct definition for <= 4.6 kernels */
struct thread_struct___v46 { /* ___v46 is a "flavor" part */
    ...
    u64 fs;
    ...
};

extern int LINUX_KERNEL_VERSION __kconfig;
...

struct thread_struct *thr = ...;
u64 fsbase;
if (LINUX_KERNEL_VERSION > KERNEL_VERSION(4, 6, 0))
    fsbase = BPF_CORE_READ((struct thread_struct___v46 *)thr, fs);
else
    fsbase = BPF_CORE_READ(thr, fsbase);
```

In this example, BPF application defines "legacy" `struct thread_struct`
definition for <= 4.6 kernels as `struct thread_struct___v46`. Three
underscores in type name and everything after it are considered to be
a “flavor” of this struct. This flavor part will be ignored by libbpf, meaning
that this type definition will still be matched against `struct thread_struct`
of the actual running kernel when performing necessary relocations. Such
convention allows to have multiple alternative (and incompatible) definitions
for the same kernel type in a single C program and be able to pick the most
appropriate one in runtime (through, say, kernel version-specific logic as in
the above example) and use **type cast to a struct flavor** to extract
necessary fields.

Without struct flavors, it would be impossible to really have a "compile once"
program that could run on multiple kernels in cases like above. You’d need
`#ifdef`’ed source code, compiled into two separate BPF program variants, with
appropriate variant picked manually by control application in runtime. All
this would be just unnecessary added complexity and pain. While not
transparently, BPF CO-RE allows to solve this problem with familiar C code
constructs even for such advanced scenarios.

### Altering behavior based on user-provided configuration

Knowing kernel version and configuration in BPF program is still sometimes
insufficient to make the right decision on what and how to get data out of
a kernel. In such cases, user-space control application might be the only
party knowing what exactly needs to be done and which features need to be
enabled or disabled. This is normally communicated through some sort of
configuration data, shared between user-space and BPF program. One way to
achieve that today without relying on BPF CO-RE is through using BPF map as
a container for configuration data. BPF program performs BPF map lookups to
extract configuration and alters its control flow based on this config.

There are major downsides to such approach:

* Runtime overhead of doing map look up each time BPF program tries to get
  configuration value. This can add up quickly and be quite prohibitive in
  some high-performance BPF applications.

* Config value, while immutable and read-only after BPF program starts, is
  still treated by BPF verifier as an unknown black box value during the
  verification phase. This means that verifier can’t prune dead code and
  perform other advanced code analysis. This makes it impossible to have
  configurable parts of BPF program logic that will use bleeding-edge
  features, supported on new kernels only, without breaking the same program
  when running on older kernels. This is due to BPF verifier having to
  pessimistically assume configuration can be anything and that this "unknown"
  functionality might be called anyways, despite user configuration clearly
  making that impossible.

The solution to such a (admittedly complicated) use case is through using
read-only global data. It is set once by a control application before the BPF
program is loaded into a kernel. From the BPF program side, this looks like
a normal global variable access. There won’t be any BPF map lookup overhead
– global variables are implemented as a direct memory access. Control
application side will set initial configuration values before BPF program is
loaded, so by the time BPF verifier will get to validation of a program,
configuration values will be well known and read-only. This will allow BPF
verifier to track them as known constants and use its advanced control flow
analysis to perform dead code elimination.

So, for the above example, on older kernel BPF verifier will prove that, say,
unknown BPF helper is never going to be used and will eliminate that code
altogether. On newer ones, though, application-provided configuration is going
to be different and will allow the use of new fancy BPF helper and this logic
will get successfully validated by BPF verifier. The following BPF code
example should make this more clear:

```c
/* global read-only variables, set up by control app */
const bool use_fancy_helper;
const u32 fallback_value;

...

u32 value;
if (use_fancy_helper)
    value = bpf_fancy_helper(ctx);
else
    value = bpf_default_helper(ctx) * fallback_value;
```

From the user-space side, application will be able to easily provide this
configuration through BPF skeleton. BPF skeleton discussion is beyond the
scope of this article, but see the [runqslower
tool](https://github.com/torvalds/linux/tree/master/tools/bpf/runqslower) in
kernel repository for a good show case of using it to simplify BPF
application.

### Recap

BPF CO-RE’s goal is to help BPF developers to solve simple portability
problems (like reading struct fields) in a simple way and make it still
possible (and tolerable, if not trivial) to address complicated portability
problems (like incompatible data structure changes, complicated user-space
controlled conditions, etc). This allows BPF developers to stay within the
"Compile Once — Run Everywhere" paradigm. This is achieved through combining
a few BPF CO-RE building blocks, described above:

* `vmlinux.h` eliminates dependency on kernel headers;

* field relocations (field offsets, existence, size, etc) make data extraction
  from kernel portable;

* libbpf-provided Kconfig extern variables allow BPF programs to accommodate
  various kernel version- and configuration-specific changes;

* when everything else fails, app-provided read-only configuration and struct
  flavors are an ultimate big hammer to address whatever complicated scenario
  application has to handle.

Not all of those features of CO-RE are going to be needed to successfully
write, deploy, and maintain portable BPF programs, but when you will need
them, they are going to be there and will help you solve your problem in the
simplest way possible. All that while still providing a good usability and
familiar workflow of compiling C code into binary and distributing lightweight
binaries around. There is no more need to drag along a heavy-weight compiler
library and pay precious runtime resources for runtime compilation. There is
no more need to catch trivial compilation errors in runtime, either.

## BPF CO-RE as of 2021

As of 2021, BPF CO-RE is now a mature technology used across a wide variety of
projects.

At Facebook, BPF CO-RE powers multiple production BPF-based applications
successfully, handling both simple cases of changing field offsets and much
more advanced cases of kernel data structures being removed, renamed, or
completely changed. All within a single compiled-once BPF application.

Since the introduction of BPF CO-RE, [more than
25](https://github.com/iovisor/bcc/blob/master/libbpf-tools/Makefile#L18) BCC
tools got converted to libbpf and BPF CO-RE (check out
[libbpf-tools](https://github.com/iovisor/bcc/blob/master/libbpf-tools)). As
more and more Linux distributions enable kernel BTF by default (see [the
list](https://github.com/libbpf/libbpf#bpf-co-re-compile-once--run-everywhere)),
BPF CO-RE-based tools become more widely applicable and more efficient
replacements for heavy-weight Python-based BCC tools. And that's the way
forward, as emphasized by Brendan Gregg in his ["BPF binaries: BTF, CO-RE, and
the future of BPF perf
tools"](http://www.brendangregg.com/blog/2020-11-04/bpf-co-re-btf-libbpf.html)
blog post.

BPF CO-RE is gaining a rapid adoption across various areas, powering efficient
BPF applications. It is used in tracing and performance monitoring, security
and audit, even networking BPF applications. Anywhere from tiny embedded
systems to huge production servers.
[libbpf-bootstrap](https://github.com/libbpf/libbpf-bootstrap) project was
created to simplify starting BPF development with libbpf and BPF CO-RE. So
make sure to check out ["Building BPF applications with
libbpf-bootstrap"](/posts/libbpf-bootstrap) blog post, if you are interested.

On the more technical level, in addition to already described field
relocations, BPF CO-RE has gained support for:
  * type size and existences relocations. When types are added, removed, or
    renamed, it's important to be able to detect this and adjust BPF
    application logic accordingly. See `bpf_core_type_exists()` and
    `bpf_core_type_size()` macros, provided by libbpf.
  * enum relocations (existence and value). Some internal, non-UAPI kernel
    enums do change across kernel versions, or even depend on exact config
    used for kernel compilation (e.g., `enum cgroup_subsys_id`, see [BPF
    selftest](https://github.com/torvalds/linux/blob/master/tools/testing/selftests/bpf/progs/profiler.inc.h#L260-L262)
    dealing with it), making it impossible to hard-code any specific value
    reliably. Enum relocations (`bpf_core_enum_value_exists()` and
    `bpf_core_enum_value()` macros, provided by libbpf) allow to check
    existence of a specific enum value and capture its value. One important
    application of this is detection of availability of new BPF helpers, and
    falling back to older one, if kernel is too old.

When compiled with read-only global variables, both are indispensable to
perform simple and reliable kernel feature detection from BPF side.

There is now also a dedicated [BPF CO-RE reference
guide](/posts/bpf-core-reference-guide/) post with practical guidance to all
BPF CO-RE features and tips on how to apply them when developing real-world
BPF application.

## References

1. BPF CO-RE presentation from LSF/MM2019 conference:
[summary](http://vger.kernel.org/bpfconf2019.html#session-2),
[slides](http://vger.kernel.org/bpfconf2019_talks/bpf-core.pdf).

2. Arnaldo Carvalho de Melo’s presentation ["BPF: The Status of
BTF"](http://vger.kernel.org/~acme/bpf/devconf.cz-2020-BPF-The-Status-of-BTF-producers-consumers/#/29)
dives deep into BPF CO-RE and dissects the runqslower tool quite nicely.

3. [BTF deduplication algorithm](/posts/btf-dedup/).

