+++
date = 2021-05-25
title = "BPF tips & tricks: the guide to bpf_trace_printk() and bpf_printk()"
description = '''
The guide to using bpf_trace_printk() and bpf_printk() for debugging BPF
applications and logging extra information from BPF side to user-space. Tips
and tricks on how to use BPF CO-RE to perform feature detection and accommodate
kernel version differences in bpf_trace_printk() behavior.
'''

[taxonomies]
categories = ["BPF"]
tags = ["bpf", "bpf-tips"]
+++

Any non-trivial BPF program always needs some amount of debugging to get it
working correctly. Unfortunately, there isn't a BPF debugger yet, so the next
best thing is to sprinkle `printf()`-like statements around and see what's
going on in the BPF program. BPF equivalent of `printf()` is the
`bpf_trace_printk()` helper. In this blog post we'll look at how to use it,
what are its limitations, and how to work around them. I'll also describe a few
important changes that happened to `bpf_trace_printk()` over last few kernel
releases and how [BPF CO-RE](/posts/bpf-portability-and-co-re/) can be used to
detect and handle those changes.

# Preliminaries

I'll be using [libbpf-bootstrap](https://github.com/libbpf/libbpf-bootstrap)'s
`minimal` example as a base for all the examples. It has everything wired up
and triggers a simple BPF program that we’ll use to test `bpf_trace_printk()`
output. If you'd like to follow along, make sure to clone
[libbpf-bootstrap](https://github.com/libbpf/libbpf-bootstrap) and open
[minimap.bpf.c](https://github.com/libbpf/libbpf-bootstrap/blob/master/examples/c/minimal.bpf.c)
in your editor:

```bash
$ # note --recursive to checkout libbpf submodule
$ git clone --recursive https://github.com/libbpf/libbpf-bootstrap
$ cd libbpf-bootstrap/examples/c
$ vim minimal.bpf.c
$ make minimal
$ sudo ./minimal
```

# Intro to `bpf_trace_printk()`

Linux kernel provides BPF helper, `bpf_trace_printk()`, with the following
definition:

```c
long bpf_trace_printk(const char *fmt, __u32 fmt_size, ...);
```

It's first argument, `fmt`, is a pointer to a `printf`-compatible format string
(with some kernel-specific extensions and limitations). `fmt_size` is the size
of that string, including terminating `\0`. The varargs are arguments
referenced from format string.

`bpf_trace_printk()` supports a limited subset of what you can expect from
libc's implementation of `printf()`. Basic things like `%s`, `%d`, and `%c`
work, but, say, positional arguments (`%1$s`) don't. Argument width specifiers
(`%10d`, `%-20s`, etc) works only on very recent kernels, but won't work on
earlier ones. Additionally, a bunch of kernel-specific modifiers (like `%pi6`
to print out IPv6 addresses or `%pks` for kernel strings) are supported as
well.

If a format string is invalid or is using unsupported features,
`bpf_trace_printk()` will return negative error code.

There are few more important restrictions on usage of `bpf_trace_printk()`
helper, unfortunately.

First, a BPF program using `bpf_trace_printk()` has to have a GPL-compatible
license. For [libbpf](https://github.com/libbpf/libbpf)-based BPF application
that means specifying license with a special variable:

```c
char LICENSE[] SEC("license") = "GPL";
```

For completeness, here are all the [GPL-compatible
licenses](https://github.com/torvalds/linux/blob/master/include/linux/license.h)
that kernel recognizes:
  - "GPL";
  - "GPL v2";
  - "GPL and additional rights";
  - "Dual BSD/GPL";
  - "Dual MIT/GPL";
  - "Dual MPL/GPL".

Another hard limitation is that `bpf_trace_printk()` can accept only up to 3
input arguments (in addition to `fmt` and `fmt_size`). This is quite often
pretty limiting and you might need to use multiple `bpf_trace_printk()`
invocations to log all the data. This limitation stems from the BPF helpers
ability to accept only up to 5 input arguments in total.

Once you get past these limitations, though, you'll find out that
`bpf_trace_printk()` dutifully emits data according to your format string to a
special file at `/sys/kernel/debug/tracing/trace_pipe`. You need to be root to
read it, so use `sudo cat` to watch your debug logs:

```bash
$ sudo cat  /sys/kernel/debug/tracing/trace_pipe
   <...>-2328034 [007] d... 5344927.816042: bpf_trace_printk: Hello, world, from BPF! My PID is 2328034

   <...>-2328034 [007] d... 5344928.816147: bpf_trace_printk: Hello, world, from BPF! My PID is 2328034

^C
```

Let's dissect this. `<...>-2328034 [007] d... 5344927.816042: bpf_trace_printk:
` part is emitted by the kernel automatically for each `bpf_trace_printk()`
invocation. It contains information like process name (sometimes shortened as
`<...>`), PID (`2328034`), timestamp since system boot (`5344927.816042`), etc.
But `Hello, world, from BPF! My PID is 2328034` is the part controlled by a BPF
program and emitted via a simple code like this:

```c
int pid = bpf_get_current_pid_tgid() >> 32;
const char fmt_str[] = "Hello, world, from BPF! My PID is %d\n";

bpf_trace_printk(fmt_str, sizeof(fmt_str), pid);
```

Note how `fmt_str` is defined as a variable on the stack. Unfortunately,
currently you can't just do something like `bpf_trace_printk("Hello, world!",
...);` due to libbpf limitations. But even if it was possible, the need to
specify `fmt_size` explicitly is quite inconvenient. Libbpf helpfully provides
a simple wrapper macro, `bpf_printk(fmt, ...)`, which takes care of such
details, though. It is currently defined in
[<bpf/bpf_helpers.h>](https://github.com/libbpf/libbpf/blob/master/src/bpf_helpers.h)
like this:

```c
/* Helper macro to print out debug messages */
#define bpf_printk(fmt, ...)                            \
({                                                      \
        char ____fmt[] = fmt;                           \
        bpf_trace_printk(____fmt, sizeof(____fmt),      \
                         ##__VA_ARGS__);                \
})
```

With it, the above "Hello, world!" example becomes more succinct and
convenient:

```c
int pid = bpf_get_current_pid_tgid() >> 32;

bpf_printk("Hello, world, from BPF! My PID is %d\n", pid);
```

Much nicer! Unfortunately, while convenient, this implementation is not ideal,
as it has to initialize char array on the stack with the contents of the format
string *every single time* `bpf_printk()` is called. Due to backwards
compatibility concerns, libbpf is stuck with such a suboptimal implementation
as it is the only one that will keep reliably working on old kernels and thus
won't break any BPF application, which is a high priority for libbpf as a
generic library.

I, on the other hand, am not constrained with backwards compatibility in this
blog post. So I can and will show how to improve upon this implementation
significantly in the rest of this post.

# Improving `bpf_printk()`

## Avoiding format string array on the stack

First thing we are going to address is the need to initialize an array on the
stack for format string. Starting with Linux 5.2, [d8eca5bbb2be ("bpf:
implement lookup-free direct value access for
maps")](https://github.com/torvalds/linux/commit/d8eca5bbb2be9) adds support
for BPF global (and static) variables, which we are going to use here to get
rid of on-the-stack array. The change to `bpf_printk()` implementation is
deceivingly minimal:

```c
#undef bpf_printk
#define bpf_printk(fmt, ...)                            \
({                                                      \
        static const char ____fmt[] = fmt;              \
        bpf_trace_printk(____fmt, sizeof(____fmt),      \
                         ##__VA_ARGS__);                \
})
```

`char[]` becomes `static const char[]`. `static const` modifiers ensure that
Clang puts `____fmt` variable into read-only `.rodata` ELF section and libbpf
will take care of wiring everything up during BPF application loading time.
When `bpf_printk()` needs to print something, underlying BPF code will just
need to fetch an address of `____fmt` in `.rodata` BPF map. This is fast and
efficient, compared to filling out a potentially big char array every single
time.

That’s all. In short, if your BPF application runs on Linux 5.2 (or newer) you
should always prefer this implementation over the one in libbpf.

## Newline behavior changes

Up until Linux 5.9, `bpf_trace_printk()` would take format string and use it as
is. So if you forgot (or chose not to) add `\n` to your format string, you'd
get a mess in `trace_pipe` output. `bpf_printk("Hello, world!")` executed few
times would result in:

```
   <...>-179528 [065] .... 1863682.484368: 0: Hello, world!           <...>-179528 [065] .... 1863682.484381: 0: Hello, world!           <...>-179528 [065] .... 1863683.484447: 0: Hello, world!
```

Starting with [ac5a72ea5c89 ("bpf: Use dedicated bpf_trace_printk event instead
of trace_printk()")](https://github.com/torvalds/linux/commit/ac5a72ea5c898)
(went into upstream Linux 5.9), `bpf_trace_printk()` will now always append
newline at the end, so for `bpf_printk("Hello, world!");` you'll see a tidy
output:

```
   <...>-200501 [001] .... 1863840.478848: 0: Hello, world!
   <...>-200501 [002] .... 1863841.478916: 0: Hello, world!
   <...>-200501 [002] .... 1863842.478991: 0: Hello, world!
```

Which is great, but if you were careful (as you should have) before and added
`\n` at the end of your format string, `bpf_printk("Hello, world!\n")` on
kernels before Linux 5.9 would result in a nice output like above. But starting
from Linux 5.9, you'll get an annoyingly sparse and wasteful output:

```
   <...>-3658431 [048] d... 5362570.510814: bpf_trace_printk: Hello, world!

   <...>-3658431 [048] d... 5362571.510933: bpf_trace_printk: Hello, world!

   <...>-3658431 [048] d... 5362572.511048: bpf_trace_printk: Hello, world!

```

While not the end of the world, it would be great to have consistent behavior
and not care about kernel version differences in handling that pesky `\n`,
wouldn’t it?

The good news is that with the help of BPF CO-RE we can transparently detect
and accommodate such kernel differences. If you look at the commit
[ac5a72ea5c89](https://github.com/torvalds/linux/commit/ac5a72ea5c898)
mentioned above you'll see that it adds a new kernel tracepoint
`bpf_trace_printk` and cleverly uses it to emit data to
`/sys/kernel/debug/tracing/trace_pipe`. Note also that each tracepoint in the
kernel has a corresponding `struct trace_event_raw_<tracepointname>` type. We
are going to use the existence of `struct trace_event_raw_bpf_trace_printk` to
detect whether a newline is added by `bpf_trace_printk()` or not. If not, we'll
make sure to add a newline silently and transparently in our own `bpf_printk()`
macro. Let’s see how all that is put together:

```c
[1] #include <bpf/bpf_core_read.h>

    /* define our own struct definition if our vmlinux.h is outdated */
[2] struct trace_event_raw_bpf_trace_printk___x {};

    #undef bpf_printk
    #define bpf_printk(fmt, ...)                                                    \
    ({                                                                              \
[3]         static char ____fmt[] = fmt "\0";                                       \
[4]         if (bpf_core_type_exists(struct trace_event_raw_bpf_trace_printk___x)) {\
[5]                 bpf_trace_printk(____fmt, sizeof(____fmt) - 1, ##__VA_ARGS__);  \
            } else {                                                                \
[6]                 ____fmt[sizeof(____fmt) - 2] = '\n';                            \
[7]                 bpf_trace_printk(____fmt, sizeof(____fmt), ##__VA_ARGS__);      \
            }                                                                       \
     })
```

Let's break it down a bit.

`[1]` includes libbpf's
[bpf_core_read.h](https://github.com/libbpf/libbpf/blob/master/src/bpf_core_read.h)
header which defines all the BPF CO-RE macros.

`[2]` defines our own local minimal (empty) definition of `bpf_trace_printk`
tracepoint struct to avoid dependency on having latest `vmlinux.h`. This is
important for cases when `vmlinux.h` header might be slightly out of date being
generated from kernel BTF before Linux 5.9. Adding `___x` suffix makes sure
that it won't conflict with the definition in up-to-date `vmlinux.h`. Libbpf
and BPF CO-RE will ignore `___` and everything after it, so this will still
match the actual `struct trace_event_raw_bpf_trace_printk` in the kernel. If
you are sure your `vmlinux.h` is recent enough, you can just skip this step.

`[3]` has two changes. We dropped the `const` modifier because we are going to
modify this string at runtime (on older kernels) so it has to be allocated in
the writable `.data` ELF section and corresponding BPF map. We also appended
extra `\0` at the end to reserve a space for `\n` characters, if we happen to
have a need for it. Replacing an existing character is way simpler than
appending one at runtime, so that’s what we are doing here.

`[4]` is BPF CO-RE-based detection of tracepoint presence. If a specified
struct exists in the kernel `bpf_core_type_exists()` evaluates to 1, otherwise
0 is substituted.

`[5]` is the case of Linux 5.9+, so we don't need to add a newline. The only
thing we should be careful about is to not pass two `\0`s in a format string,
as some kernels will reject this at runtime (and you won’t see any output in
the `trace_pipe` file). That's why `sizeof(____fmt) - 1` is specified as the
size of the format string, skipping the implicit '\0' added by the compiler
when allocating a string.

`[6]`-`[7]` is the case of older Linux, so we'll have to replace explicitly
reserved `\0` with `\n` to ensure that we'll get properly wrapped output. We
pass full `____fmt` size to `bpf_trace_printk()`, including implicit `\0`.

With this, `bpf_printk("Hello, world!")` will always have a newline emitted at
the end, without callers having to care about kernel version. You just need to
make sure that you always pass format strings without explicit ‘\n’.

## Detecting full-powered `bpf_trace_printk()`

In (upcoming) Linux 5.13 release `bpf_trace_printk()` implementation got a
really nice boost in capabilities thanks to [Florent
Revest](http://florentrevest.github.io/)'s work in [d9c9e4db186a ("bpf:
Factorize bpf_trace_printk and
bpf_seq_printf")](https://github.com/torvalds/linux/commit/d9c9e4db186ab).

Previously, `bpf_trace_printk()` allowed the use of only one string (`%s`)
argument, which was quite limiting. Linux 5.13 release lifts this restriction
and allows multiple string arguments, as long as total formatted output doesn't
exceed 512 bytes. Another annoying restriction was the lack of support for
width specifiers, like `%10d` or `%-20s`. This restriction is gone now as well.
Here's the list of other great improvements (from the above commit’s
description):

- bpf_trace_printk always expected fmt[fmt_size] to be the terminating NULL
  character, this is no longer true, the first 0 is terminating.
- bpf_trace_printk now supports %% (which produces the percentage char).
- bpf_trace_printk now skips width formatting fields.
- bpf_trace_printk now supports the X modifier (capital hexadecimal).
- bpf_trace_printk now supports %pK, %px, %pB, %pi4, %pI4, %pi6 and %pI6
- bpf_trace_printk now supports the %ps and %pS specifiers to print symbols.

This means that on recent enough kernels you can do quite a lot more with
`bpf_trace_printk()`. But if you want to support older kernels, you'll need to
have a fallback to a simpler logic. The question is whether it's possible to
reliably detect whether more powerful `bpf_trace_printk()` behavior can be
expected.

BPF CO-RE and libbpf can actually help with this nicely. One way would be to
check for the upstream Linux version explicitly by using the `extern int
LINUX_KERNEL_VERSION __kconfig;` variable, but that's not very reliable in the
presence of backports in Linux kernels. For such backported features Linux
kernel version doesn't correspond to the included features in the kernel. So
it’s always better to detect desired functionality support directly, if
possible.

It so happens that `bpf_trace_printk()` refactoring coincides with adding a new
BPF helper, `bpf_snprintf()`, for which those refactorings and improvements
were done in the first place. So instead of relying on kernel version checks we
are going to detect the support for `bpf_snprintf()` helper.

Each BPF helper has a corresponding `BPF_FUNC_<helpername>` enum value in [enum
bpf_func_id](https://github.com/torvalds/linux/blob/ad9f25d338605d26acedcaf3ba5fab5ca26f1c10/include/uapi/linux/bpf.h#L4911-L4916).
So by checking if a given enum value is present in vmlinux BTF, it's possible
to determine the presence of the corresponding BPF helper. Let's see how we can
do this in code:

```c
    /* don't rely on up-to-date vmlinux.h */
[1] enum bpf_func_id___x { BPF_FUNC_snprintf___x = 42 /* avoid zero */ };

[2] #define printk_is_powerful  \
            (bpf_core_enum_value_exists(enum bpf_func_id___x, BPF_FUNC_snprintf___x))

...

            const char power[] = "POWER";
            int pid = bpf_get_current_pid_tgid() >> 32;

            if (printk_is_powerful)
[4]                 bpf_printk("I've got the =%%= %7s, %s, %-7s =%%=!", power, power, power);
            else
[5]                 bpf_printk("Sorry, NO %s! :( But my PID is %d", power, pid);
```

`[1]` defines our own minimal definition of `enum bpf_func_id` and
`BPF_FUNC_snprintf` enum value within it. This is, again, to avoid depending on
having the most up-to-date `vmlinux.h`, so feel free to skip this if this
doesn't concern you. Notice the use of `___x` suffix both on enum and enum
value, in both cases `___x` suffix is going to be ignored by libbpf. The actual
value, `42`, doesn't matter as well, but it's a good idea to avoid using zero
(default value, unless explicitly specified) due to some older versions of
Clang having problems with it.

`[2]` uses `bpf_core_enum_value_exists()` to detect the presence of
`BPF_FUNC_snprintf` enum value in the running kernel. It's similar to
previously used `bpf_core_type_exists()` except applicable for enums. It will
evaluate to 1 if an enum value exists, otherwise 0 will be returned.

`[4]` handles the case of having a more feature-rich `bpf_trace_printk()`
implementation and shows off using 3 string arguments with some fancier
formatting. Also, just for fun, it makes use of `%%` escaping.

`[5]` is a fallback case using more primitive and restricted formatting.

And that's all. If you are running on Linux 5.13+, you should see:

```
   minimal-2167    [002] d..5 20804.858999: bpf_trace_printk: I've got the =%=   POWER, POWER, POWER   =%=!
   minimal-2167    [002] d..5 20805.859180: bpf_trace_printk: I've got the =%=   POWER, POWER, POWER   =%=!
```

On older kernels you'll get:

```
   <...>-3998551 [008] d... 5367146.858854: bpf_trace_printk: Sorry, NO POWER! :( But my PID is 3998551
   <...>-3998551 [008] d... 5367147.858990: bpf_trace_printk: Sorry, NO POWER! :( But my PID is 3998551
```

## Conclusion

`bpf_trace_printk()` (or rather, practically speaking, `bpf_printk()` wrapper)
is an extremely useful instrument that eases debugging BPF applications
immensely. It allows you to dump a lot of useful information from the BPF side
of your BPF application and watch it through the `trace_pipe` file. There are,
unfortunately, inconveniences associated with gradual changes in behavior and
capabilities of `bpf_trace_printk()`, but hopefully this blog post showed how
that can be abstracted away reasonably well and transparently enough by a
careful use of BPF CO-RE and other libbpf capabilities (e.g., BPF static
variables). Hopefully this information will save you some time in the future
and will let you get more out of your BPF applications.

The logical continuation of `bpf_trace_printk()` evolution is the support for
passing in more than 3 input arguments, similarly to how modern `printf()`-like
BPF helpers, `bpf_seq_printf()` and `bpf_snprintf()`, do this. This,
undoubtedly, will be added very soon, so keep an eye out on
`bpf@vger.kernel.org` [mailing
list](http://vger.kernel.org/vger-lists.html#bpf).
