+++
date = 2022-08-22
title = "Journey to libbpf 1.0"
description = '''
The road to libbpf 1.0 was long, but we've finally arrived!
What went into libbpf 1.0. What are the main breaking changes.
What exciting new features were added. And great lengths libbpf
goes to to ensure best user experience when dealing with
a complicated world of BPF.
'''

[extra]
toc = false

[taxonomies]
categories = ["BPF"]
tags = ["bpf", "libbpf"]
+++

<p style="text-align:center;">
<img src="/images/libbpf-textured-header-tight.jpg" width="100%"
     title="Libbpf logo"
     alt="Libbpf logo" />
</p>

## [Libbpf 1.0 release](https://github.com/libbpf/libbpf/releases/tag/v1.0.0) is here!

It has been a pretty long journey to get to libbpf 1.0, so to commemorate this
event I decided to write a post that would highlight main features and API
changes (especially breaking ones) and show large amount of work done by
libbpf community that went into improved usability and robustness of libbpf
1.0.

The idea to clean up and future-proof and shed some organically grown over
time cruft was born almost 1.5 years ago. At that time libbpf was in active
development for a while already and all the main conventions and concepts on
how to structure and work with BPF programs and maps, collectively grouped
into "BPF objects", were more or less formulated and stabilized, so it felt
like a good time to start cleaning up and setting up a good base for the
future without dragging along suboptimal early decisions.

The journey started with public discussion on what should be changed and
dropped from libbpf APIs to improve library's usability and long term
maintainability. The resulting plan for all the backwards-incompatible
changes was recorded in ["Libbpf: the road to v1.0"](https://github.com/libbpf/libbpf/wiki/Libbpf:-the-road-to-v1.0)
wiki page, initial [46 tracking issues](https://github.com/libbpf/libbpf/issues?page=1&q=is%3Aissue+label%3Alibbpf-1.0)
were filed, and since then libbpf community have been diligently working
towards libbpf 1.0 release.

During this time, libbpf went through five minor version releases
([v0.4](https://github.com/libbpf/libbpf/releases/tag/v0.4.0) –
[v0.8](https://github.com/libbpf/libbpf/releases/tag/v0.8)). We've developed
a transitioning plan and mechanisms for early adoption of new (stricter)
behaviors, deprecated lots of APIs, and for many of those we added better
and cleaner alternatives.

But it wasn't just about removing stuff. A lot of new features were
implemented and added, closing some of the long standing gaps in
functionality (e.g., as compared to [BCC](https://github.com/iovisor/bcc)),
making many typical scenarios simpler for end users, as well as adding enough
control and flexibility to support more advanced scenarios. A lot of work
went into helping users to deal with unavoidable differences between
different kernel versions, Clang versions, and sometimes even special Linux
distro quirks. Whenever possible this was done in a completely transparent
and robust way to free end users from such distracting details.

We also expanded and improved support for various CPU architectures
beyond the most popular and actively used in production x86-64
(amd64) architecture.

Of course, lots of bugs were reported and fixed by BPF community as we went
along, together making a better BPF loader library for everyone.

Lastly, we've started an effort of improving libbpf documentation. Automated
pipeline was setup and we now host libbpf docs
[here](https://libbpf.readthedocs.io/en/latest/api.html). This is an ongoing
effort and could always use more active contributions, of course. I'm sure
with time and community effort we'll get to the point where libbpf
documentation will be exemplary and a self-sufficient way for BPF newbies to
start in an exciting BPF world. But while we are working towards that,
we've also started a companion
[libbpf-bootstrap](https://github.com/libbpf/libbpf-bootstrap) repository
with simple and clean examples of using libbpf to build BPF applications of
various kinds, which seem to be quite popular with users.

Libbpf 1.0 marks the release in which all the deprecated APIs and features are
physically removed from the code base and new stricter behaviors are mandatory and not an opt-in anymore. This both signifies maturity of libbpf and sets it
up for better future functionality with less maintenance burden for future backwards-compatible v1.x libbpf versions.

Oh, and, as you might have noticed already, libbpf now has its own logo to
give the project a bit more personality! I hope you like it!

On behalf of [libbpf project](https://github.com/libbpf/libbpf) I'd like to
thank all the [contributors](https://github.com/libbpf/libbpf/graphs/contributors)
to the overall **libbpf ecosystem**, which, besides *libbpf* itself,
includes also:
  - [BPF selftests](https://github.com/torvalds/linux/tree/master/tools/testing/selftests/bpf);
  - [BPF CI](https://github.com/libbpf/ci) and related infrastructure;
  - [bpftool](https://github.com/libbpf/bpftool);
  - [libbpf-bootstrap](https://github.com/libbpf/libbpf-bootstrap);
  - [libbpf-tools](https://github.com/iovisor/bcc/tree/master/libbpf-tools)
    project of many libbpf-based observability tools, inspiring multitude of libbpf features;
  - [libbpf-rs](https://github.com/libbpf/libbpf-rs) Rust library companion to libbpf;
  - [libbpf-sys](https://github.com/libbpf/libbpf-sys) Rust low-level bindings.

**Thanks a lot for your continuing help and contributions, without which
libbpf wouldn't be were it is today!**

To make this not just a celebratory post, in the rest of it I'll try to
describe major breaking changes users should be aware of and summarize
various new additions to libbpf functionality. Separately, I'll try to
highlight libbpf functionality that most people might not be aware of,
because it generally does it's job very well and stays out of the way,
shielding BPF users from pain or handling various quirks of kernels,
compilers, and Linux distros. This is not intended as an exhaustive list of
features and changes, it's just things I chose to highlight, so please don't
be offended if I missed or skipped a favorite feature of yours.

## Breaking changes

### Error reporting streamlining

Libbpf started out as internal Linux kernel project and inherited some of
kernel-specific traits and conventions. One of the most prominent and
pervasive thoughout API was a convention to return error code embedded into
the pointer. Almost all pointer-returning APIs in libbpf on failure would
return error code (one of [many](https://github.com/torvalds/linux/blob/master/include/uapi/asm-generic/errno-base.h)
`-Exxx` [values](https://github.com/torvalds/linux/blob/master/include/uapi/asm-generic/errno.h)) encoded as a pointer. And while convenient for those
in the know, it was way too easy to forget about this and perform natural
but incorrect `NULL` error check.

For integer-returning APIs, the situation with returning errors wasn't completely straightforward as well. Some APIs would return `-1` on error and set `errno` to actual `Exxx` error code, just like typical `libc` API would do. Other APIs would return `-Exxx` directly as return value, usually not caring about setting `errno` at all. And what's worse, some APIs did both depending on specific errors. Quite a mess, indeed.

Libbpf 1.0 breaks apart from *internal kernel convention* of encoding error
code in pointers and sets up clear and consistently followed error reporting
rules across any API:
  - any pointer-returning API returns `NULL` on error, and sets `errno` to
    actual (positive) `Exxx` error code;
  - any other API that might fail returns (negative) `-Exxx` error code
    directly as an `int` result **and** sets (positive) `errno` to `Exxx`.

This allows to consistently rely on `errno` for extracting error code across
any type of API. It also makes an intuitive and natural `NULL` check *safe
and correct*. And for integer-returning API, user can directly capture
`-Exxx` error code from return result (so no more useless `-1` error codes,
confusingly translated as `-EPERM`) and not be too careful about capturing
and not clobbering `errno`.

Note also, that any destructor-like API in libbpf (e.g., `bpf_object__close
()`, `btf__free()`, etc) always safely accepts `NULL` pointers, so you don't
need to guard them with extra `NULL` checks. It's a small, but nice,
usability improvement that adds up in a big code base.

### BPF program SEC() annotation streamlining

Libbpf expects BPF programs to be annotated with `SEC()` macro, where string
argument passed into `SEC()` determines BPF program type and, optionally,
additional attach parameters, like kernel function name to attach to for
kprobe programs or hook type for cgroup programs. This `SEC()` definition
ends up being recorded as ELF *section name*.

Initially libbpf only allowed one BPF program for each unique
`SEC()` definition, so you couldn't have two BPF programs with, say,
`SEC("xdp")` annotation. Which led to two undesirable consequences:

  - such section names were used as *unique identifiers* for BPF programs.
    This convention made it all the way to generic tools like `bpftool` and
    `iproute2` which expected section names to identify specific BPF program
    out of potentially many programs in a single BPF ELF object file;

  - people started adding random suffixes (e.g., `SEC("xdp_my_prog1")`) to
    create a unique identifier both as a convenience and as a work around
    for the single BPF program per `SEC()` limitation.

Both were problematic. First, since those ancient times libbpf started
supporting as many BPF programs with the same `SEC()` annotation as user
needs. Second, non-uniform use of `SEC()` annotation was harming overall
ecosystem, especially confusing newcomers as to what exact `SEC()` annotation
is correct and appropriate.

*Libbpf 1.0 is breaking with this legacy and sloppy approach*. Libbpf
normalized a set of supported `SEC()` annotations and doesn't support
random suffixes anymore. For the above example, no matter how many XDP
programs one has in BPF object file, all of them should be annotated
strictly as `SEC("xdp")`. Not `SEC("xdp1")`, or `SEC("xdp_prog1")`,
or `SEC("xdp/prog1")`.

As for unique identification of BPF program for generic tools, **please use BPF program name (i.e., its *C function name*) to identify BPF programs uniquely,** if your application or tool isn't doing that already.

Note also, that as of libbpf 1.0, BPF program *name* is used when **pinning
BPF program**, while previously BPF program's (non-unique) *section name* was
used. This is another breaking change to keep in mind, if you rely on
libbpf's pinning logic.

Another important change related to `SEC()` handling is that for a lot of BPF
programs that used to require to always specify attach target (e.g.,
`SEC("kprobe/__x64_sys_bpf")`), this requirement has been lifted and it's
completely supported to specify just BPF program **type** (i.e.,
`SEC("kprobe")`). In the latter case, BPF program auto-attachment through BPF
skeleton or through `bpf_program__attach()` won't be supported, but otherwise
BPF program will be marked with correct program type and loaded into kernel
during BPF object loading phase.

### API extensibility through OPTS framework

Libbpf 1.0 got rid of a bunch of APIs that were not extendable without
breaking backwards (i.e, *newer application* dynamically linked
against *older libbpf*) and/or forward (i.e., *older application* dynamically
linked against *newer libbpf*) compatibility. Such APIs historically were
using fixed-sized structs to pass extra arguments, but we've learned the hard
way that this approach doesn't work well.

To solve these problems, we've developed a so-called `OPTS`-based framework,
and a lot of APIs (e.g., `bpf_object__open_file()`) accept optional
`struct xxx_opts` arguments. Use of `OPTS` approach lets libbpf deal with
backwards and forward compatibility transparently without relying on
cumbersome complexities of ELF symbol versioning or defining entire families
of very similar APIs, each differing just slightly from each other to
accommodate some new optional argument. Please utilize `LIBBPF_OPTS()`
macro to simplify instantiation of such `OPTS` structs, e.g.:

```c
char log_buf[64 * 1024];
LIBBPF_OPTS(bpf_object_open_opts, opts,
    .kernel_log_buf = log_buf,
    .kernel_log_size = sizeof(log_buf),
    .kernel_log_level = 1,
);
struct bpf_object *obj;

obj = bpf_object__open_file("/path/to/file.bpf.o", &opts);
if (!obj)
    /* error handling */
...
```

### Other API clean ups, renames, removals, etc

We used libbpf 1.0 milestone to clean up libbpf API surface significantly. We
removed, renamed, or consolidates a good chunk of APIs:

  - AF_XDP-related parts of libbpf (APIs from `xsk.h`) were consolidated into
    [libxdp](https://github.com/xdp-project/xdp-tools/tree/master/lib/libxdp).

  - We streamlined naming for getter and setter high-level APIs: getters
    don't add "get_" prefix (e.g., `bpf_program__type()`), while setters
    always have "set_" in the name (e.g., `bpf_program__set_type()`).

  - Entire family of low-level APIs for BPF program loading were consolidated
    into a single extensible `bpf_prog_load()` API; similarly for BPF map
    creation, `bpf_map_create()` API superseded a small zoo of six very
    similar APIs; similar sort of API normalization was done for other APIs.

  - Support for legacy BPF map definitions in `SEC("maps")` was completely
    dropped. It was original and limited way to define BPF maps with no
    clean mechanism to provide additional key and value BTF type information
    associated with BPF map. We've long since switched to using BTF-defined
    map definitions defined in `SEC(".maps")`, please use them instead, if
    you haven't done so already.

  - A whole set of very niche and specialized APIs that were only used by
    a handful of applications (like Linux's `perf` tool, or BCC framework)
    were moved or removed, giving more freedom to change libbpf internals. In
    turn, libbpf gained more general APIs that allow to implement complex
    scenarios more generically (e.g., support for custom `SEC()`
    annotations) and covered existing niche use cases well. Typically, users
    are unlikely to ever notice this, but if you find some APIs missing,
    please reach out at [BPF mailing list](https://lore.kernel.org/bpf/).

To familiarize yourself with finalized set of libbpf 1.0 APIs, please consult
the following public headers:
  - user-space APIs:
    - [libbpf.h](https://github.com/libbpf/libbpf/blob/master/src/libbpf.h);
    - [bpf.h](https://github.com/libbpf/libbpf/blob/master/src/bpf.h);
    - [btf.h](https://github.com/libbpf/libbpf/blob/master/src/btf.h);
  - BPF-side APIs:
    - [bpf_helpers.h](https://github.com/libbpf/libbpf/blob/master/src/bpf_helpers.h);
    - [bpf_tracing.h](https://github.com/libbpf/libbpf/blob/master/src/bpf_tracing.h);
    - [bpf_core_read.h](https://github.com/libbpf/libbpf/blob/master/src/bpf_core_read.h);
    - [usdt.bpf.h](https://github.com/libbpf/libbpf/blob/master/src/usdt.bpf.h);
    - [bpf_endian.h](https://github.com/libbpf/libbpf/blob/master/src/bpf_endian.h).

## Graceful degradation and taking care of kernel differences

While new features are exciting and most visible, I think it's important to appreciate small and mostly invisible things that libbpf is doing for the user to simplify their life and hide as many different quirks and limitations of different (especially older) kernel and Clang versions (and even some gotchas of particular Linux distros), as possible. It is unsung part of libbpf and a lot of thought and collective work went into making sure that things that can be abstracted away and handled transparently without user involvement "just work". Here's a list of just some of the stuff that libbpf is doing on behalf of users, so they don't have to.

  - `bpf_probe_read_{kernel, user}()` is automatically "downgraded" to
    `bpf_probe_read()` on old kernels, so user should freely use
    `bpf_probe_read_kernel()` in their BPF code and not worry about backwards
    compatibility problems.
  - `bpf_printk()` macro is conservatively using `bpf_trace_printk()` BPF
    helper, supported since oldest kernels, if possible, but automatically
    and transparently switching to newer and more powerful
    `bpf_trace_vprintk()` BPF helper, available only on newer kernels, if user needs to log more arguments, so same `bpf_printk()` macro can be used
    universally with largest possible backwards kernel compatibility.
  - libbpf will automatically set `RLIMIT_MEMLOCK` to infinity (user can
    override the limit), but only if kernel is old enough to require that. On
    newer kernels, `RLIMIT_MEMLOCK` is not used and so doesn't have to be
    increased to do anything useful with BPF. `RLIMIT_MEMLOCK` has been a
    bump in the road for lots of BPF newbies, and now it's taken care of by
    libbpf automatically (and only if necessary).
  - libbpf is taking care of automatically "sanitizing" BPF object's BTF
    information so it's not rejected by older kernels. User doesn't have to
    worry about which BTF features kernel and compiler support and whether they are compatible. BTF will be automatically sanitized to satisfy
    Linux kernel's level of BTF support. One less thing to worry about.
  - libbpf APIs overall are smart enough to drop optional features,
    if kernel is too old to support them and features themselves are not
    critically important for correct functioning of BPF applications
    (e.g., BPF program/map name for `bpf_prog_load()`/`bpf_map_create()`;
    optional BTF/BTF.ext data associated with BPF program/map).
  - libbpf will post-process BPF verifier logs to augment it with more useful
    and relevant information for well-known and common situations
    (e.g., extending verifier log with information about failed BPF CO-RE
    relocation). This significantly improves usefulness of verification
    failure log.
  - for cases when it's safe to do so, libbpf will auto-adjust BPF map
    parameters, if necessarty. E.g., BPF ringbuf size will be rounded up to a
    proper multiple of page size on host system. Or key/value BTF type
    information will be dropped, if libbpf believes that specific BPF map
    doesn't support specifying BTF type ID; it will still calculate and
    provide correct key/value sizes, though.
  - on BPF side, macros like `BPF_KPROBE()`, `BPF_PROG()`, `BPF_USDT()`,
    `BPF_KSYSCALL()` were developed to make it much easier and ergonomic
    to write tracing BPF programs. Use of such macro both improves code
    readability and maintainability, as well as hides some of the nasty
    kernel- and architecture-specific quirks, so that typical user doesn't
    have to care or know about them for typical use cases.

This is just a few examples of what goes on behind the scenes in libbpf in the name of better user experience. Of course, not all kernel- or architecture-specific differences can be hidden and handled automatically, but whenever it can be, libbpf strives to do it transparently, efficiently *and* correctly.

## New functionality

Libbpf is developed in lockstep with Linux kernel BPF support. This was the
case before and is going to be the case going forward. Any new BPF kernel
feature gets necessary APIs and overall support in libbpf at the time of that
feature landing upstream into Linux kernel. This means features like unstable
BPF helpers (exposed as `extern __ksym` functions), safe typed kernel
pointers (`__kptr` and `__kptr_ref`), and lots of other features are ready to
be used through libbpf from the day they are accepted into the kernel, no
matter how bleeding edge they are.

This is par for the course for libbpf, so instead of concentrating on all the
new kernel functionality supported and exposed through libbpf, I'll highlight
new functionality that required additional purely user-space code to be added
on the way to libbpf 1.0.

### C language constructs support

Libbpf has come a long way since its early days in terms of supporting all the
typical C language features one would expect from user-space code base:
  - There are no restrictions on number of BPF programs and their `SEC
    ()` annotation uniqueness.
  - Users don't have to ``__always_inline`` their C functions (a.k.a. BPF
    subprograms) anymore, libbpf is smart enough to figure out which ones are
    used by each BPF program and perform code transformations to make sure
    BPF verifier gets correct final BPF assembly instructions.
  - C global variables are supported as well and provide a tremendous
    usability improvements for a lot of typical use cases (e.g., configuring
    BPF program logic from user-space; see below on BPF skeleton as well);
  - **Static linking of object files is now supported.** There is no more
    restrictions on keeping entire BPF-side logic within a single `.bpf.c`
    file. Libbpf implements BPF static linker functionality and allows to
    compile each individual `.bpf.c` file separately and then link them all
    together into a final `.bpf.o` file. Static linking is normally performed
    through `bpftool gen object` command, but it is also available as public
    APIs for programmatic use in more advanced applications. Static linking
    means that **BPF static libraries** are now possible and supported! This
    allows to structure user's application in the way that makes sense for
    long-term maintainability and code reuse, instead of cramming all the
    code into single text file (even if through `#include`-ing `.c` files as
    if they were C headers). This also means `extern` subprograms, maps, and
    variables declarations are supported and work as one'd expect with usual
    user-space C application.

Beyond that, BPF skeleton improves logistics of deploying, loading and
interacting with BPF ELF object files. BPF skeleton allows to embed final BPF
object file for ease of distribution, load it at runtime, set and configure
all the programs, maps, *and global variables* from user-space conveniently
to prepare BPF object for loading it into the kernel, and afterward to keep
interfacing with it at runtime. If you haven't tried BPF skeleton, please
consider giving it a go, it quite profoundly changes how one structures and
interacts with their BPF-side functionality in BPF application.

It's been a gradual work towards supporting all the typical constructs of
user-space C code and by libbpf 1.0 there shouldn't be many things that can't
be expressed with BPF-side C, as long as it is supported by BPF verifier.

### Improved customizability by user

Lots of small features and APIs were added and implemented to allow users more
precise control over which parts of their BPF applications are loaded (for BPF
programs) or created (for BPF maps). With the use of `SEC("?...")` pattern
it's possible to opt-out from auto-loading BPF program declaratively in
BPF-side C code. Libbpf also provides equivalent programmatic controls with
`bpf_program__set_autoload()` API. For BPF maps, there is equivalent
`bpf_map__set_autocreate()` API to disable automatic creation of BPF map, if
that's undesirable at particular host system. And with
`bpf_program__set_autoattach()` it's possible to further control which BPF
programs will be automatically attached by BPF skeleton logic, at a granular
level.

Look for all the getters and setters for `bpf_object`, `bpf_program`, and
`bpf_map` to see what else can be tuned and adjusted at runtime to implement
the most portable BPF applications possible.

To improve debuggability, we've also added ability to flexibly capture BPF
verifier log into user-provided log buffers the help of `kernel_log_buf`,
`kernel_log_size`, and `kernel_log_level` open options, passed into
`bpf_object__open_file()` and `bpf_object__open_mem()`. To get even more
control, each BPF program's logi buffer can be set and retrieved with
`bpf_program__[set_]log_buf()` and `bpf_program__[set_]log_level()` APIs.
This control of BPF verifier log output comes very handy during active
development and troubleshooting.

As another example of libbpf allowing more control and customizability, libbpf
now supports defining custom `SEC()` handler callbacks to implement more
advanced application-specific BPF program loading scenarios. As a proof of
concept, this was used by `perf` application to support their
highly-specialized (and not supported by libbpf 1.0) way of defining custom
tracing BPF programs. This functionality is clearly for advanced users, but
it's good to have it when the need comes.

### Tracing support improvements

Using BPF for tracing kernel and user-space functionality is one of the main
BPF and libbpf use cases. Libbpf has significantly boosted its tracing
support since its early days.

**USDT tracing support.** A long-standing feature request for a while was
ability to trace [USDT](https://lwn.net/Articles/753601/)
(User Statically-Defined Tracing) probes with libbpf. This is now possible
and is an integral part of libbpf, adding support for
`SEC("usdt")`-annotated BPF programs. Check
[BPF-side API](https://github.com/libbpf/libbpf/blob/master/src/usdt.bpf.h),
and also note `BPF_USDT()` macro which allows to declaratively
define expected USDT arguments for ease of use and better readability.
Make sure to check a simple USDT BPF example from libbpf-bootstrap
([usdt.c](https://github.com/libbpf/libbpf-bootstrap/blob/master/examples/c/usdt.c)
and [usdt.bpf.c](https://github.com/libbpf/libbpf-bootstrap/blob/master/examples/c/usdt.bpf.c)).
Furthermore, thanks to the community, libbpf has support for *five* CPU
architectures from day one:
  - x86-64 (amd64);
  - x86 (i386);
  - s390x;
  - ARM64 (aarch64);
  - RISC V (riscv).

**Improved old kernel support.** Libbpf gained support for attaching kprobes,
uprobes, and tracepoint on much older kernels, falling back from more
modern and preferred `perf_event_open()`-based attachment mechanism, to
older tracefs-based one. All these details are transparent to user and
are hidden behind standard `bpf_link` interface. Just make sure to call
`bpf_link__destroy()` to detach and clean up the system.

**Syscall tracing support.** Tracing kernel syscalls are trickier than a
typical kernel function due to differences between host architectures and
kernel versions, as the exact naming of kernel functions and use of a
syscall wrapper mechanism (see `CONFIG_ARCH_HAS_SYSCALL_WRAPPER` if you are
curious). Instead of expecting users to figure all this out (e.g., is it
`__x64_sys_close` or `__se_sys_close`? Is `PT_REGS_SYSCALL_REGS()`
indirection necessary or not?), libbpf now supports special
`SEC("ksyscall/<syscall>")` (and corresponding `SEC("kretsyscall")` for
retprobes) BPF programs. In a combination with `BPF_KSYSCALL()` macro
this allows to trace syscalls much more easily. There are various advanced
corner cases which still require user care, though, so please see
[documentation](https://libbpf.readthedocs.io/en/latest/api.html)
for `bpf_program__attach_ksyscall()` API for more details. But common
scenarios are covered and well supported.

**Improved uprobes.** User-space tracing (uprobes) with libbpf used to
require user to do pretty much all the hard work themselves: figuring out exact binary paths, calculating function offsets within target process, manually attaching BPF programs, etc. Not anymore! Libbpf now supports
specifying target function by name and will do all the necessary calculations
automatically. Additionally, libbpf is smart enough to figure out absolute
path to system-wide libraries and binaries, so just annotating your
BPF uprobe program as `SEC("uprobe/libc.so.6:malloc")` will
allow to auto-attach to `malloc()` in your system's C runtime library.
You still get full control, if you need to, of course,
with `bpf_program__attach_uprobe()` API. Check uprobe example in
libbpf-bootstrap ([uprobe.bpf.c](https://github.com/libbpf/libbpf-bootstrap/blob/master/examples/c/uprobe.c)
and [uprobe.c](https://github.com/libbpf/libbpf-bootstrap/blob/master/examples/c/uprobe.c)).

**Expanded set of supported architectures for kprobes** Thanks to community
expertise, libbpf supports quite a variety of different CPU architectures
when it comes to low-level tracing and fetching kernel function arguments.
For kprobes, here's the current list of architectures libbpf supports and for which it provides tracing helpers:
  - x86-64 (amd64);
  - x86 (i386);
  - arm64 (aarch64);
  - arm;
  - s390x;
  - mips;
  - riscv;
  - powerpc;
  - sparc;
  - arc.

### Networking support improvements

While tracing is perhaps the area that got the biggest boost in new features,
BPF networking support wasn't forgotten either.

Libbpf now provides its own dedicated API for creating TC (Traffic Control)
hooks and attaching BPF programs to them. Previously this was only possible
through either shelling out to external `iproute2` tool or implementing
custom netlink-based functionality on your own. Both can be a significant
burden for BPF networking applications. Now, with `bpf_tc_hook_create()`,
`bpf_tc_hook_destroy()`, `bpf_tc_attach()`, `bpf_tc_detach()`, and `bpf_tc_query()` APIs there is no need to take extra dependency on
`iproute2` just to attach BPF TC programs. All the batteries are included
with libbpf.

Similar in naming and spirit `bpf_xdp_attach()`, `bpf_xdp_detach()`, and
`bpf_xdp_query()` APIs abstract away netlink-based XDP attachment details.
Note that there is also a safer-by-default alternative `bpf_link`-based XDP
attachment API, `bpf_program__attach_xdp()`, which is a preferred way of
performing XDP attachment on newer kernels.

## Summary

It's been a long journey for libbpf to get to 1.0, but it was worth it. By
taking time to get here, with community help and involvement, we got more
well thought out, user friendly, and full-featured library. libbpf 1.0 now
provides a battle-tested foundation for building any kind of
BPF application. It also sets a good base for future libbpf releases with more
exciting functionality while backwards compatibility across minor
version releases, all while keeping maintainability in focus.

A big **"Thank you!"** goes to *hundreds* of contributors and bug reporters
across entire **libbpf family of projects** for all your work and support!
Congratulations on the long-awaited [v1.0](https://github.com/libbpf/libbpf/releases/tag/v1.0.0)!
