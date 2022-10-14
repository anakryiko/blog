+++
date = 2022-10-14
title = "Tracing Linux kernel with retsnoop"
description = '''
Tracing Linux kernel internals easily and ergonomically with retsnoop tool.
Real-world examples and deep dives into stack trace, function call trace, and
LBR (Last Branch Record) modes and how they could be used to solve real-world
problems.
'''

[extra]
toc = true

[taxonomies]
categories = ["Retsnoop"]
tags = ["bpf", "retsnoop"]
+++

# What is retsnoop?

In short, [retsnoop](https://github.com/anakryiko/retsnoop) is a BPF-based
tool for non-intrusive mass-tracing of Linux kernel internals. `Retsnoop`'s
official summary gives a pretty good overview of why it exists in the first
place, so I'll quote it in full:

> Retsnoop's main goal is to provide a flexible and ergonomic way to extract the
> exact information from the kernel that is useful to the user. At any given
> moment, a running kernel is doing many different things, across many different
> subsystems, and on many different cores. Extracting and reviewing all of the
> various logs, callstacks, tracepoints, etc across the whole kernel can be a
> very time consuming and burdensome task. Similarly, iteratively adding printk()
> statements requires long iteration cycles of recompiling, rebooting, and
> rerunning testcases. Retsnoop, on the other hand, allows users to achieve a
> much higher signal-to-noise ratio by allowing them to specify both the specific
> subset of kernel functions that they would like to monitor, as well as the
> types of information to be collected from those functions, all without
> requiring any kernel changes.

`Retsnoop` started out as a personal experiment in trying to utilize BPF for
a powerful and customizable deep kernel tracing capabilities. Over time,
`retsnoop` grew into a proper tool with sophisticated and versatile
capabilities. Along the way, it inspired a bunch of new BPF features (BPF
cookies, on-demand LBR capture, BPF multi-kprobes) and required various bug
fixes in the Linux kernel. It is now at the point of being a well rounded tool
that I personally (and a few people that used `retsnoop` in practice) found
pretty very useful in practice when working with the Linux kernel or debugging
application issues originating in the kernel (e.g., unexplained errors from
syscalls).

While no silver bullet, it does sometimes feel magical in how easily it allows
to track down various problems and discover inner workings of the kernel.
I decided it might be a good time to try to tell people more about `retsnoop`,
improve [documentation](https://github.com/anakryiko/retsnoop#readme), and
provide a few detailed deep dives into its various features, guiding the
reader through the process, so that they can start using `retsnoop` for their
own investigations more easily.

This blog post gives a tour over main `retsnoop` features and shows how
they can be used in practice. We go over three different cases and demonstrate
`retsnoop`'s three modes of operation: **stack trace mode**, **function call
trace mode**, and **LBR mode**. This post is intended as a companion to
`retsnoop`'s [documentation](https://github.com/anakryiko/retsnoop#readme) on
its [Github repo](https://github.com/anakryiko/retsnoop). Please consult
documentation for more of a reference-style description of various `retsnoop`
features and concepts.

# Case studies

This section demonstrates various features of `retsnoop` and example output to
get a quick feel for retsnoop's capabilities. These examples are all captured
with the help of the `simfail` binary, which is part of `retsnoop`'s repo and
can be built from sources with `make simfail`. `simfail` can simulate various
error conditions in the `bpf()` syscall and is useful to get reproducible
examples and try out various `retsnoop` features in a "controlled environment".

## Failed per-CPU BPF array map case

Let's start with `simfail`'s `bpf-bad-map-val-size-percpu-array` case, which
attempts to create a per-CPU BPF array map with a value size of 1MB. This
causes the kernel to reject this because per-CPU value can't be bigger than
32KB (due to internal kernel per-CPU allocator limitations). But let's pretend
we don't know about this and we are trying to figure out why it's failing.

We do know that BPF map creation is done through the `bpf()` syscall, which on
modern x86-64 Linux kernels has has a corresponding entry point function
`__x64_sys_bpf`. On older kernels it would be `__se_sys_bpf`, and on other
architectures it could have another prefix. As we don't want to care about
such details, we'll just ask `retsnoop` to trace any function matching the
`*sys_bpf` pattern as an entry function specification. Also we know that
a lot of BPF-related code lives in `kernel/bpf/*.c` files, so we'll just
specify `-a ':kernel/bpf/*.c'` to make `retsnoop` trace a bunch of additional
functions, that, according to DWARF debug information, are defined in
`kernel/bpf/*.c` source files.

```
# run `sudo ./simfail bpf-bad-map-val-size-percpu-array` after retsnoop successfully starts

$ sudo ./retsnoop -e '*sys_bpf' -a ':kernel/bpf/*.c'
Receiving data...
20:19:36.372607 -> 20:19:36.372682 TID/PID 8346/8346 (simfail/simfail):

                    entry_SYSCALL_64_after_hwframe+0x63  (arch/x86/entry/entry_64.S:120:0)
                    do_syscall_64+0x35                   (arch/x86/entry/common.c:80:7)
                    . do_syscall_x64                     (arch/x86/entry/common.c:50:12)
    73us [-ENOMEM]  __x64_sys_bpf+0x1a                   (kernel/bpf/syscall.c:5067:1)
    70us [-ENOMEM]  __sys_bpf+0x38b                      (kernel/bpf/syscall.c:4947:9)
                    . map_create                         (kernel/bpf/syscall.c:1106:8)
                    . find_and_alloc_map                 (kernel/bpf/syscall.c:132:5)
!   50us [-ENOMEM]  array_map_alloc
!*   2us [NULL]     bpf_map_alloc_percpu


^C
Detaching... DONE in 251 ms.
```

Let's try to parse what `retsnoop` printed out. First, each completed sample
gets a single-line header:

```
20:19:36.372607 -> 20:19:36.372682 TID/PID 8346/8346 (simfail/simfail):
```

`20:19:36.372607 -> 20:19:36.372682` shows the timestamps of when
`__x64_sys_bpf` (earliest activated entry function) was called and
subsequently returned. `TID/PID 8346/8346 (simfail/simfail)` prints out thread
and process IDs, as well as thread and process short names (so-called `comm`
in kernel lingo, which you can get from `/proc/<pid>/comm`), respectively.
This information lets you identify in the context of which process and thread
the stack trace was captured.

After this we see a stack trace with some additional information. Note that
functions on the top are calling functions below them, forming a top-down call
chain.

Each line in a stack trace corresponds to one kernel function call. The kernel
function name is in the center (e.g,. `__x64_sys_bpf`, `map_create`,
`array_map_alloc`, etc), plus a hexadecimal offset (in bytes) within that
function, which allows the user to pinpoint the exact assembly instruction
that was executed at that time. Functions that have '.' (dot) in front of them
(e.g., `.  map_create` and `.  find_and_alloc_map`) are inlined functions and
they don't really have their own stack frame in the stack trace, but thanks to
available DWARF debug information `retsnoop` managed to recover this
information and present it as a `__sys_bpf` → `map_create` → 
`find_and_alloc_map` call chain, which is what is logically happening
according to C source code.

To the right of most functions we can see the source code line location that
was executing at the time of capturing the stack trace.
`kernel/bpf/syscall.c:1106:8` (next to `. map_create`) points out that
C source code line #1106 in `kernel/bpf/syscall.c` file was the one being
executed at the time of the `find_and_alloc_map` → `array_map_alloc`
→ `bpf_map_alloc_percpu` call chain executing inside `map_create`. In source
code corresponding to the kernel I've run this maps correctly to:

```c
  1105        /* find map type and init map: hashtable vs rbtree vs bloom vs ... */
  1106        map = find_and_alloc_map(attr);
  1107        if (IS_ERR(map))
  1108                return PTR_ERR(map);
```

To the left of each function that was specified through `-e` and `-a`
arguments you'll also see two more pieces of information: function latency
(e.g., 73 microseconds for `__x64_sys_bpf`) and its return result, which could
be one of `-Exxx` error codes, `NULL`, `true`/`false`, some valid address like
`0xffffc90000243000`, some other integer value, or even `void`; `retsnoop`
will do its best to represent the captured return value as meaningfully as
possible, based on the captured return value and function's type signature, if
available. In our case, we see that innermost call to `bpf_map_alloc_percpu`
returned `NULL`, which then got translated to `-ENOMEM` error and bubbled up
all the way through `__x64_sys_bpf` and eventually got returned to user space
as a result of `bpf()` syscall.

There could be a few extra markers like '!', '*', etc, at the very left side
of a line, but we won't go into detail on what they mean. Usually it's pretty
harmless to ignore them.

In general, what did we achieve and what can we understand from `retsnoop`'s
output? With a single invocation of `retsnoop` with a few arguments we managed
to trace the `bpf()` syscall all the way down to the `bpf_map_alloc_percpu`
function returning NULL. If we dig deeper into source code, we'll see that it
further calls `__alloc_percpu_gfp` (which we didn't instruct `retsnoop` to
trace, as it wasn't in either `-e` nor `-a` arguments, so we can't see that
from the above stack trace) which got to be the one further returning `NULL`,
indicating an allocation failure. We can keep digging deeper to try to
understand exactly why, by reading code and adjusting `retsnoop`'s set of
traced function, but that's beyond the scope of this example.

The main point is that we can see the sequence of function calls leading all
the way to an attempt to allocate memory for a per-CPU BPF array map, and that
it is exactly `bpf_map_alloc_percpu` that's failing. We also know that `NULL`
got translated into `-ENOMEM` before bubbling up to user space as an error
returned from the `bpf()` syscall.  Without `retsnoop` it would be
considerably harder to narrow it down so precisely without the need to modify,
recompile, and install a new kernel version.

Note also that we can see how did we get to our entry function
(`__x64_sys_bpf`) through a chain of `entry_SYSCALL_64_after_hwframe`
→ `do_syscall_64` → `__x64_sys_bpf` calls, even though we didn't instruct
`retsnoop` to trace the first two functions. This ability to see how we got to
our entry function without explicitly specifying possible functions with `-e`
or `-a` is extremely useful, especially when dealing with unfamiliar parts of
the kernel.

## Tracing BPF verification flow

`retsnoop` was initially designed to debug cases when kernel returns some
generic error (like `-EINVAL`) from somewhere deep inside a complex syscall
(like `bpf()` syscall) and it's hard to guess what that error is. So by
default `retsnoop` will only catch the call stack that ends up returning an
error (`-Exxx` or `NULL`), and will try to get the deepest call stack possible
to help narrow down the origin of the error. Just like we saw in the previous
example.

But `retsnoop` can do much more than that. It can trace any call stack, not
just an error-returning one. It also has a function call tracing mode, in
which we can see how kernel control flow evolves. In this example we'll try to
get an overview of a pretty complex workflow: successful validation of a BPF
program by the BPF verifier.

We can use `sudo ./simfail bpf-simple-obj` to load and verify a very simple
BPF program. Let's see what we can get from this.

```
# after retsnoop loads, run `sudo ./simfail bpf-simple-obj` in a separate terminal

$ sudo ./retsnoop -e 'bpf_prog_load' -a ':kernel/bpf/*.c' \
                  -d 'bpf_lsm*' -d 'push_insn' -d '__mark_reg_unknown' \
                  -S -T
Receiving data...

21:33:58.438961 -> 21:33:58.439338 TID/PID 8385/8385 (simfail/simfail):

FUNCTION CALL TRACE                               RESULT                 DURATION
-----------------------------------------------   --------------------  ---------
→ bpf_prog_load
    → bpf_prog_alloc
        ↔ bpf_prog_alloc_no_stats                 [0xffffc9000031e000]    5.539us
    ← bpf_prog_alloc                              [0xffffc9000031e000]   10.265us
    ↔ bpf_obj_name_cpy                            [11]                    2.169us
    → bpf_check
        ↔ check_subprogs                          [0]                     1.903us
        ↔ btf_get_by_fd                           [0xffff888122ae9100]    1.851us
        ↔ bpf_check_uarg_tail_zero                [0]                     1.740us
        [...]
        ↔ check_cfg                               [0]                     2.199us
        → do_check_common
            [...]
            ↔ free_verifier_state                 [void]                  2.011us
        ← do_check_common                         [0]                   215.469us
        ↔ check_max_stack_depth                   [0]                     1.844us
        ↔ convert_ctx_accesses                    [0]                     2.156us
        ↔ do_misc_fixups                          [0]                     1.848us
        ↔ verbose                                 [void]                  1.775us
    ← bpf_check                                   [0]                   293.563us
    → bpf_prog_select_runtime
        ↔ bpf_prog_alloc_jited_linfo              [0]                     2.943us
        → bpf_int_jit_compile
            [...]
        ← bpf_int_jit_compile                     [0xffffc9000031e000]   30.520us
        ↔ bpf_prog_jit_attempt_done               [void]                  1.778us
    ← bpf_prog_select_runtime                     [0xffffc9000031e000]   42.725us
    → bpf_prog_kallsyms_add
        ↔ bpf_ksym_add                            [void]                  2.046us
    ← bpf_prog_kallsyms_add                       [void]                  6.104us
← bpf_prog_load                                   [5]                   374.697us

                 entry_SYSCALL_64_after_hwframe+0x63  (arch/x86/entry/entry_64.S:120:0)
                 do_syscall_64+0x35                   (arch/x86/entry/common.c:80:7)
                 . do_syscall_x64                     (arch/x86/entry/common.c:50:12)
                 __x64_sys_bpf+0x1a                   (kernel/bpf/syscall.c:5067:1)
                 __sys_bpf+0x11a4                     (kernel/bpf/syscall.c:4965:9)
!  374us [5]     bpf_prog_load
!    6us [void]  bpf_prog_kallsyms_add
!    2us [void]  bpf_ksym_add
!*  29us [0]     check_mem_access
!*   1us [NULL]  bpf_map_kptr_off_contains
```

Ok, that's quite a lot of output. And note that I've already truncated it
somewhat and inserted `[...]` markers to keep it shorter. Let's take it in step
by step.

Let's first see how we invoke `retsnoop`:

```
$ sudo ./retsnoop -e 'bpf_prog_load' -a ':kernel/bpf/*.c' \
                  -d 'bpf_lsm*' -d 'push_insn' -d '__mark_reg_unknown' \
                  -S -T
```

Instead of using the `bpf()` syscall as an entry point, we ask `retsnoop` to
only activate tracing when `bpf_prog_load` is called. This is an entry point
for the BPF program validation command inside the `bpf()` syscall. This allows
us to ignore other irrelevant `bpf()` syscall invocations.

Next, we ask `retsnoop` to trace all discovered (through DWARF debug info)
functions in `kernel/bpf/*.c` files. As we don't really know what we are
looking for, it makes sense to "cast the net wide". Note also that I filtered
out (with the use of `-d` arguments) a few naming patterns ('bpf_lsm\*',
'push_insn', and '__mark_reg_unknown') as they are called pretty frequently
and pollute output without adding too much to it. In practice using `retsnoop`
is an iterative process, where you gradually add more filters like this to
capture most relevant data and avoid distracting and non-essential noise.

Last, but not least, two more arguments: `-S` (`--success-stacks`) and `-T`
(`--trace`).

`-S` instructs `retsnoop` to not ignore "successful cases", i.e., those that
don't end up returning errors. In our case we wanted to investigate
successfully verified BPF program, so not specifying `-S` would completely
ignore such case. Adding `-S` disables `retsnoop`'s default error-focused
capturing logic.

`-T` instructs `retsnoop` to collect a detailed record of all traced function
calls and emit it in a separate table, like the one you can see above.

We'll look at what that function call trace table shows in a second. But
first, we do also see a standard stack trace at the very bottom of the output.
In this case, it ends up with `bpf_map_kptr_off_contains` that returns `NULL`.
This is `retsnoop` trying to be helpful to detect the deepest error-returning
function call chain (and `NULL` is considered an error by default). We don't
care much about the call stack in this example, so just ignore it in this
case. Stack trace is always emitted by `retsnoop`, there is no way to suppress
it. This might be possible in the future.

Back to function call trace table. Let's shorten it some more to explain the
main elements:

```
FUNCTION CALL TRACE                               RESULT                 DURATION
-----------------------------------------------   --------------------  ---------
→ bpf_prog_load
    → bpf_prog_alloc
        ↔ bpf_prog_alloc_no_stats                 [0xffffc9000031e000]    5.539us
    ← bpf_prog_alloc                              [0xffffc9000031e000]   10.265us
    [...]
    → bpf_prog_kallsyms_add
        ↔ bpf_ksym_add                            [void]                  2.046us
    ← bpf_prog_kallsyms_add                       [void]                  6.104us
← bpf_prog_load                                   [5]                   374.697us
```

In the left column we see function names with one of `→`, `←`, or `↔` markers
on the left. `→` marks entry into the traced function, `←` marks an exit from a
function. If a function didn't have any further calls to any other traced
function and thus is a leaf function (as far as a specified set of functions of
interest goes; note, that in reality there could have been other functions
called further, but they were not traced and thus captured by `retsnoop` due to
them being omitted from `-e`/`-d` specifications). Such leaf functions, instead
of taking two rows with separate `→` (entry) and `←` (exit) records, are
collapsed into a single entry-exit row marked with `↔`. As calls become more
nested, `retsnoop` indents each level of nestedness with four spaces.

For each row corresponding to a function exit event (so, `→`- and `↔`-marked
rows), retsnoop also reports the return result, similarly to the default stack
trace view. And also similarly to the stack trace view, we get a duration of
a complete function call (inclusive of all the child functions' execution time
as well).

So from the short excerpt above, we can see that `bpf_prog_load` was called,
it then called `bpf_prog_alloc` which further called
`bpf_prog_alloc_no_stats`.  `bpf_prog_alloc_no_stats` returned some valid
kernel address (`0xffffc9000031e000`), which was subsequently returned from
`bpf_prog_alloc` back to `bpf_prog_load`. We then skip a lot of function
invocations for brevity, but eventually we get to `bpf_prog_kallsyms_add`,
which calls `bpf_ksym_add` once. Then `bpf_ksym_add` and, subsequently,
`bpf_prog_kallsyms_add` return with no results (they are `void` functions,
according to BTF type information). After `bpf_prog_kallsyms_add` returned,
`bpf_prog_load` completes and returns a successful result (FD 5 in this case).
The entire invocation of `bpf_prog_load` takes almost 375 microseconds.

So, as you can see from the short analysis above, `-T` (`--trace`) mode allows
you to investigate internal control flow of kernel logic as can be seen
through function calls. With `retsnoop`'s flexible mechanism to specify just
the right subset of functions to be traced, such function traces can omit
a lot of uninteresting function calls, if you know what you are most
interested in.

This tracing mode is really great to discover how kernel works and I hope
you'll find it valuable!

## Peering deep into functions with LBR

To demonstrate LBR mode and how it helps in practice we'll utilize the
`bpf-bad-map-max-entries-array` test case from `simfail`. In this test case
`simfail` is attempting to create a BPF array map with invalid parameters,
leading to the `bpf()` syscall returning a generic `-EINVAL` error. Let's see
how we can quickly narrow this down to a specific check without knowing much
about the BPF subsystem source code.

We'll start simple. We know the `bpf()` syscall is failing, so our entry glob
will be `*sys_bpf`. And we know that we want to use LBR mode, so let's also
specify `--lbr`. Here's what we get (note that I cut out a bunch of irrelevant
LBR output rows for brevity, as they are not relevant to this example).

```
# after retsnoop starts, run `sudo ./simfail bpf-bad-map-max-entries-array` in another terminal

$ sudo ./retsnoop -e '*sys_bpf' --lbr
Receiving data...
16:48:57.961192 -> 16:48:57.961216 TID/PID 1056498/1056498 (simfail/simfail):

[... cut LBRs #08 through #31 ...]

[#07] _copy_from_user+0x2f       (lib/usercopy.c:21:1)          ->  __sys_bpf+0xcc             (kernel/bpf/syscall.c:4620:5)
[#06] security_bpf+0x3e          (security/security.c:2525:1)   ->  __sys_bpf+0xe5             (kernel/bpf/syscall.c:4624:5)
[#05] memchr_inv+0x78            (lib/string.c:1126:1)          ->  __sys_bpf+0xbcf            (kernel/bpf/syscall.c:4629:9)
                                                                    . map_create               (kernel/bpf/syscall.c:838:5)
[#04] array_map_alloc_check+0x74 (kernel/bpf/arraymap.c:79:1)   ->  __sys_bpf+0xca0            (kernel/bpf/syscall.c:4629:9)
                                                                    . map_create               (kernel/bpf/syscall.c:863:8)
                                                                    . find_and_alloc_map       (kernel/bpf/syscall.c:123:6)
[#03] __sys_bpf+0x93             (kernel/bpf/syscall.c:4747:1)  ->  __kretprobe_trampoline+0x0

                    entry_SYSCALL_64_after_hwframe+0x44  (arch/x86/entry/entry_64.S:112:0)
                    do_syscall_64+0x2d                   (arch/x86/entry/common.c:46:12)
    24us [-EINVAL]  __x64_sys_bpf+0x1c                   (kernel/bpf/syscall.c:4749:1)
!   10us [-EINVAL]  __sys_bpf
```

Looking at stack trace output we do indeed see that `__x64_sys_bpf` returned
`-EINVAL`, so this must be what we are trying to catch. But beyond that it
doesn't really help as we haven't captured any extra non-entry functions.

But let's look at LBR output. It's quite different from stack trace and
function call trace outputs. LBR fundamentally is a collection of records,
where each record is a pair of memory addresses: instruction address from
which CPU performed a call, return, or jump and correspondingly instruction
address to where such call/return/jump was performed. When post-processed to
turn such addresses into matched function and source code location (as we do
for stack traces as well), we'll get a collection of pairs of from/to
functions (and, if DWARF data is available, their source code locations, down
to exact line within a specific source code file).

This is what we basically see in the output above. To the left of `->` we see
where the jump was performed from, and to the right is a destination of such
jump. Each individual LBR record is also numbered and in the output above we
see `[#03]` through `[#07]`. In the full output I get records numbered all the
way to `[#31]`, because my CPU supports capturing 32 LBR records. The reason
we don't see `[#00]` through `[#02]` is due to `retsnoop` filtering out the
few irrelevant LBR records that are "wasted" on kernel-internal functionality
necessary to capture LBR at all. This is an unavoidable price to be able to
capture LBR at any point with BPF: we have to waste a few records (and
sometimes quite a few more than 3, depending on LBR configuration. E.g., with
`--lbr=any`, a whopping 18 records are wasted. Hopefully in the future this
can be improved in the kernel itself).

Each record can somewhat confusingly have multiple functions associated with
it, like record #4 above:

```
[#04] array_map_alloc_check+0x74 (kernel/bpf/arraymap.c:79:1)  ->  __sys_bpf+0xca0      (kernel/bpf/syscall.c:4629:9)
                                                                   . map_create         (kernel/bpf/syscall.c:863:8)
                                                                   . find_and_alloc_map (kernel/bpf/syscall.c:123:6)
```

This is actually the case of having inlined function calls. Both `map_create`
is inlined inside `__sys_bpf`, and `find_and_alloc_map` is inlined within
`map_create`, so the LBR record captured a jump to what used to be code
belonging to `find_and_alloc_map` and `retsnoop` helpfully points out that
it's logically a sequence of `__sys_bpf` → `map_create` → `find_and_alloc_map`
calls.

Apart from that, each side of an `->` arrow is a simplified version of default
stack trace output format, except there is no return result or duration there.
Just a function name (with a hexadecimal offset) and a corresponding source code
location.

Ok, back to the task at hand. It's best to "unwind" LBR records from the most
recent (bottom ones, with smalles numbers) to older (top ones, with higher
numbers).

Record #3 isn't all that useful, it just shows that we exited `__sys_bpf` and
jumped into the kernel's kretprobe handler (which is what `retsnoop` is using
to collect all this data). But moving up to record #4 is much more
interesting.  The jump-from location at `kernel/bpf/arraymap.c:79` is pointing
to a return from `array_map_alloc_check` and `kernel/bpf/syscall.c:123` (the
jump-to location) points inside `find_and_alloc_map`:

```c
   121        if (ops->map_alloc_check) {
   122                err = ops->map_alloc_check(attr);
   123                if (err)
   124                        return ERR_PTR(err);
   125        }
```

So this is a generic callback (`ops->map_alloc_check()`) and without being
very familiar with how things are set up in BPF subsystem, it would be very
hard to figure out what the actual function is that's called here. But
luckily, with LBR data we know that it's `array_map_alloc_check` defined in
`kernel/bpf/arraymap.c`. This gets us closer to narrowing this down, but we
are not quite there yet.

Unfortunately, moving to record #5 doesn't give any new clue, as we apparently
already skipped the relevant parts, it's some generic preliminary check that
didn't fail. So we have to stop here and consider what to do next. It's
actually a very common pattern to need to use `retsnoop` iteratively and
adjust as you learn new bits of information. This also means that `retsnoop`
is most useful if you have a quickly reproducible case, otherwise each
iteration is extremely painful and slow.

Anyways, we know that `array_map_alloc_check` is most probably where the error
is returned, so let's zero in on it. We'll add it as a non-entry function and
switch to capturing any jump (not just function returns anymore) with LBR.
Note `--lbr=any` below, that's how we can tune LBR configuration to be used.

```
# after retsnoop starts, run `sudo ./simfail bpf-bad-map-max-entries-array` in another terminal

$ sudo ./retsnoop -e '*sys_bpf' -a 'array_map_alloc_check' --lbr=any
Receiving data...
20:29:17.844718 -> 20:29:17.844749 TID/PID 2385333/2385333 (simfail/simfail):

[#31] trace_call_bpf+0xf6         (kernel/trace/bpf_trace.c:133:1)          ->  kprobe_perf_func+0x59       (kernel/trace/trace_kprobe.c:1596:6)
[#30] kprobe_perf_func+0x67       (kernel/trace/trace_kprobe.c:1596:6)      ->  kprobe_perf_func+0x8c       (kernel/trace/trace_kprobe.c:1598:6)
[#29] kprobe_perf_func+0x8e       (kernel/trace/trace_kprobe.c:1598:6)      ->  kprobe_perf_func+0x1f9      (kernel/trace/trace_kprobe.c:1620:9)
[#28] kprobe_perf_func+0x1fb      (kernel/trace/trace_kprobe.c:1620:9)      ->  kprobe_perf_func+0x69       (kernel/trace/trace_kprobe.c:1621:1)
[#27] kprobe_perf_func+0x8b       (kernel/trace/trace_kprobe.c:1621:1)      ->  aggr_pre_handler+0x3a       (kernel/kprobes.c:1171:7)
[#26] aggr_pre_handler+0x5d       (kernel/kprobes.c:1177:1)                 ->  kprobe_ftrace_handler+0x10c (arch/x86/kernel/kprobes/ftrace.c:43:23)
[#25] kprobe_ftrace_handler+0x10e (arch/x86/kernel/kprobes/ftrace.c:43:23)  ->  kprobe_ftrace_handler+0x156 (arch/x86/kernel/kprobes/ftrace.c:48:38)
[#24] kprobe_ftrace_handler+0x175 (arch/x86/kernel/kprobes/ftrace.c:53:13)  ->  kprobe_ftrace_handler+0x110 (arch/x86/kernel/kprobes/ftrace.c:59:3)
[#23] kprobe_ftrace_handler+0x146 (arch/x86/kernel/kprobes/ftrace.c:64:1)   ->  ftrace_trampoline+0xc8
[#22] ftrace_trampoline+0x14c                                               ->  array_map_alloc_check+0x5   (kernel/bpf/arraymap.c:53:20)
[#21] array_map_alloc_check+0x13  (kernel/bpf/arraymap.c:54:18)             ->  array_map_alloc_check+0x75  (kernel/bpf/arraymap.c:54:18)
      . bpf_map_attr_numa_node    (include/linux/bpf.h:1735:19)                 . bpf_map_attr_numa_node    (include/linux/bpf.h:1735:19)
[#20] array_map_alloc_check+0x7a  (kernel/bpf/arraymap.c:54:18)             ->  array_map_alloc_check+0x18  (kernel/bpf/arraymap.c:57:5)
      . bpf_map_attr_numa_node    (include/linux/bpf.h:1735:19)
[#19] array_map_alloc_check+0x1d  (kernel/bpf/arraymap.c:57:5)              ->  array_map_alloc_check+0x6f  (kernel/bpf/arraymap.c:62:10)
[#18] array_map_alloc_check+0x74  (kernel/bpf/arraymap.c:79:1)              ->  __kretprobe_trampoline+0x0

                    entry_SYSCALL_64_after_hwframe+0x44  (arch/x86/entry/entry_64.S:112:0)
                    do_syscall_64+0x2d                   (arch/x86/entry/common.c:46:12)
    30us [-EINVAL]  __x64_sys_bpf+0x1c                   (kernel/bpf/syscall.c:4749:1)
    23us [-EINVAL]  __sys_bpf+0xca0                      (kernel/bpf/syscall.c:4629:9)
                    . map_create                         (kernel/bpf/syscall.c:863:8)
                    . find_and_alloc_map                 (kernel/bpf/syscall.c:123:6)
!    8us [-EINVAL]  array_map_alloc_check
```

We'll completely ignore `bpf_map_attr_numa_node` as it's a straightforward
helper function that can't return an error. But let's carefully check what's
going on with all these jumps inside `array_map_alloc_check`, records #18-#21
are of the most interest here. Most of the action seems to happen between
lines 53-62 in `kernel/bpf/arraymap.c`, and line 79 is just an exit from the
function (literally closing '}' on its own line). Here's the relevant source
code for the reference:

```c
    51 int array_map_alloc_check(union bpf_attr *attr)
    52 {
    53         bool percpu = attr->map_type == BPF_MAP_TYPE_PERCPU_ARRAY;
    54         int numa_node = bpf_map_attr_numa_node(attr);
    55
    56         /* check sanity of attributes */
    57         if (attr->max_entries == 0 || attr->key_size != 4 ||
    58             attr->value_size == 0 ||
    59             attr->map_flags & ~ARRAY_CREATE_FLAG_MASK ||
    60             !bpf_map_flags_access_ok(attr->map_flags) ||
    61             (percpu && numa_node != NUMA_NO_NODE))
    62                 return -EINVAL;
```

Ok, so looking at the LBR record carefully, we see that we jumped from line 54
(`numa_node` initialization) to line 57 (`max_entries` and `key_size` checks
in that giant if condition), then immediately to line 62 (`return -EINVAL;`,
it checks out) before exiting the function (line 79). Note that we didn't go
to lines 58-61, so either `max_entries` is zero, or `key_size` isn't 4. At
this point you'd go to your application and double check which of those
conditions are not satisfied, and in my case `simfail` does leave
`max_entries` at zero, which is exactly the reason for that `-EINVAL`.

Victory! As you can see, in just two steps (and if we started with `--lbr=any`
even without knowing about `array_map_alloc_check` we'd be able to achieve the
same in one step, in this simple case)  we narrowed down the problem
precisely.  Of course in practice it could take more work and iterations to
pinpoint the problem, but nevertheless LBR proved extremely useful in practice
so far.  Hopefully this example will help you utilize the power of `retsnoop`
to solve your practical issues.

# Conclusion

This concludes a tour of `retsnoop`'s three different modes of operation.
Hopefully this helped to understand what `retsnoop` is, what
capabilities it provides, and how to make the best use of it in practice.

Please also check out `retsnoop`'s [Github repo](https://github.com/anakryiko/retsnoop)
for more information, source code, and instruction on building `retsnoop` or
downloading pre-built versions.
